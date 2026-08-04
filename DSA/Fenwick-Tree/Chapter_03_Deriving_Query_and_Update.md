# Chapter 3 — Deriving Query() and Update()
*"Don't memorize the algorithm. Discover it."*

---

# 3.1 Where We Left Off

In the previous chapter, we discovered something remarkable.

Every index stores a range.

The range is determined by

```text
lowbit(index)
```

For example

| Index | Stored Range |
|-------:|-------------|
|1|1...1|
|2|1...2|
|3|3...3|
|4|1...4|
|5|5...5|
|6|5...6|
|7|7...7|
|8|1...8|

Notice something.

These ranges overlap.

Yet every range has a clear purpose.

Now the question becomes

> **How can we use these stored ranges to answer prefix sums?**

---

# 3.2 The Goal

Suppose someone asks

```text
Prefix Sum(7)
```

which means

```text
A1+A2+A3+A4+A5+A6+A7
```

Should we simply add

```text
Tree[1]

Tree[2]

Tree[3]

...

Tree[7]
```

No.

That would again become O(n).

We need something smarter.

---

# 3.3 Think Like a Human

Forget algorithms.

Suppose you have these stored ranges.

```text
Tree[1] = 1

Tree[2] = 1...2

Tree[3] = 3

Tree[4] = 1...4

Tree[5] = 5

Tree[6] = 5...6

Tree[7] = 7

Tree[8] = 1...8
```

Now someone asks

```text
Prefix Sum(7)
```

How would YOU answer?

Start from the end.

```
Need

1.........7
```

Immediately notice

```
Tree[7]

already gives

7
```

Take it.

Remaining part

```
1.........6
```

---

# 3.4 Continue the Same Idea

Now we need

```
1.........6
```

Look at index

```
6
```

Tree[6] stores

```
5.........6
```

Perfect.

Take it.

Now remaining

```
1.........4
```

---

Again

Tree[4]

already stores

```
1.........4
```

Done.

We used only

```
Tree[7]

Tree[6]

Tree[4]
```

instead of

```
7 values.
```

Amazing.

---

# 3.5 What Just Happened?

Let's observe carefully.

Started at

```
7
```

Then moved to

```
6
```

Then moved to

```
4
```

How did we know where to move?

Let's investigate.

---

# 3.6 The Pattern

Current index

```
7

lowbit=1
```

Subtract

```
7-1

=

6
```

---

Current index

```
6

lowbit=2
```

Subtract

```
6-2

=

4
```

---

Current index

```
4

lowbit=4
```

Subtract

```
4-4

=

0
```

Stop.

Notice the pattern.

Every time,

we simply removed the block we had already taken.

Therefore,

the next index is

```cpp
i = i - lowbit(i)
```

This is exactly

```cpp
i -= i & (-i)
```

We did not memorize it.

We discovered it.

---

# 3.7 Deriving Query()

Now the algorithm becomes obvious.

```cpp
sum = 0

while(i>0)
{
    sum += Tree[i];

    i -= lowbit(i);
}
```

Every iteration removes one stored block.

Eventually,

nothing remains.

---

# 3.8 Example

Find

```
Prefix Sum(13)
```

Binary

```
13

1101
```

lowbit

```
1
```

Take

```
13...13
```

Remaining

```
1...12
```

---

Move

```
13→12
```

Tree[12]

stores

```
9...12
```

Remaining

```
1...8
```

---

Move

```
12→8
```

Tree[8]

stores

```
1...8
```

Done.

Visited

```
13

12

8
```

Only

```
3 indices.
```

---

# 3.9 Why Does Query Take O(log n)?

Every subtraction removes the lowest set bit.

Example

```
13

1101
```

↓

```
12

1100
```

↓

```
8

1000
```

↓

```
0
```

Each iteration removes one set bit.

A number has at most

```
log₂(n)
```

bits.

Therefore,

Query takes

```
O(log n)
```

---

# 3.10 Now Let's Derive Update()

Suppose

```
A5

changes.
```

Which Tree nodes should change?

Not every node.

Only those ranges containing index 5.

Let's find them.

Tree[5]

stores

```
5
```

Needs update.

---

Tree[6]

stores

```
5...6
```

Contains 5.

Update.

---

Tree[8]

stores

```
1...8
```

Contains 5.

Update.

---

Tree[16]

stores

```
1...16
```

Contains 5.

Update.

Pattern

```
5

↓

6

↓

8

↓

16
```

---

# 3.11 Another Pattern

Current index

```
5

lowbit=1
```

Next

```
5+1

=

6
```

---

Current

```
6

lowbit=2
```

Next

```
6+2

=

8
```

---

Current

```
8

lowbit=8
```

Next

```
8+8

=

16
```

Therefore

```cpp
i += lowbit(i)
```

Exactly

```cpp
i += i&(-i)
```

Again,

we discovered it ourselves.

---

# 3.12 Update Algorithm

```cpp
while(i<=n)
{
    Tree[i]+=value;

    i+=lowbit(i);
}
```

Every jump moves to the next larger block containing the current index.

---

# 3.13 Complete Implementation

```cpp
struct Fenwick
{
    int n;

    vector<long long> tree;

    Fenwick(int n)
    {
        this->n=n;

        tree.assign(n+1,0);
    }

    void update(int i,long long val)
    {
        while(i<=n)
        {
            tree[i]+=val;

            i+=i&(-i);
        }
    }

    long long query(int i)
    {
        long long sum=0;

        while(i>0)
        {
            sum+=tree[i];

            i-=i&(-i);
        }

        return sum;
    }

    long long rangeQuery(int l,int r)
    {
        return query(r)-query(l-1);
    }
};
```

---

# 3.14 The Big Picture

Notice how beautiful the symmetry is.

## Query

Moves

```
Right

↓

Left
```

because we keep removing blocks already counted.

```cpp
i-=lowbit(i)
```

---

## Update

Moves

```
Small Block

↓

Bigger Block
```

because every larger block also contains this element.

```cpp
i+=lowbit(i)
```

One algorithm walks **backward**.

The other walks **forward**.

Both use exactly the same binary trick.

---

# Chapter Summary

This chapter derived the two most famous Fenwick Tree algorithms.

We never memorized

```cpp
i-=i&(-i)
```

or

```cpp
i+=i&(-i)
```

Instead,

we discovered them from the stored ranges.

Remember these two intuitions.

### Query

> Remove the block you have already used.

### Update

> Move to the next bigger block that also contains this index.

Everything else in the Fenwick Tree is built on these two simple ideas.
