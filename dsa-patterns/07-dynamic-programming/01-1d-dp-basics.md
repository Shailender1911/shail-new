# Pattern: 1D DP Basics (Linear Dynamic Programming)

## How to Identify This Pattern

### Trigger Words
- "minimum/maximum number of ways"
- "can you reach", "how many ways to reach"
- "minimum cost", "minimum steps"
- "fibonacci", "climbing stairs"
- "at each step you can choose"
- "rob houses", "jump to next"
- "decode", "count combinations"

### Constraint Clues
- Input is a single array or a single integer `n`
- The answer at position `i` depends only on previous positions (i-1, i-2, etc.)
- Greedy doesn't work because a locally optimal choice may not be globally optimal
- Brute-force recursion has overlapping subproblems (same state computed many times)
- Constraints like `n <= 10^4` or `n <= 10^5` hint at O(n) or O(n * k)

### When to Use This vs Similar Patterns
| Consideration | 1D DP | 2D DP | Greedy |
|---|---|---|---|
| State depends on | 1 variable (index or amount) | 2 variables (i, j) | Local info only |
| Decision at each step | Pick from fixed set of transitions | Two dimensions of choice | Always take best local choice |
| Overlapping subproblems? | Yes | Yes | No (or greedy proof exists) |
| Typical structure | `dp[i] = f(dp[i-1], dp[i-2], ...)` | `dp[i][j] = f(dp[i-1][j], dp[i][j-1], ...)` | Sort + scan |

## Core Approach

1. **Define state:** `dp[i]` = the answer (min cost, number of ways, reachability) considering elements `0..i` or representing "amount i".
2. **Write recurrence:** Express `dp[i]` in terms of smaller subproblems.
   - Fibonacci-style: `dp[i] = dp[i-1] + dp[i-2]`
   - Decision-style: `dp[i] = max(dp[i-1], dp[i-2] + value[i])` (take or skip)
   - Min-cost-style: `dp[i] = min over all transitions + cost`
3. **Base cases:** Initialize `dp[0]`, `dp[1]`, etc. based on the problem definition.
4. **Build order:** Fill left to right (increasing `i`).
5. **Answer location:** Usually `dp[n-1]` or `dp[n]` depending on 0-indexed or 1-indexed formulation.

## Java Template

### Bottom-Up (Tabulation)
```java
public int solve(int[] nums) {
    int n = nums.length;
    if (n == 0) return 0;
    if (n == 1) return nums[0];

    int[] dp = new int[n];
    dp[0] = nums[0];
    dp[1] = Math.max(nums[0], nums[1]);

    for (int i = 2; i < n; i++) {
        dp[i] = Math.max(dp[i - 1], dp[i - 2] + nums[i]);
    }
    return dp[n - 1];
}
```

### Space-Optimized (Rolling Variables)
```java
public int solveOptimized(int[] nums) {
    int n = nums.length;
    if (n == 0) return 0;
    if (n == 1) return nums[0];

    int prev2 = nums[0];
    int prev1 = Math.max(nums[0], nums[1]);

    for (int i = 2; i < n; i++) {
        int curr = Math.max(prev1, prev2 + nums[i]);
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
```

### Top-Down (Memoization)
```java
public int solveMemo(int[] nums) {
    int[] memo = new int[nums.length];
    Arrays.fill(memo, -1);
    return helper(nums, nums.length - 1, memo);
}

private int helper(int[] nums, int i, int[] memo) {
    if (i < 0) return 0;
    if (i == 0) return nums[0];
    if (memo[i] != -1) return memo[i];

    memo[i] = Math.max(
        helper(nums, i - 1, memo),
        helper(nums, i - 2, memo) + nums[i]
    );
    return memo[i];
}
```

## Dry Run Example

**Problem:** House Robber — `nums = [2, 7, 9, 3, 1]`

State: `dp[i]` = maximum money robbing from houses `0..i` (cannot rob adjacent houses).

Recurrence: `dp[i] = max(dp[i-1], dp[i-2] + nums[i])`

| i | nums[i] | dp[i-2] + nums[i] | dp[i-1] | dp[i] = max(...) | Explanation |
|---|---------|-------------------|---------|-------------------|-------------|
| 0 | 2 | — | — | 2 | Base case: rob house 0 |
| 1 | 7 | — | 2 | max(2, 7) = 7 | Better to rob house 1 alone |
| 2 | 9 | 2 + 9 = 11 | 7 | max(7, 11) = 11 | Rob houses 0 and 2 |
| 3 | 3 | 7 + 3 = 10 | 11 | max(11, 10) = 11 | Skip house 3 |
| 4 | 1 | 11 + 1 = 12 | 11 | max(11, 12) = 12 | Rob houses 0, 2, and 4 |

**Answer:** `dp[4] = 12` (rob houses 0, 2, 4 → 2 + 9 + 1 = 12) ✓

## Complexity
- **Time:** O(n) for single-pass DP; O(n × k) when each state checks k transitions (e.g., Coin Change with k coin types)
- **Space:** O(n) for full DP array; O(1) when optimized to rolling variables

## Edge Cases
- `n = 0` — no elements to process; return 0 or appropriate default
- `n = 1` — only one element; the answer is trivially that element
- All elements are identical — ensure the recurrence still works correctly
- Negative values in the array — min/max logic must handle negative numbers
- Circular arrangement (House Robber II) — run linear DP twice: once excluding the first element, once excluding the last
- Very large `n` — ensure O(n) time and consider space optimization

## Common Mistakes
- **Forgetting base cases:** Not initializing `dp[0]` and `dp[1]` correctly leads to wrong answers.
- **Off-by-one in loop bounds:** Starting the loop at `i = 1` instead of `i = 2` when `dp[0]` and `dp[1]` are already set.
- **Not considering space optimization:** Using O(n) space when the recurrence only looks back 1–2 positions. Interviewers often ask for O(1) space follow-up.
- **Confusing "number of ways" with "min/max":** Ways problems use addition (`dp[i] = dp[i-1] + dp[i-2]`), optimization problems use min/max.
- **Returning `dp[n]` instead of `dp[n-1]`:** Off-by-one between 0-indexed arrays and 1-indexed formulations.
- **Jump Game: using DP when greedy suffices** — Jump Game I is greedy (track max reachable); Jump Game II needs DP or BFS.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Fibonacci Number | Easy | Pure `dp[i] = dp[i-1] + dp[i-2]`; the simplest DP to internalize the pattern | https://leetcode.com/problems/fibonacci-number/ |
| 2 | N-th Tribonacci Number | Easy | Extension to 3 terms: `dp[i] = dp[i-1] + dp[i-2] + dp[i-3]`; use 3 rolling vars | https://leetcode.com/problems/n-th-tribonacci-number/ |
| 3 | Climbing Stairs | Easy | Identical to Fibonacci — the number of ways to reach step `i` is sum of ways to `i-1` and `i-2` | https://leetcode.com/problems/climbing-stairs/ |
| 4 | Min Cost Climbing Stairs | Easy | `dp[i] = min(dp[i-1], dp[i-2]) + cost[i]`; can start from step 0 or 1 | https://leetcode.com/problems/min-cost-climbing-stairs/ |
| 5 | House Robber | Medium | Classic take/skip decision: `dp[i] = max(dp[i-1], dp[i-2] + nums[i])` | https://leetcode.com/problems/house-robber/ |
| 6 | House Robber II | Medium | Circular array — run House Robber twice: `nums[0..n-2]` and `nums[1..n-1]`, take the max | https://leetcode.com/problems/house-robber-ii/ |
| 7 | Decode Ways | Medium | At each position choose 1-digit or 2-digit decode; handle '0' carefully as it blocks single decoding | https://leetcode.com/problems/decode-ways/ |
| 8 | Coin Change | Medium | `dp[amount] = min(dp[amount - coin] + 1)` over all coins; unbounded knapsack flavour with 1D state | https://leetcode.com/problems/coin-change/ |
| 9 | Jump Game | Medium | Track the farthest reachable index; greedy works but DP also valid — `dp[i] = can reach i?` | https://leetcode.com/problems/jump-game/ |
| 10 | Jump Game II | Medium | BFS / greedy on levels; or DP `dp[i] = min jumps to reach i` with O(n²) transitions | https://leetcode.com/problems/jump-game-ii/ |
