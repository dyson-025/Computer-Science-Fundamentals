# Chapter 5 — Range Update + Range Sum (Two Fenwick Trees)
*"Why one Fenwick Tree is no longer enough."*

---

# 5.1 A New Challenge

In the previous chapter, we solved

```text
Range Update

+

Point Query
```

using

- Difference Array
- One Fenwick Tree

The operations became

```text
Range Update → O(log n)

Point Query → O(log n)
```

Everything looked perfect.

But now the problem changes again.

Instead of asking

```text
What is A[5]?
```

someone asks

```text
What is the sum from index 3 to 8?
```

Immediately a question appears.

Can one Fenwick Tree still solve it?

---

# 5.2 Our First Thought

We already know

```text
query(i)
```

returns

```text
A[i]
```

So maybe

```cpp
query(r)-query(l-1)
```

should work.

Let's test it.

Suppose

```text
Array

0 5 5 5 0
```

This came from

```text
+5 on [2,4]
```

Difference array stored inside BIT

```text
0 5 0 0 -5
```

Now ask

```text
Sum(2,4)
```

Actual answer

```text
5+5+5

=

15
```

Now use

```cpp
query(4)-query(1)
```

Fenwick returns

```text
5-0

=

5
```

Wrong.

---

# 5.3 Why Did It Fail?

Remember what our Fenwick Tree stores.

It stores the

```text
Difference Array
```

Query performs

```text
Prefix(Difference)
```

which reconstructs

```text
Original Array
```

Therefore

```text
query(i)

=

A[i]
```

NOT

```text
A1+A2+...+Ai
```

Subtracting two point values

can never produce

a range sum.

The problem is not the formula.

The problem is that

our query returns the wrong thing.

---

# 5.4 What Do We Actually Need?

Suppose we want

```text
Prefix Sum(5)
```

That means

```text
A1+A2+A3+A4+A5
```

But

```text
A[i]

=

Prefix(D)
```

Substitute every A.

```text
A1=D1

A2=D1+D2

A3=D1+D2+D3

A4=D1+D2+D3+D4

A5=D1+D2+D3+D4+D5
```

Now add them.

```text
P(5)

=

D1

+

(D1+D2)

+

(D1+D2+D3)

+

(D1+D2+D3+D4)

+

(D1+D2+D3+D4+D5)
```

This is the key step.

---

# 5.5 Expand Carefully

Instead of looking row by row,

group equal terms.

```text
D1 appears

5 times
```

```text
D2 appears

4 times
```

```text
D3 appears

3 times
```

```text
D4 appears

2 times
```

```text
D5 appears

1 time
```

Therefore

```text
P(5)

=

5D1

+

4D2

+

3D3

+

2D4

+

D5
```

This is much more informative.

Let's generalize it.

---

# 5.6 General Formula

For Prefix Sum(i)

```text
P(i)

=

iD1

+

(i-1)D2

+

(i-2)D3

+

...

+

Di
```

Notice the coefficient.

For

```text
Dj
```

the coefficient is

```text
i-j+1
```

Therefore

```text
P(i)

=

Σ Dj(i-j+1)
```

This formula is the heart of the two Fenwick Tree technique.

---

# 5.7 Splitting the Formula

Now rewrite

```text
(i-j+1)
```

as

```text
(i+1)-j
```

Substitute.

```text
P(i)

=

ΣDj((i+1)-j)
```

Expand.

```text
P(i)

=

ΣDj(i+1)

-

Σ(jDj)
```

Take

```text
(i+1)
```

outside.

```text
P(i)

=

(i+1)ΣDj

-

Σ(jDj)
```

Now stop.

Look carefully.

How many different prefix sums do we need?

---

# 5.8 The Big Observation

The formula contains

```text
ΣDj
```

and

```text
Σ(jDj)
```

These are two completely different prefix sums.

One Fenwick Tree can maintain

only one prefix sum.

Therefore

one Fenwick Tree is no longer enough.

We need

```text
Two Fenwick Trees.
```

This is the real reason behind the technique.

Not because someone invented a clever formula.

Because mathematically,

two different prefix sums must be maintained.

---

# 5.9 What Does Each Fenwick Tree Store?

The first Fenwick Tree stores

```text
Difference Array

Dj
```

Exactly the same as Chapter 4.

The second Fenwick Tree stores

```text
Index × Difference

j × Dj
```

Now

```text
ΣDj
```

comes from

```text
BIT1
```

and

```text
Σ(jDj)
```

comes from

```text
BIT2
```

Combining both,

we reconstruct

the prefix sum.

---

# 5.10 Why Does the Implementation Look Different?

Our derivation produced

```text
P(i)

=

(i+1)ΣDj

-

Σ(jDj)
```

Most implementations write

```cpp
Prefix(i)

=

i×BIT1.query(i)

-

BIT2.query(i)
```

Why?

Because the second BIT is stored in a slightly shifted form.

Instead of updating

```text
j×Dj
```

directly,

we update

```cpp
BIT2.update(l,val*(l-1));

BIT2.update(r+1,-val*r);
```

This shift absorbs the extra

```text
+1
```

making the implementation shorter.

The mathematics remains exactly the same.

---

# 5.11 Range Update

Suppose

```text
Add x on [l,r]
```

Difference array changes only at

```text
l

r+1
```

BIT1

```cpp
update(l,+x);

update(r+1,-x);
```

BIT2

```cpp
update(l,+x*(l-1));

update(r+1,-x*r);
```

Only four point updates.

Still

```text
O(log n)
```

---

# 5.12 Prefix Query

Compute

```cpp
long long prefix(int i)
{
    return i*BIT1.query(i)-BIT2.query(i);
}
```

This returns

```text
A1+A2+...+Ai
```

Finally,

Range Sum becomes

```cpp
Range(l,r)

=

Prefix(r)

-

Prefix(l-1)
```

Exactly like an ordinary prefix sum array.

---

# 5.13 Complete Implementation

```cpp
struct RangeBIT
{
    Fenwick B1,B2;

    RangeBIT(int n):B1(n),B2(n){}

    void rangeUpdate(int l,int r,long long val)
    {
        B1.update(l,val);
        B1.update(r+1,-val);

        B2.update(l,val*(l-1));
        B2.update(r+1,-val*r);
    }

    long long prefix(int idx)
    {
        return idx*B1.query(idx)-B2.query(idx);
    }

    long long rangeQuery(int l,int r)
    {
        return prefix(r)-prefix(l-1);
    }
};
```

---

# 5.14 Understanding the Whole Picture

Think of the data flowing through transformations.

## Normal Fenwick Tree

```text
Array

↓

Prefix

↓

Answer
```

One prefix.

One Fenwick Tree.

---

## Difference Array Fenwick Tree

```text
Difference

↓

Original Array
```

Again,

one prefix.

One Fenwick Tree.

---

## Range Update + Range Sum

```text
Difference

↓

Original Array

↓

Prefix Sum
```

Now there are

**two prefix operations.**

One Fenwick Tree performs

one prefix operation.

Therefore

```text
Two Prefixes

↓

Two Fenwick Trees
```

This is the easiest way to remember the technique.

---

# Chapter Summary

In this chapter,

we discovered why one Fenwick Tree fails.

The important realization was

```text
query(i)

=

A[i]
```

instead of

```text
A1+A2+...+Ai
```

To compute range sums,

we expanded the mathematics and derived

```text
P(i)

=

(i+1)ΣDj

-

Σ(jDj)
```

This formula revealed

two independent prefix sums,

forcing us to maintain

two Fenwick Trees.

Remember this intuition.

```text
One Prefix

↓

One BIT
```

```text
Two Prefixes

↓

Two BITs
```

Never memorize the formula.

Understand **why** it appears.
