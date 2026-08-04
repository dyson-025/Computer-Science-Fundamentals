# Chapter 7 — Fenwick Tree Indexed by Value
*"The biggest shift in thinking while solving Fenwick Tree problems."*

---

# 7.1 Everything Changes Here

Until now,

our Fenwick Tree looked like this.

```
Array Index

↓

Fenwick Tree Index
```

Example

```
Array

5 2 8 1
```

Fenwick Tree

```
Index

1

2

3

4
```

Every Fenwick Tree node represented

an **array position**.

---

Now suppose the interviewer asks

> "How many numbers smaller than 8 have appeared so far?"

Notice something.

This question is **not about positions.**

It is about

```
Values.
```

This changes everything.

---

# 7.2 Why Position No Longer Matters

Suppose

```
8 3 5 2
```

Current element

```
5
```

Question

```
How many numbers smaller than 5
have already appeared?
```

Look carefully.

We don't care

where those numbers are.

We only care

whether

```
Value

<5
```

This is the first major mindset shift.

Fenwick Tree will no longer be indexed

by

```
Position
```

Instead,

it will be indexed by

```
Value.
```

---

# 7.3 The New Fenwick Tree

Suppose values are

```
1

2

3

4

5

6

7

8
```

Fenwick Tree

```
Index

1

2

3

4

5

6

7

8
```

does NOT mean

```
Array Position.
```

It now means

```
Value.
```

Example

```
BIT[5]
```

means

```
Information about value 5
```

not

```
Information about position 5.
```

---

# 7.4 What Do We Store?

Suppose

```
8

3

5

2
```

We process from left to right.

Initially

Fenwick Tree

```
empty
```

Read

```
8
```

Insert

```
Value 8

Count++

```

Fenwick now stores

```
Value

1 2 3 4 5 6 7 8

Count

0 0 0 0 0 0 0 1
```

---

Read

```
3
```

Insert

```
Value 3

Count++
```

Now

```
Value

1 2 3 4 5 6 7 8

Count

0 0 1 0 0 0 0 1
```

Notice

we are storing

```
Frequency.
```

---

# 7.5 What Does Query Mean Now?

Suppose current value is

```
5
```

We ask

```
query(4)
```

Why

```
4
```

Because

```
All values

<5
```

are

```
1

2

3

4
```

Fenwick returns

```
How many numbers

≤4

have already appeared.
```

Exactly what we need.

---

# 7.6 General Pattern

Whenever you see

```
How many numbers

smaller than x

greater than x

less than or equal

greater than or equal
```

Immediately think

```
Fenwick Tree

indexed by value.
```

Not

indexed by position.

---

# 7.7 Why Coordinate Compression Was Necessary

Suppose

```
Value

1000000000
```

Can we create

```
BIT[1000000000]
```

No.

This is exactly why

Chapter 6

came before this one.

We first compress

```
1000000000

↓

4
```

Now

Fenwick Tree size

```
4
```

instead of

```
1000000000
```

---

# 7.8 The Universal Processing Pattern

Almost every advanced BIT problem follows the same four steps.

### Step 1

Coordinate Compress.

---

### Step 2

Create Fenwick Tree.

---

### Step 3

Process elements

either

```
Left → Right
```

or

```
Right → Left
```

depending on the question.

---

### Step 4

For every element

```
Query First

↓

Update Later
```

or

```
Update First

↓

Query Later
```

depending on whether the current element should count itself.

---

# 7.9 Why Query Before Update?

Suppose

Current value

```
5
```

Question

```
How many previous values

are smaller than 5?
```

Should

```
5
```

count itself?

No.

Therefore

```
Query

↓

Update
```

is correct.

If we updated first,

the current element would incorrectly count itself.

This small detail causes many bugs.

---

# 7.10 The Master Pattern

Almost every Fenwick Tree interview problem becomes

```
Coordinate Compression

↓

BIT indexed by Value

↓

Sweep Array

↓

Query

↓

Update
```

Only the direction changes.

---

# Chapter Summary

Until now,

Fenwick Tree was indexed by

```
Array Position.
```

From this chapter onward,

Fenwick Tree will usually be indexed by

```
Value.
```

This single idea solves

- Inversion Count
- Count Smaller After Self
- Reverse Pairs
- Create Sorted Array Through Instructions
- Count Range Sum (with modification)

The next chapter applies this exact pattern to solve

the famous

**Inversion Count** problem in O(n log n).
