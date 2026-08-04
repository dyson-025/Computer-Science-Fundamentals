# Chapter 4 — Range Update + Point Query
*"Changing an entire range without touching every element."*

---

# 4.1 A New Problem Appears

So far, our Fenwick Tree supports

- Point Update
- Prefix Sum Query

This works well when only **one element changes**.

For example,

```text
A[5] += 10
```

The update affects only one position.

Fenwick Tree updates only a few stored ranges.

Everything is fast.

---

But now suppose the problem changes.

Instead of updating one element,

someone asks

```text
Add 10 to every element from index 3 to index 8.
```

Example

Before

```text
Index

1 2 3 4 5 6 7 8

Array

2 5 1 7 3 4 6 8
```

After

```text
2 5 11 17 13 14 16 8
```

---

# 4.2 The Naive Solution

The most obvious solution is

```cpp
for(int i=l;i<=r;i++)
    A[i]+=x;
```

Suppose

```text
n = 100000
```

Worst case

```text
update(1,n)
```

touches

```text
100000 elements.
```

Therefore

```text
One update

=

O(n)
```

If there are

```text
100000 updates
```

Total work becomes

```text
100000 × 100000

=

10¹⁰
```

Impossible.

---

# 4.3 Can We Avoid Updating Every Element?

Think carefully.

Suppose we perform

```text
+5 on [3,7]
```

Do we really need to modify

```text
3

4

5

6

7
```

Or can we simply remember

```text
The increase starts here.

The increase stops here.
```

This idea leads to one of the most important concepts in algorithms.

The **Difference Array**.

---

# 4.4 Difference Array

Instead of storing

```text
Actual values
```

we store

```text
How much each position changes compared to the previous one.
```

Definition

```text
D[1]=A[1]

D[i]=A[i]-A[i-1]
```

Example

Original array

```text
2 5 8 6 9
```

Difference array

```text
2

3

3

-2

3
```

Let's verify.

```text
2

2+3=5

5+3=8

8-2=6

6+3=9
```

The original array comes back perfectly.

---

# 4.5 Why Does Prefix Sum Recover the Original Array?

This is not magic.

Let's derive it.

We know

```text
D2=A2-A1

D3=A3-A2

D4=A4-A3
```

Now add everything.

```text
A1

+

(A2-A1)

+

(A3-A2)

+

(A4-A3)
```

Notice something beautiful.

Everything cancels.

```text
+A1

-A1

+A2

-A2

+A3

-A3
```

Only

```text
A4
```

remains.

Therefore

```text
A[i]

=

PrefixSum(D,i)
```

This is the first important property.

---

# 4.6 How Does Range Update Become Easy?

Suppose

```text
+5 on [3,6]
```

Original

```text
2 5 8 6 9 4 3
```

Difference

```text
2

3

3

-2

3

-5

-1
```

Instead of changing

```text
3

4

5

6
```

observe what actually changes.

The increase begins at

```text
3
```

Therefore

```text
D[3]+=5
```

After index

```text
6
```

the increase should disappear.

So

```text
D[7]-=5
```

That's it.

Only

```text
Two positions
```

change.

Not four.

Not a thousand.

Always two.

---

# 4.7 Why Does This Work?

Think of the prefix sum.

Adding

```text
+5
```

at

```text
3
```

means

```text
Every future prefix

contains

+5
```

So

```text
3

4

5

6

7

8...
```

all become larger.

But we don't want

```text
7

8

9...
```

to increase.

Therefore,

at

```text
r+1
```

we insert

```text
-5
```

The prefix sum becomes

```text
+5

+5

+5

+5

0

0

0
```

Exactly what we wanted.

---

# 4.8 Visual Intuition

Suppose

```text
+4 on [2,5]
```

Difference array

```text
0

+4

0

0

0

-4

0
```

Taking prefix sums

```text
0

4

4

4

4

0

0
```

Notice

The increase automatically spreads until

```text
r
```

and disappears after

```text
r+1
```

without updating every element.

This is the beauty of the difference array.

---

# 4.9 Using Fenwick Tree

A Fenwick Tree already supports

```text
Point Update

Prefix Query
```

Exactly what a difference array needs.

Remember

Difference Array operations are

```text
Point Update

↓

Prefix Query
```

Fenwick Tree operations are

```text
Point Update

↓

Prefix Query
```

Perfect match.

---

# 4.10 Algorithm

Range Update

```cpp
update(l,+x);

update(r+1,-x);
```

Point Query

```cpp
query(i)
```

The prefix sum automatically reconstructs

```text
A[i]
```

---

# 4.11 Complete Implementation

```cpp
struct RangeUpdateBIT
{
    Fenwick bit;

    RangeUpdateBIT(int n):bit(n){}

    void rangeUpdate(int l,int r,int val)
    {
        bit.update(l,val);

        bit.update(r+1,-val);
    }

    int pointQuery(int idx)
    {
        return bit.query(idx);
    }
};
```

---

# 4.12 Example

Initial array

```text
0 0 0 0 0 0
```

Perform

```text
+3 on [2,5]
```

Fenwick actually stores

```text
Index

1 2 3 4 5 6

Difference

0 3 0 0 0 -3
```

Now query

```text
Index 4
```

Fenwick computes

```text
0+3+0+0

=

3
```

Therefore

```text
A4=3
```

Query

```text
Index 6
```

Fenwick computes

```text
0+3+0+0+0-3

=

0
```

Correct.

---

# 4.13 Complexity

Range Update

```text
Two Point Updates

↓

O(log n)
```

Point Query

```text
One Prefix Query

↓

O(log n)
```

Space

```text
O(n)
```

---

# 4.14 The Limitation

Now another interesting question appears.

Suppose someone asks

```text
Sum(3,8)
```

Can we answer

```cpp
query(8)-query(2)
```

No.

Because

```text
query(i)
```

now returns

```text
A[i]
```

NOT

```text
Prefix Sum
```

A single Fenwick Tree reconstructs

```text
Difference

↓

Original Array
```

But

Range Sum needs

```text
Difference

↓

Original Array

↓

Prefix Sum
```

That is

**two prefix operations.**

One Fenwick Tree performs only

**one prefix operation.**

So a new idea is needed.

That idea is

> **Two Fenwick Trees.**

The next chapter derives that idea from scratch.

---

# Chapter Summary

We wanted to support

```text
Range Update

Point Query
```

Updating every element individually was too slow.

Instead,

we introduced the **Difference Array**.

The key observations were

- The original array is the prefix sum of the difference array.
- A range update changes only two positions in the difference array.
- Fenwick Tree already supports point updates and prefix queries.
- Therefore, a single Fenwick Tree over the difference array gives

```text
Range Update

+

Point Query

=

O(log n)
```

However,

it still cannot answer range sums.

Understanding **why** leads naturally to the two-Fenwick Tree technique in the next chapter.
