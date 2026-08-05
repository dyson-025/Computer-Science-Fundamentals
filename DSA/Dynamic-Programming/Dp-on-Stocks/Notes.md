# DP on Stocks --- Complete Interview Notes

> **Pattern:** Decision DP\
> **Core Idea:** Every day, make one decision based on your current
> state.

------------------------------------------------------------------------

# 1. Pattern Recognition

Whenever a problem contains:

-   Buy / Sell
-   Maximum Profit
-   Transactions
-   Cooldown
-   Transaction Fee

Think:

``` text
Dynamic Programming on Decisions
```

Every day you can only perform one meaningful action.

    Buy
    Sell
    Skip

Never think about all possible sequences.

Instead ask:

> **"What are my choices today?"**

------------------------------------------------------------------------

# 2. Universal DP State

Almost every stock problem can be modeled as

``` cpp
dp[day][state][extra]
```

where

-   `day` → Current index
-   `state`
    -   `0` → Can Buy
    -   `1` → Holding Stock
-   `extra`
    -   Transactions Left
    -   Cooldown
    -   Fee (usually handled directly in transition)

------------------------------------------------------------------------

# 3. Universal Decisions

## State = Can Buy

### Choices

### Skip

``` cpp
skipBuy = nextCanBuy;
```

### Buy

``` cpp
buyToday = -price + nextHolding;
```

Transition

``` cpp
dpCanBuy = max(skipBuy, buyToday);
```

------------------------------------------------------------------------

## State = Holding Stock

### Choices

### Skip

``` cpp
skipSell = nextHolding;
```

### Sell

``` cpp
sellToday = price + nextCanBuy;
```

Transition

``` cpp
dpHolding = max(skipSell, sellToday);
```

------------------------------------------------------------------------

# 4. Golden Rule

**Transactions decrease only after SELLING.**

Never while buying.

Reason:

Buying starts a transaction.

Selling completes it.

------------------------------------------------------------------------

# 5. Stock Variants

## Stock I

-   Only one transaction.
-   Best solved greedily.

Time: **O(N)**

------------------------------------------------------------------------

## Stock II (Unlimited Transactions)

State

``` cpp
dp[day][holding]
```

Transition

``` text
Can Buy
 ├── Skip
 └── Buy

Holding
 ├── Skip
 └── Sell
```

------------------------------------------------------------------------

## Stock III (At most 2 transactions)

State

``` cpp
dp[day][transactionsLeft][holding]
```

------------------------------------------------------------------------

## Stock IV (At most K transactions)

Exactly same as Stock III.

Only

``` cpp
transactionsLeft = K
```

------------------------------------------------------------------------

## Stock with Fee

Only selling changes.

Instead of

``` cpp
sell = price + nextCanBuy;
```

Use

``` cpp
sell = price - fee + nextCanBuy;
```

------------------------------------------------------------------------

## Stock with Cooldown

Selling jumps two days.

Instead of

``` cpp
price + dp[day+1][buy]
```

Use

``` cpp
price + dp[day+2][buy]
```

------------------------------------------------------------------------

# 6. Space Optimization

Most stock DP depends only on the next day.

    day
    ↓

    day+1

Hence

    O(N)
    ↓

    O(1)

Cooldown requires

    day+2

so keep

``` cpp
after1
after2
```

------------------------------------------------------------------------

# 7. Interview Template (Unlimited Transactions)

``` cpp
class Solution {
public:
    int maxProfit(vector<int>& prices) {

        vector<int> curr(2,0), next(2,0);

        for(int day = prices.size()-1; day>=0; day--){

            int buyToday = -prices[day] + next[1];
            int skipBuy = next[0];

            curr[0] = max(skipBuy, buyToday);

            int sellToday = prices[day] + next[0];
            int skipSell = next[1];

            curr[1] = max(skipSell, sellToday);

            next = curr;
        }

        return next[0];
    }
};
```

------------------------------------------------------------------------

# 8. Interview Template (K Transactions)

``` cpp
class Solution {
public:
    int maxProfit(int k, vector<int>& prices) {

        int n = prices.size();

        vector<vector<vector<int>>> dp(
            n+1,
            vector<vector<int>>(k+1, vector<int>(2,0))
        );

        for(int day=n-1; day>=0; day--){

            for(int t=1; t<=k; t++){

                int skipBuy = dp[day+1][t][0];
                int buyToday = -prices[day] + dp[day+1][t][1];

                dp[day][t][0] = max(skipBuy, buyToday);

                int skipSell = dp[day+1][t][1];
                int sellToday = prices[day] + dp[day+1][t-1][0];

                dp[day][t][1] = max(skipSell, sellToday);
            }
        }

        return dp[0][k][0];
    }
};
```

------------------------------------------------------------------------

# 9. Interview Template (Fee)

Only one line changes.

``` cpp
sellToday = prices[day] - fee + nextCanBuy;
```

------------------------------------------------------------------------

# 10. Interview Template (Cooldown)

Only one line changes.

``` cpp
sellToday = prices[day] + after2[0];
```

------------------------------------------------------------------------

# 11. Complexity

  Problem     Time     Space
  ----------- -------- -------
  Stock I     O(N)     O(1)
  Stock II    O(N)     O(1)
  Stock III   O(N×2)   O(1)
  Stock IV    O(N×K)   O(K)
  Cooldown    O(N)     O(1)
  Fee         O(N)     O(1)

------------------------------------------------------------------------

# 12. Common Interview Mistakes

-   Decreasing transaction count while buying.
-   Forgetting cooldown uses `day + 2`.
-   Applying fee on the wrong transition (be consistent).
-   Mixing up `canBuy` and `holding`.

------------------------------------------------------------------------

# 13. One-Minute Revision

    State:
    (day, canBuy, transactionsLeft)

    Can Buy
    ---------
    Skip
    Buy

    Holding
    ---------
    Skip
    Sell

    Sell
    → Transaction--

    Cooldown
    → day+2

    Fee
    → price-fee

    Unlimited
    → Remove transaction dimension

------------------------------------------------------------------------

# 14. Takeaway

All stock DP problems are the **same DP**.

Only one transition changes depending on the constraint.

If you derive the recurrence from **today's decisions** instead of
memorizing formulas, you can solve unseen stock variants confidently in
interviews.
