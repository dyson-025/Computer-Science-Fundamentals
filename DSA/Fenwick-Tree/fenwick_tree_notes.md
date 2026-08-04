# Fenwick Tree (Binary Indexed Tree) — Complete Notes

## 1. What It Solves

A Fenwick Tree (BIT) maintains a **prefix aggregate** (usually sum) over an array while supporting:
- **Point update**: change a single element
- **Prefix query**: get sum of first `i` elements
- **Range query**: get sum of elements in `[l, r]` (via two prefix queries)

Both operations run in **O(log n)**, versus O(n) for a naive array (update fast, query slow) or a full segment tree (more code, more memory, same asymptotic complexity but bigger constant).

**Use a Fenwick Tree when:**
- You need repeated prefix-sum (or similar) queries interleaved with point updates.
- The operation is invertible (sum, XOR) — not min/max (segment tree is better there, though BIT can be adapted with tricks).
- You want less code than a segment tree.

---

## 2. Core Idea (Intuition)

Every integer can be decomposed into a sum of powers of 2 using its binary representation. The Fenwick Tree exploits this: each index `i` in the BIT array stores the sum of a range of elements, and the **size of that range equals the lowest set bit of `i`** (in 1-indexed form).

Key operation: **`i & (-i)`** — isolates the lowest set bit of `i`.

Example: `i = 12` → binary `1100` → `-i` in two's complement → `i & (-i) = 4` (binary `0100`).

This single trick drives everything:
- **To update index `i`**: add the lowest set bit to move to the next responsible node → `i += i & (-i)`
- **To query prefix sum up to `i`**: subtract the lowest set bit to move to the previous range → `i -= i & (-i)`

**Tree is 1-indexed.** This is critical — index 0 breaks the bit-manipulation trick (0 & -0 = 0, infinite loop).

---

## 3. Standard Template (Point Update, Prefix Sum Query)

### C++
```cpp
struct Fenwick {
    int n;
    vector<long long> tree;

    Fenwick(int n) : n(n), tree(n + 1, 0) {}

    // add val to index i (1-indexed)
    void update(int i, long long val) {
        for (; i <= n; i += i & (-i))
            tree[i] += val;
    }

    // sum of [1, i]
    long long query(int i) {
        long long sum = 0;
        for (; i > 0; i -= i & (-i))
            sum += tree[i];
        return sum;
    }

    // sum of [l, r], 1-indexed inclusive
    long long rangeQuery(int l, int r) {
        if (l > r) return 0;
        return query(r) - query(l - 1);
    }
};
```

### Python
```python
class Fenwick:
    def __init__(self, n):
        self.n = n
        self.tree = [0] * (n + 1)

    def update(self, i, val):
        while i <= self.n:
            self.tree[i] += val
            i += i & (-i)

    def query(self, i):
        s = 0
        while i > 0:
            s += self.tree[i]
            i -= i & (-i)
        return s

    def range_query(self, l, r):
        if l > r:
            return 0
        return self.query(r) - self.query(l - 1)
```

**Note on `update`**: this adds a **delta**, not sets a value. To *set* index `i` to value `v`, first compute `delta = v - current_value(i)` (need a separate array to track current values), then `update(i, delta)`.

---

## 4. Building from an Initial Array

**Naive:** call `update(i, a[i])` for each `i` → O(n log n).

**Fast O(n) build:**
```cpp
vector<long long> tree(n + 1, 0);
for (int i = 1; i <= n; i++) {
    tree[i] += a[i]; // a is 1-indexed
    int parent = i + (i & (-i));
    if (parent <= n) tree[parent] += tree[i];
}
```
This works because every node's contribution propagates upward exactly once.

---

## 5. Variants (Very Important for Problem-Solving)

### A. Range Update, Point Query
Use a **difference array** trick with a BIT.
- To add `val` to range `[l, r]`: `update(l, val)`, `update(r+1, -val)`
- To query value at index `i`: `query(i)` (prefix sum of the difference array = actual value at `i`)

```cpp
void rangeUpdate(int l, int r, long long val) {
    update(l, val);
    update(r + 1, -val);
}
long long pointQuery(int i) {
    return query(i); // prefix sum up to i
}
```

### B. Range Update, Range Query
Needs **two BITs**. Maintain difference array `d[i] = a[i] - a[i-1]`.

Prefix sum formula: `sum(1..i) = i * prefix(d, i) - prefix(i * d, i)`

```cpp
struct RangeBIT {
    Fenwick B1, B2; // B1 tracks d[i], B2 tracks i*d[i]
    RangeBIT(int n) : B1(n), B2(n) {}

    void updateRange(int l, int r, long long val) {
        B1.update(l, val);
        B1.update(r + 1, -val);
        B2.update(l, val * (l - 1));
        B2.update(r + 1, -val * r);
    }

    long long prefixSum(int i) {
        return i * B1.query(i) - B2.query(i);
    }

    long long rangeSum(int l, int r) {
        return prefixSum(r) - prefixSum(l - 1);
    }
};
```

### C. 2D Fenwick Tree (Matrix prefix sums, point update)
```cpp
struct Fenwick2D {
    int n, m;
    vector<vector<long long>> tree;
    Fenwick2D(int n, int m) : n(n), m(m), tree(n + 1, vector<long long>(m + 1, 0)) {}

    void update(int x, int y, long long val) {
        for (int i = x; i <= n; i += i & (-i))
            for (int j = y; j <= m; j += j & (-j))
                tree[i][j] += val;
    }

    long long query(int x, int y) {
        long long sum = 0;
        for (int i = x; i > 0; i -= i & (-i))
            for (int j = y; j > 0; j -= j & (-j))
                sum += tree[i][j];
        return sum;
    }

    long long rangeQuery(int x1, int y1, int x2, int y2) {
        return query(x2, y2) - query(x1 - 1, y2) - query(x2, y1 - 1) + query(x1 - 1, y1 - 1);
    }
};
```
Complexity: O(log n log m) per operation.

### D. Fenwick Tree for XOR / Other Invertible Ops
Same structure works for XOR since XOR is invertible (self-inverse). Replace `+=`/`-=` with `^=`. Does **not** work directly for min/max because you can't "subtract" a min — needs different handling (e.g., a segment tree, or a BIT restricted to prefix-min queries only, no arbitrary point decrease-undo).

### E. Order Statistics / "Find k-th Element" (BIT + Binary Search)
Used for problems like "count of smaller elements after self," or finding the k-th smallest remaining element (like in a Fenwick-based order-statistics tree).

**Binary lifting on the Fenwick tree** to find smallest `i` such that `prefix(i) >= k` in O(log n):
```cpp
int findKth(int k) {
    int pos = 0;
    int logn = log2(n);
    for (int pw = 1 << logn; pw > 0; pw >>= 1) {
        if (pos + pw <= n && tree[pos + pw] < k) {
            pos += pw;
            k -= tree[pos];
        }
    }
    return pos + 1; // 1-indexed position of k-th element
}
```
This is the backbone of solving "count inversions," "k-th smallest with updates," and similar problems in O(log n) per query instead of O(log^2 n) with binary search + query.

---

## 6. Classic Problem Patterns

| Pattern | Technique |
|---|---|
| Range sum queries + point updates | Standard BIT |
| Range sum queries + range updates | Two BITs (difference trick) |
| Count inversions in array | BIT over value range: iterate array, query count of elements greater/less than current already inserted, then insert |
| Count of smaller elements to the right | Same as inversions, process right to left, BIT indexed by value (compress values if large) |
| K-th order statistic / find k-th smallest | BIT + binary lifting (see 5E) |
| 2D range sum (matrix) | 2D BIT |
| Frequency table + rank queries | BIT indexed by value, `update(v, 1)` on insert, `query(v)` = count ≤ v |
| Longest Increasing Subsequence variants (count-based DP) | BIT storing max/count indexed by compressed value |
| XOR prefix queries | BIT with XOR instead of sum |

**Coordinate compression is common**: when values are large or arbitrary (not small dense integers), sort unique values, map each to its rank, then use rank as the BIT index.

---

## 7. Complexity Summary

| Operation | Time | Space |
|---|---|---|
| Build (fast) | O(n) | O(n) |
| Build (naive, n updates) | O(n log n) | O(n) |
| Point update | O(log n) | — |
| Prefix query | O(log n) | — |
| Range query | O(log n) | — |
| Range update + range query | O(log n) | O(n) (2 BITs) |
| 2D update/query | O(log n log m) | O(n·m) |
| Find k-th element | O(log n) | — |

---

## 8. Common Pitfalls / Debugging Checklist

1. **1-indexing.** Always convert 0-indexed input arrays to 1-indexed before inserting into a BIT. Index 0 is invalid for update/query loops.
2. **`update` adds a delta, not sets a value.** If you need "set index i to x," track the old value separately.
3. **Off-by-one in range queries.** `rangeQuery(l, r) = query(r) - query(l-1)`, not `query(r) - query(l)`.
4. **Negative numbers / overflow.** Use `long long` (C++) if sums can exceed 32-bit range.
5. **`i & (-i)` requires signed integers** with two's complement behavior — works natively in C++/Java/Python ints, just be aware if porting to unusual environments.
6. **Difference array reset for range update+point query:** don't forget the `update(r+1, -val)` step, or the added value "leaks" past index r.
7. **When values are huge/sparse**, compress coordinates first — a BIT array of size 10^9 will blow memory.
8. **BIT can't do range min/max updates naturally** — sum/XOR/count only (invertible operations). For min/max with updates, use a segment tree instead.

---

## 9. Fenwick Tree vs Segment Tree — Quick Decision Guide

| | Fenwick Tree | Segment Tree |
|---|---|---|
| Code complexity | Very short (~15 lines) | Longer |
| Supports | Sum, XOR, invertible ops | Anything (min, max, gcd, custom merge) |
| Range update + range query | Possible but needs 2 BITs | Native with lazy propagation |
| Memory | O(n) | O(4n) typical |
| Speed constant | Faster in practice | Slightly slower |

**Rule of thumb:** if the problem only needs prefix sums / counts with point updates, reach for a Fenwick Tree first — it's simpler and faster to code under time pressure (e.g., competitive programming).

---

## 10. Worked Mini-Example

Array (1-indexed): `a = [_, 3, 2, -1, 6, 5, 4, -3, 3, 7, 2]` (index 0 unused)

- `update(3, 5)` → adds 5 to `a[3]`, propagates to BIT indices `3 → 4 → 8` (since `3+1=4`, `4+4=8`).
- `query(5)` → sums `a[1..5]`, by visiting BIT indices `5 → 4 → 0` (stop), combining `tree[5] + tree[4]`.

Trace `i & (-i)` for update starting at i=3:
- i=3 → 3 & -3 = 1 → i becomes 4
- i=4 → 4 & -4 = 4 → i becomes 8
- i=8 → continues until i > n

Trace for query starting at i=5:
- i=5 → 5 & -5 = 1 → i becomes 4
- i=4 → 4 & -4 = 4 → i becomes 0 → stop

---

## 11. LeetCode-Focused Additions (for interviews / rating 2200+)

CF-specific tools (Euler-tour BIT-on-trees, persistent BIT, CDQ divide-and-conquer) almost never come up on LeetCode. What actually shows up repeatedly instead:

### A. BIT Storing Max (not sum) — for DP-on-value-range problems
Same structure, but `update` takes the max instead of adding, and there's **no subtraction** in query (max isn't invertible, so no `l-1` trick — you only ever query prefix max, never a plain range unless values are compressed and monotonic).

```python
class MaxBIT:
    def __init__(self, n):
        self.n = n
        self.tree = [0] * (n + 1)

    def update(self, i, val):
        while i <= self.n:
            self.tree[i] = max(self.tree[i], val)
            i += i & (-i)

    def query(self, i):  # max over [1, i]
        res = 0
        while i > 0:
            res = max(res, self.tree[i])
            i -= i & (-i)
        return res
```
Used for: "longest chain/subsequence ending with value ≤ v" type DP — you compress values, then for each element query `max(dp[1..v-1])` and update `dp[v]`. This turns an O(n²) LIS-style DP into O(n log n).

### B. Coordinate Compression Template (the actual bottleneck skill)
Almost every hard LC Fenwick problem needs this first, and getting it wrong (or not realizing you need it) is the #1 reason people get stuck.

```python
def compress(vals):
    sorted_unique = sorted(set(vals))
    rank = {v: i + 1 for i, v in enumerate(sorted_unique)}  # 1-indexed for BIT
    return rank

# usage:
rank = compress(nums)
bit = Fenwick(len(rank))
for x in nums:
    bit.update(rank[x], 1)
```
Watch for: duplicate values, negative numbers, and whether you need rank of value or rank of value±1 (e.g. "count elements strictly less than x" vs "≤ x" — off-by-one here is the most common bug).

### C. "BIT indexed by value" — the single most common LC pattern
This is the pattern behind the majority of hard BIT problems on LC. General template:

1. Compress all relevant values (the array itself, or all values ± some offset if the problem involves sums/differences, e.g. reverse pairs, range sum count).
2. Sweep the array left-to-right (or right-to-left, depending on what "after self" / "before self" means).
3. At each step, **query first** (count of already-inserted elements satisfying the condition), **then insert** current element into the BIT.
4. Order of query vs insert, and direction of sweep, is where most bugs happen — always ask "does this element see elements inserted before or after it?"

### D. Mapped LC Problem List (patterns → problems)
| Pattern | LC Problems |
|---|---|
| Count smaller/larger after self | 315. Count of Smaller Numbers After Self |
| Reverse pairs (i<j, nums[i] > 2*nums[j]) | 493. Reverse Pairs |
| Range sum count (prefix sum + compression) | 327. Count of Range Sum |
| Insert with rank tracking | 1649. Create Sorted Array through Instructions |
| BIT storing max for DP | 1626. Best Team With No Conflicts (sort+BIT-max variant), 2407. Longest Increasing Subsequence II |
| Range update + range query | 2949-style "range add" problems, 3165. Maximum Sum of Subsequence With Non-adjacent Elements (segment tree more typical, but BIT variants exist) |
| 2D BIT / matrix range sum with updates | 2536. Increment Submatrix by One, 308. Range Sum Query 2D - Mutable |
| Sliding window + BIT (count in range) | 2179. Count Good Triplets in an Array |
| Offline queries sorted + BIT sweep | 1505. Minimum Possible Integer After at Most K Adjacent Swaps |

If you can independently derive the solution approach for **315, 327, 493, and 2407** using a BIT, you have genuinely internalized the pattern — those four cover almost every trick this data structure needs on LeetCode.

### E. Interview Framing Tips
- If asked "can you do better than O(n²) / O(n log² n)", BIT is often the answer when the problem is: prefix aggregate + point update + you can compress the value domain.
- Say out loud: "I'll use a Fenwick tree here since we need prefix sums with point updates in O(log n), and the value range can be compressed to fit an array-indexed structure." This signals you know *why*, not just *how*.
- Mention the segment-tree alternative and why you're not using it (BIT is simpler/faster when you don't need min/max or lazy propagation) — this is a strong signal in interviews.

---

## 12. Quick-Reference Cheat Sheet

```
1-indexed always.
lowbit(i) = i & (-i)

update(i, val):   while i <= n: tree[i] += val; i += lowbit(i)
query(i):         s = 0; while i > 0: s += tree[i]; i -= lowbit(i); return s
rangeSum(l, r):   query(r) - query(l - 1)

Range update + point query: difference array BIT
Range update + range query: two BITs, formula i*B1.query(i) - B2.query(i)
2D: nested loops on both dimensions with lowbit
k-th element: binary lifting from highest power of 2 down to 1
```
