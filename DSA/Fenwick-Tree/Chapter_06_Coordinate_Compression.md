# Chapter 6 — Coordinate Compression
*"When your values are too large to become Fenwick Tree indices."*

---

# 6.1 A New Problem

Everything we have learned so far assumes something.

Suppose our array is

```text
3 5 1 8 2
```

We build a Fenwick Tree of size

```text
5
```

Everything works.

Now suppose the problem changes.

```text
1000000000

999999998

500000000

7
```

Can we still build a Fenwick Tree?

---

# 6.2 The First Thought

Fenwick Tree uses an array.

Usually

```cpp
vector<int> bit(n+1);
```

The index of the Fenwick Tree must exist.

Suppose

```text
Value

=

1,000,000,000
```

Then we would need

```cpp
bit[1000000000]
```

which means

```text
One Billion Elements
```

Even if every integer takes only

```text
4 bytes
```

memory required is

```text
4 × 10^9 bytes

≈4GB
```

Only for one array.

Completely impractical.

---

# 6.3 Can We Store Only Used Values?

Look carefully.

Suppose our array is

```text
1000000000

7

25

999999998
```

How many different values do we actually have?

Only

```text
4
```

Then why should we allocate space for

```text
1,000,000,000
```

positions?

We only need

```text
4
```

positions.

This observation leads to

Coordinate Compression.

---

# 6.4 What Do We Really Care About?

Suppose we ask

```text
How many numbers smaller than 25 exist?
```

Do we actually care that

```text
25
```

is twenty-five?

No.

We only care that

```text
7

<

25

<

999999998

<

1000000000
```

The actual values are irrelevant.

Only their relative order matters.

This is the most important idea.

---

# 6.5 Compressing the Values

Original values

```text
1000000000

7

25

999999998
```

Sort them.

```text
7

25

999999998

1000000000
```

Assign ranks.

```text
7

↓

1
```

```text
25

↓

2
```

```text
999999998

↓

3
```

```text
1000000000

↓

4
```

Now replace every value.

Original

```text
1000000000

7

25

999999998
```

Compressed

```text
4

1

2

3
```

Instead of needing

```text
10^9
```

indices,

we only need

```text
4.
```

---

# 6.6 Why Does This Work?

Fenwick Tree never compares numbers using

```text
Actual Value
```

It only uses

```text
Ordering
```

Suppose

```text
7<25
```

After compression

```text
1<2
```

Still true.

Suppose

```text
25<999999998
```

Compressed

```text
2<3
```

Still true.

Every comparison remains correct.

Therefore,

all BIT operations remain correct.

---

# 6.7 Formal Definition

Coordinate Compression is the process of

> Replacing every distinct value by its position in the sorted order of all distinct values.

Example

| Original | Rank |
|----------|------|
|7|1|
|25|2|
|999999998|3|
|1000000000|4|

The smallest value always becomes

```text
1
```

The second smallest becomes

```text
2
```

and so on.

---

# 6.8 Algorithm

Step 1

Copy the array.

```cpp
vector<int> temp=a;
```

---

Step 2

Sort.

```cpp
sort(temp.begin(),temp.end());
```

---

Step 3

Remove duplicates.

```cpp
temp.erase(unique(temp.begin(),temp.end()),temp.end());
```

---

Step 4

Find rank using binary search.

```cpp
rank=lower_bound(temp.begin(),temp.end(),x)-temp.begin()+1;
```

Notice

```text
+1
```

Fenwick Tree is

```text
1-indexed.
```

---

# 6.9 Complete Function

```cpp
vector<int> compress(vector<int> a)
{
    vector<int> values=a;

    sort(values.begin(),values.end());

    values.erase(unique(values.begin(),values.end()),values.end());

    vector<int> compressed;

    for(int x:a)
    {
        compressed.push_back(
            lower_bound(values.begin(),values.end(),x)
            -values.begin()+1
        );
    }

    return compressed;
}
```

---

# 6.10 Example

Original

```text
8

100

8

20

50
```

Sorted Unique

```text
8

20

50

100
```

Ranks

```text
8→1

20→2

50→3

100→4
```

Compressed Array

```text
1

4

1

2

3
```

Duplicates receive

the same rank.

---

# 6.11 Complexity

Sorting

```text
O(n log n)
```

Finding each rank

```text
O(log n)
```

Total

```text
O(n log n)
```

Memory

```text
O(n)
```

---

# 6.12 When Should You Think About Coordinate Compression?

Whenever you see

```text
Values up to

10^9

10^12

Negative Numbers

Very Sparse Values
```

Immediately ask

> "Do I really care about the values?"

If the answer is

```text
No.

I only care about ordering.
```

Then

Coordinate Compression is almost certainly the solution.

---

# 6.13 Common Mistakes

### Mistake 1

Compress without removing duplicates.

Wrong.

Equal values must receive

the same rank.

---

### Mistake 2

Forget

```text
+1
```

Fenwick Tree cannot use index

```text
0.
```

---

### Mistake 3

Think compression changes the answer.

It doesn't.

It only changes labels.

Relative ordering remains exactly the same.

---

# Chapter Summary

Fenwick Trees require

small integer indices.

Real problems often contain

very large values,

negative numbers,

or sparse values.

Instead of storing the actual values,

we replace each distinct value by its rank.

This process is called

Coordinate Compression.

The most important idea to remember is

```text
Fenwick Tree does not care

about values.

It cares about order.
```

In the next chapter,

we will use Coordinate Compression to solve one of the most famous Fenwick Tree problems:

> **Counting Inversions in O(n log n).**
