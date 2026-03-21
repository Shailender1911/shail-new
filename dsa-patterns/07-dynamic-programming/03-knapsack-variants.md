# Pattern: Knapsack Variants (0/1, Unbounded, Subset Sum)

## How to Identify This Pattern

### Trigger Words
- "subset", "partition into two groups"
- "can you achieve a target sum/weight/value"
- "number of ways to make amount"
- "select items with constraints"
- "knapsack", "capacity", "weight limit"
- "each item can be used once" (0/1) vs "unlimited times" (unbounded)
- "combination sum", "perfect squares"

### Constraint Clues
- Given a set of items each with a weight/value, and a capacity/target
- `n` items × `target` capacity → O(n × target) is feasible when both ≤ 1000–2000
- Boolean decision for each item: include or exclude (0/1 knapsack)
- Can reuse items (unbounded knapsack) — inner loop direction changes
- Target sum often equals `totalSum / 2` in partition problems

### When to Use This vs Similar Patterns
| Consideration | 0/1 Knapsack | Unbounded Knapsack | Greedy | Backtracking |
|---|---|---|---|---|
| Item usage | Each item used at most once | Each item used unlimited times | Fraction allowed | All combinations |
| Inner loop direction | Right to left (1D) | Left to right (1D) | N/A | Recursive |
| DP state | `dp[i][w]` or `dp[w]` | `dp[w]` | No DP needed | No memoization |
| When to use | Fixed items, binary choice | Coin change, rod cutting | Fractional knapsack | Small n, need all solutions |

## Core Approach

### 0/1 Knapsack
1. **Define state:** `dp[i][w]` = max value using items `0..i` with capacity `w`. Space-optimized: `dp[w]`.
2. **Write recurrence:** `dp[i][w] = max(dp[i-1][w], dp[i-1][w - weight[i]] + value[i])` if `w >= weight[i]`.
3. **Base cases:** `dp[0][w] = 0` for all `w` (no items → 0 value).
4. **Build order (1D):** For each item, iterate `w` from **capacity down to weight[i]** (right to left prevents reusing the same item).
5. **Answer location:** `dp[capacity]`.

### Unbounded Knapsack
1. **Define state:** `dp[w]` = max value / min coins to achieve weight/amount `w`.
2. **Write recurrence:** `dp[w] = min/max over all items: dp[w - weight[i]] + value[i]`.
3. **Base cases:** `dp[0] = 0`.
4. **Build order (1D):** For each amount `w`, iterate through all items (or iterate `w` **left to right** inside item loop).
5. **Answer location:** `dp[target]`.

### Subset Sum / Partition
1. **Define state:** `dp[s]` = boolean (or count): can we achieve sum `s` using a subset of items?
2. **Write recurrence:** `dp[s] = dp[s] || dp[s - nums[i]]`.
3. **Base cases:** `dp[0] = true`.
4. **Build order:** Right to left for 0/1 (each number used once).
5. **Answer location:** `dp[target]` where target = totalSum / 2 for equal partition.

## Java Template

### 0/1 Knapsack (Space-Optimized)
```java
public int knapsack01(int[] weights, int[] values, int capacity) {
    int[] dp = new int[capacity + 1];

    for (int i = 0; i < weights.length; i++) {
        for (int w = capacity; w >= weights[i]; w--) {
            dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
        }
    }
    return dp[capacity];
}
```

### Unbounded Knapsack (Coin Change — Minimum Coins)
```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);
    dp[0] = 0;

    for (int a = 1; a <= amount; a++) {
        for (int coin : coins) {
            if (coin <= a) {
                dp[a] = Math.min(dp[a], dp[a - coin] + 1);
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

### Subset Sum (Partition Equal Subset Sum)
```java
public boolean canPartition(int[] nums) {
    int total = 0;
    for (int num : nums) total += num;
    if (total % 2 != 0) return false;

    int target = total / 2;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;

    for (int num : nums) {
        for (int s = target; s >= num; s--) {
            dp[s] = dp[s] || dp[s - num];
        }
    }
    return dp[target];
}
```

### Counting Ways (Coin Change 2)
```java
public int change(int amount, int[] coins) {
    int[] dp = new int[amount + 1];
    dp[0] = 1;

    for (int coin : coins) {
        for (int a = coin; a <= amount; a++) {
            dp[a] += dp[a - coin];
        }
    }
    return dp[amount];
}
```

## Dry Run Example

**Problem:** Partition Equal Subset Sum — `nums = [1, 5, 11, 5]`

Total = 22, target = 11. Can we find a subset that sums to 11?

`dp[s]` = can we make sum `s`? Initialize: `dp[0] = true`.

| Process num | dp[0] | dp[1] | dp[5] | dp[6] | dp[10] | dp[11] | Notes |
|---|---|---|---|---|---|---|---|
| Init | T | F | F | F | F | F | |
| num=1 | T | **T** | F | F | F | F | s=1: dp[1]=dp[0]=T |
| num=5 | T | T | **T** | **T** | F | F | s=6: dp[1]=T→dp[6]=T; s=5: dp[0]=T→dp[5]=T |
| num=11 | T | T | T | T | F | **T** | s=11: dp[0]=T→dp[11]=T |

**Answer:** `dp[11] = true` ✓ (subset {11} sums to 11, other subset {1, 5, 5} also sums to 11)

## Complexity
- **Time:** O(n × target) where n = number of items, target = capacity/sum
- **Space:** O(target) with 1D optimization; O(n × target) with 2D table

## Edge Cases
- Total sum is odd in partition problems → immediately return false
- Target is 0 → always achievable (empty subset)
- Single item → can only partition if it equals target
- All items are the same → check if `count × value` can be halved
- Amount is 0 in coin change → return 0 coins (base case)
- No coins can make the target → return -1 (coin change) or 0 (count)
- Very large target with small items → ensure O(n × target) fits in time/memory

## Common Mistakes
- **Wrong loop direction:** 0/1 knapsack must iterate capacity **right to left** in 1D to avoid using the same item twice. Unbounded goes **left to right**.
- **Coin Change 2 vs Combination Sum IV:** Coin Change 2 counts combinations (order doesn't matter) — iterate coins in outer loop. Combination Sum IV counts permutations (order matters) — iterate amount in outer loop.
- **Forgetting the "impossible" sentinel:** In min-cost knapsack, initialize `dp[1..n]` to `Integer.MAX_VALUE` or `amount + 1`, not 0.
- **Partition with negative numbers:** Standard subset sum assumes non-negative. With negatives, shift indices or use a HashMap.
- **Target Sum (+ and -):** Transform to subset sum: find subset with sum `(total + target) / 2`. Check that `(total + target)` is even and non-negative.
- **Ones and Zeroes:** This is a 2D knapsack with two capacities (m zeros, n ones); need `dp[i][j]` with both iterated right to left.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Partition Equal Subset Sum | Medium | Subset sum where target = totalSum / 2; 0/1 knapsack with boolean dp | https://leetcode.com/problems/partition-equal-subset-sum/ |
| 2 | Target Sum | Medium | Transform to subset sum: find subset summing to (total + target) / 2; count ways | https://leetcode.com/problems/target-sum/ |
| 3 | Last Stone Weight II | Medium | Minimize |S1 - S2| → same as partition: find largest subset sum ≤ total/2 | https://leetcode.com/problems/last-stone-weight-ii/ |
| 4 | Coin Change | Medium | Unbounded knapsack to minimize coins; `dp[a] = min(dp[a-coin] + 1)` over all coins | https://leetcode.com/problems/coin-change/ |
| 5 | Coin Change 2 | Medium | Count combinations (not permutations): outer loop over coins, inner over amounts left to right | https://leetcode.com/problems/coin-change-ii/ |
| 6 | Ones and Zeroes | Medium | 2D 0/1 knapsack: two capacities (m zeros, n ones); iterate both right to left | https://leetcode.com/problems/ones-and-zeroes/ |
| 7 | Combination Sum IV | Medium | Count permutations: outer loop over target, inner over nums; left to right unbounded style | https://leetcode.com/problems/combination-sum-iv/ |
| 8 | Perfect Squares | Medium | Unbounded knapsack where "coins" are perfect squares 1, 4, 9, ...; minimize count | https://leetcode.com/problems/perfect-squares/ |
| 9 | Profitable Schemes | Hard | 3D DP: `dp[i][j]` = ways using at most i members to achieve at least j profit; iterate right to left for 0/1 | https://leetcode.com/problems/profitable-schemes/ |
| 10 | Knapsack Problem (GFG) | Medium | Classic 0/1 knapsack: maximize value under weight capacity; iterate weight right to left in 1D | https://www.geeksforgeeks.org/problems/0-1-knapsack-problem0945/1 |
