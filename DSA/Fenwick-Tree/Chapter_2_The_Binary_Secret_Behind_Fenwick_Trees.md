# Chapter 2 — The Binary Secret Behind Fenwick Trees
*"Understanding why the binary trick exists before learning the algorithm."*

---

# 2.1 The Question We Left Unanswered

In Chapter 1, we reached an important conclusion.

A normal array gives

- Fast updates
- Slow queries

A prefix sum array gives

- Fast queries
- Slow updates

Fenwick Tree promises

- Fast updates
- Fast queries

But we never answered one important question.

> **How does a Fenwick Tree decide what information to store?**

This chapter answers that question.

---

# 2.2 Why Can't We Store Random Ranges?

Suppose we have the array

```text
5 3 7 2 6 1 9 4
```

Someone tells us,

> "Store partial sums instead of storing the whole prefix sum."

Immediately another question appears.

**Which partial sums?**

For example, should we store

```text
[1...2]

[3...5]

[6...8]
```

or

```text
[1...4]

[5...8]
```

or

```text
[2...6]

[7...8]
```

There are infinitely many possible ways.

Most of them are useless.

So we need a systematic way of choosing ranges.

---

# 2.3 What Properties Should These Ranges Have?

Before designing the ranges, think about what we actually want.

Suppose someone asks

```text
Prefix Sum(13)
```

We do **not** want to add

```text
13 different ranges.
```

Otherwise the query becomes O(n).

Instead,

we should be able to combine only a few stored ranges.

Similarly,

suppose

```text
A[5] += 10
```

If this element belongs to hundreds of stored ranges,

we would need to update all of them.

Again,

the update becomes O(n).

Therefore, the stored ranges should satisfy two properties.

---

### Property 1

Every prefix sum should be formed using only a few stored ranges.

---

### Property 2

Every element should belong to only a few stored ranges.

---

If both properties are true,

both operations become fast.

The question now becomes

> **Can we design such ranges?**

The answer comes from binary numbers.

---

# 2.4 Looking at Binary

Instead of looking at indices as decimal numbers,

look at them in binary.

| Index | Binary |
|-------:|:------|
|1|0001|
|2|0010|
|3|0011|
|4|0100|
|5|0101|
|6|0110|
|7|0111|
|8|1000|

At first,

nothing seems special.

But observe one thing.

Every number has a **rightmost 1 bit**.

Examples

```text
6

110
```

Rightmost 1

```text
10

=

2
```

---

```text
12

1100
```

Rightmost 1

```text
100

=

4
```

---

```text
20

10100
```

Rightmost 1

```text
100

=

4
```

This rightmost 1 bit turns out to be the key idea behind the Fenwick Tree.

---

# 2.5 Lowest Set Bit

The value represented by the rightmost 1 bit is called the

> **Lowest Set Bit**

Examples

| Number | Binary | Lowest Set Bit |
|---------|--------|----------------|
|1|0001|1|
|2|0010|2|
|3|0011|1|
|4|0100|4|
|5|0101|1|
|6|0110|2|
|7|0111|1|
|8|1000|8|
|12|1100|4|

Notice something interesting.

The lowest set bit is always a power of two.

```text
1

2

4

8

16
```

Never

```text
3

5

6

7
```

---

# 2.6 Finding the Lowest Set Bit

Computers can extract the lowest set bit using

```cpp
lowbit(x) = x & (-x)
```

You do **not** need to memorize this.

Let's see why it works.

Take

```text
12

00001100
```

First find

```text
-12
```

Using two's complement.

Step 1

Invert all bits

```text
11110011
```

Step 2

Add one

```text
11110100
```

Now perform AND.

```text
00001100

11110100

---------

00000100
```

Result

```text
4
```

Exactly the lowest set bit.

Another example.

```text
6

00000110
```

Negative

```text
11111010
```

AND

```text
00000110

11111010

---------

00000010
```

Result

```text
2
```

Again,

the lowest set bit.

Therefore

```cpp
lowbit(x)=x&(-x)
```

extracts the lowest set bit in O(1).

---

# 2.7 What Does the Lowest Set Bit Mean?

This is the most important intuition in the entire Fenwick Tree.

The lowest set bit tells us

> **How many elements this index is responsible for storing.**

Suppose

```text
Index = 8

Binary = 1000
```

Lowest set bit

```text
8
```

This means

```text
Index 8 stores the sum of 8 elements.
```

---

Suppose

```text
Index = 6

Binary = 110
```

Lowest set bit

```text
2
```

This means

```text
Index 6 stores the sum of 2 elements.
```

---

Suppose

```text
Index = 7

Binary = 111
```

Lowest set bit

```text
1
```

This means

```text
Index 7 stores only one element.
```

Suddenly,

every index automatically knows

how large its stored block should be.

No extra information is needed.

---

# 2.8 Deriving the Stored Range

Suppose

```text
Index = 6
```

Lowest set bit

```text
2
```

So this index stores

```text
2 elements.
```

Since every block always ends at its own index,

the ending position is

```text
6
```

The starting position is therefore

```text
6-2+1

=

5
```

Hence

```text
Tree[6]

stores

Sum(5...6)
```

---

Another example.

```text
Index = 12
```

Lowest set bit

```text
4
```

Block size

```text
4
```

Ending position

```text
12
```

Starting position

```text
12-4+1

=

9
```

Therefore

```text
Tree[12]

stores

Sum(9...12)
```

From these examples,

we obtain the general formula.

```text
Tree[i]

stores

[i-lowbit(i)+1 , i]
```

This formula is not memorized.

It is simply

```text
Ending Index

-

Block Size

+

1
```

---

# 2.9 Examples

| Index | lowbit | Stored Range |
|-------:|--------:|-------------|
|1|1|1...1|
|2|2|1...2|
|3|1|3...3|
|4|4|1...4|
|5|1|5...5|
|6|2|5...6|
|7|1|7...7|
|8|8|1...8|
|12|4|9...12|

Study this table carefully.

Almost every Fenwick Tree algorithm comes directly from these ranges.

---

# Chapter Summary

This chapter answered one important question.

> **How does a Fenwick Tree decide what to store?**

The answer is based entirely on binary representation.

Every index has a

```text
Lowest Set Bit
```

which tells us

```text
How many elements this index is responsible for.
```

The block always

- ends at the current index,
- has size equal to the lowest set bit.

Therefore,

```text
Stored Range

=

[i-lowbit(i)+1 , i]
```

This simple observation is the foundation of the entire Fenwick Tree.

In the next chapter, we will derive the two most famous Fenwick Tree algorithms:

```cpp
query()

update()
```

Instead of memorizing

```cpp
i -= lowbit(i)

i += lowbit(i)
```

we will derive them from the stored ranges themselves.
