# 1D Dynamic Programming --- Complete Interview Notes

> **Theme:** The answer at position `i` depends on one or more
> previously computed positions.

------------------------------------------------------------------------

# 1. How to Recognize 1D DP

Typical clues:

-   Count ways
-   Minimum cost
-   Maximum sum
-   Reach the end
-   Choices at every index
-   "From index i..."

General state:

``` cpp
dp[i]
```

Meaning:

> Answer considering index `i` (or first `i` elements).

------------------------------------------------------------------------

# 2. Standard Workflow

1.  Write recursion.
2.  Identify changing parameters → DP state.
3.  Memoize.
4.  Convert to tabulation.
5.  Space optimize if only previous states are used.

------------------------------------------------------------------------

# 3. Climbing Stairs

## State

``` cpp
dp[i] = ways to reach step i
```

Transition

``` cpp
dp[i] = dp[i-1] + dp[i-2];
```

Base

``` cpp
dp[0]=1;
dp[1]=1;
```

Complexity

-   Time: O(N)
-   Space: O(1)

------------------------------------------------------------------------

# 4. Frog Jump

## State

``` cpp
dp[i] = minimum energy to reach i
```

Choices

-   Jump from `i-1`
-   Jump from `i-2`

Transition

``` cpp
dp[i]=min(
dp[i-1]+abs(h[i]-h[i-1]),
dp[i-2]+abs(h[i]-h[i-2])
);
```

Base

``` cpp
dp[0]=0;
```

Complexity

-   O(N)

------------------------------------------------------------------------

# 5. Frog Jump with K Distance

Generalization of Frog Jump.

Transition

``` cpp
dp[i]=INF;

for(int j=1;j<=k;j++){

    if(i-j>=0)

        dp[i]=min(dp[i],
                  dp[i-j]
                  +abs(h[i]-h[i-j]));
}
```

Complexity

-   Time: O(NK)
-   Space: O(N)

------------------------------------------------------------------------

# 6. Maximum Sum of Non Adjacent Elements

## State

``` cpp
dp[i]
=
maximum sum using first i elements
```

Choices

### Pick

``` cpp
arr[i]+dp[i-2]
```

### Not Pick

``` cpp
dp[i-1]
```

Transition

``` cpp
dp[i]=max(
pick,
notPick
);
```

Base

``` cpp
dp[0]=arr[0];
```

Space Optimization

``` cpp
prev2
prev1
```

------------------------------------------------------------------------

# 7. House Robber

Same recurrence as Maximum Sum of Non Adjacent.

Difference:

Array is circular.

Cannot rob

    First
    and
    Last

simultaneously.

Solution

Solve twice.

    Exclude first

and

    Exclude last

Answer

``` cpp
max(
solve(1,n-1),
solve(0,n-2)
)
```

Complexity

-   O(N)

------------------------------------------------------------------------

# 8. Universal Decision Pattern

Every 1D DP generally looks like

``` text
Current Index

│

├── Take

└── Skip
```

or

``` text
Come from

i-1

i-2

...

i-k
```

------------------------------------------------------------------------

# 9. Interview Templates

## Take / Skip DP

``` cpp
pick = value + dp[i-2];

notPick = dp[i-1];

dp[i]=max(pick,notPick);
```

------------------------------------------------------------------------

## Previous-State DP

``` cpp
dp[i]=combine(
dp[i-1],
dp[i-2]
);
```

------------------------------------------------------------------------

# 10. Pattern Summary

  Problem            State      Transition
  ------------------ ---------- --------------------------------
  Climbing Stairs    Ways       dp\[i-1\]+dp\[i-2\]
  Frog Jump          Min Cost   min(i-1,i-2)
  Frog K Jump        Min Cost   min(last K states)
  Max Non Adjacent   Max Sum    Pick / Skip
  House Robber       Max Sum    Pick / Skip on two linear runs

------------------------------------------------------------------------

# 11. Common Mistakes

-   Wrong base cases.
-   Forgetting bounds (`i-2`, `i-k`).
-   Not space optimizing when only previous states are needed.
-   Forgetting House Robber is circular.

------------------------------------------------------------------------

# 12. One-Minute Revision

    Climbing
    → Count Ways

    Frog
    → Minimum Cost

    Frog K
    → Previous K states

    Max Non Adjacent
    → Pick / Skip

    House Robber
    → Pick / Skip
    + Solve twice

------------------------------------------------------------------------

# 13. Takeaway

Most 1D DP problems reduce to one of two ideas:

1.  **Reach current index from previous indices** (Climbing Stairs, Frog
    Jump).
2.  **Take or Skip the current element** (Maximum Sum of Non-Adjacent,
    House Robber).

Always define the meaning of `dp[i]` first, then derive the recurrence
from the available choices.
