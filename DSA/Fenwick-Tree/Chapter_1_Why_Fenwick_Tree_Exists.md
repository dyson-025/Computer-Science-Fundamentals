# Chapter 1 — Why Fenwick Tree Exists?
*"Before learning the solution, understand the problem."*

---

# 1.1 Imagine You're Building a Bank App

Suppose you are building a simple banking application.

Each element of an array represents the money deposited on a particular day.

```text
Day :   1   2   3   4   5
Money: 10  20  15   5  30
```

Now customers continuously ask questions like:

> "How much money has been deposited from Day 1 to Day 4?"

which means

```text
10 + 20 + 15 + 5 = 50
```

At the same time, new deposits also happen.

Example:

```text
Day 3 deposit increases by 10.
```

Now the array becomes

```text
10 20 25 5 30
```

Notice something important.

We are receiving **two completely different types of operations.**

---

## Operation 1

Someone changes a value.

```text
Update Day 3 by +10
```

---

## Operation 2

Someone asks

```text
What is the sum from Day 1 to Day 4?
```

---

Now imagine this happening

```text
100,000 updates

100,000 queries
```

Suddenly the problem becomes much harder.

---

# 1.2 First Idea — Just Store the Array

The most natural solution is

```cpp
vector<int> a;
```

Whenever someone asks

```text
Sum(1,4)
```

simply do

```cpp
int sum = 0;
for(int i=1;i<=4;i++)
    sum += a[i];
```

Simple. Easy. Correct.

But a worst-case query takes **O(n)**.

---

# 1.3 Second Idea — Prefix Sum Array

Instead of storing

```text
10 20 15 5 30
```

store

```text
10
30
45
50
80
```

where

```text
prefix[i] = sum of first i elements
```

Now any range sum becomes

```text
Prefix(r) - Prefix(l-1)
```

Queries are now **O(1)**.

Unfortunately, updating one element forces every later prefix to change.

Updates become **O(n)**.

---

# 1.4 Compare Both Approaches

| Approach | Update | Query |
|-----------|:------:|:-----:|
| Normal Array | O(1) | O(n) |
| Prefix Sum Array | O(n) | O(1) |

Each solves one problem while creating another.

---

# 1.5 The Real Question

Instead of choosing between them, ask:

> **Can we make both updates and queries fast?**

Our goal becomes

| Operation | Desired Complexity |
|-----------|--------------------|
| Update | O(log n) |
| Query | O(log n) |

---

# 1.6 Why O(log n)?

For

```text
n = 100000
```

we have

```text
log2(100000) ≈ 17
```

Instead of touching **100000** elements, we touch only about **17**.

That is the power of logarithmic algorithms.

---

# 1.7 The Big Idea

A Fenwick Tree stores neither

- every array element, nor
- every prefix sum.

Instead, it stores **carefully chosen partial sums**.

These partial sums overlap in a clever way so that:

- Updates modify only a few stored values.
- Queries combine only a few stored values.

The entire trick depends on binary numbers.

---

# 1.8 What's Next?

The next question is:

> **How does a Fenwick Tree decide which partial sums to store?**

The answer begins with one tiny binary operation:

```text
lowbit(x) = x & (-x)
```

This single expression is the heart of the Fenwick Tree.

The next chapter explains why it works.

---

# Chapter Summary

- A normal array has fast updates but slow queries.
- A prefix sum array has fast queries but slow updates.
- Fenwick Tree balances both in **O(log n)**.
- It achieves this by storing carefully selected partial sums based on binary representation.
