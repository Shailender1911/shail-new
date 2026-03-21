# Pattern: Interval DP (Range DP)

## How to Identify This Pattern

### Trigger Words
- "burst", "remove", "merge" elements from a range
- "minimum/maximum cost to reduce/combine"
- "parenthesization", "matrix chain multiplication"
- "partition into", "split the array"
- "game theory", "optimal play", "stone game"
- "palindrome partitioning"
- "polygon triangulation"

### Constraint Clues
- Problem asks to optimally split or merge a **contiguous** range
- Input size n ≤ 500 (O(n³) is feasible) or n ≤ 100 (higher-degree polynomial okay)
- The answer for a range `[i, j]` depends on trying **every possible split point** `k` in `[i, j]`
- Elements can only be combined/removed from ends or by choosing a partition point
- The problem has "optimal substructure over intervals" — solving `[i, k]` and `[k+1, j]` independently

### When to Use This vs Similar Patterns
| Consideration | Interval DP | 1D DP | 2D Grid DP | Divide and Conquer |
|---|---|---|---|---|
| Subproblem structure | Range [i, j] | Prefix [0..i] | Cell (i, j) in grid | Independent halves |
| Key operation | Try all split points k in [i, j] | Look back 1-2 positions | Come from top/left | Fixed split (mid) |
| Time complexity | O(n³) | O(n) or O(n×k) | O(m×n) | O(n log n) |
| Typical n | ≤ 500 | ≤ 10⁵ | ≤ 1000 | ≤ 10⁵ |
| Overlap? | Yes, many shared subranges | Yes | Yes | No (disjoint halves) |

## Core Approach

1. **Define state:** `dp[i][j]` = the optimal answer (min cost, max score, etc.) for the subproblem on the range `[i, j]`.
2. **Write recurrence:** Try every split point `k` where `i ≤ k < j`:
   ```
   dp[i][j] = optimize over k: dp[i][k] + dp[k+1][j] + cost(i, k, j)
   ```
   where `cost(i, k, j)` is the cost of combining the two subranges.
3. **Base cases:** `dp[i][i] = base value` (single element — often 0 or the element itself).
4. **Build order:** Iterate by **increasing length** of the interval: `len = 1, 2, ..., n`. For each length, iterate all starting points `i`, compute `j = i + len - 1`.
5. **Answer location:** `dp[0][n-1]` (the entire range).

## Java Template

### Bottom-Up Interval DP
```java
public int intervalDP(int[] arr) {
    int n = arr.length;
    int[][] dp = new int[n][n];

    // len = 1: base cases (single elements)
    // dp[i][i] is already 0 or set to base value

    for (int len = 2; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            dp[i][j] = Integer.MAX_VALUE; // or Integer.MIN_VALUE for max
            for (int k = i; k < j; k++) {
                int cost = dp[i][k] + dp[k + 1][j] + mergeCost(arr, i, k, j);
                dp[i][j] = Math.min(dp[i][j], cost);
            }
        }
    }
    return dp[0][n - 1];
}
```

### Top-Down Interval DP (Memoization)
```java
int[][] memo;

public int solve(int[] arr) {
    int n = arr.length;
    memo = new int[n][n];
    for (int[] row : memo) Arrays.fill(row, -1);
    return helper(arr, 0, n - 1);
}

private int helper(int[] arr, int i, int j) {
    if (i == j) return 0;
    if (memo[i][j] != -1) return memo[i][j];

    memo[i][j] = Integer.MAX_VALUE;
    for (int k = i; k < j; k++) {
        int cost = helper(arr, i, k) + helper(arr, k + 1, j)
                   + mergeCost(arr, i, k, j);
        memo[i][j] = Math.min(memo[i][j], cost);
    }
    return memo[i][j];
}
```

### Burst Balloons (Classic Interval DP with virtual boundaries)
```java
public int maxCoins(int[] nums) {
    int n = nums.length;
    int[] arr = new int[n + 2];
    arr[0] = arr[n + 1] = 1;
    for (int i = 0; i < n; i++) arr[i + 1] = nums[i];

    int[][] dp = new int[n + 2][n + 2];

    for (int len = 1; len <= n; len++) {
        for (int i = 1; i <= n - len + 1; i++) {
            int j = i + len - 1;
            for (int k = i; k <= j; k++) {
                // k is the LAST balloon to burst in range [i, j]
                dp[i][j] = Math.max(dp[i][j],
                    dp[i][k - 1] + arr[i - 1] * arr[k] * arr[j + 1] + dp[k + 1][j]);
            }
        }
    }
    return dp[1][n];
}
```

## Dry Run Example

**Problem:** Burst Balloons — `nums = [3, 1, 5, 8]`

Pad with 1s: `arr = [1, 3, 1, 5, 8, 1]` (indices 0–5, balloons are at indices 1–4).

State: `dp[i][j]` = max coins from bursting all balloons in range `[i, j]`, where `k` is the **last** balloon burst.

**Length 1 (single balloons, each burst last):**

| Range | k | Coins = arr[i-1] × arr[k] × arr[j+1] | dp |
|---|---|---|---|
| [1,1] | 1 | arr[0]×arr[1]×arr[2] = 1×3×1 = 3 | 3 |
| [2,2] | 2 | arr[1]×arr[2]×arr[3] = 3×1×5 = 15 | 15 |
| [3,3] | 3 | arr[2]×arr[3]×arr[4] = 1×5×8 = 40 | 40 |
| [4,4] | 4 | arr[3]×arr[4]×arr[5] = 5×8×1 = 40 | 40 |

**Length 2:**

| Range | k=i | k=j | dp = max |
|---|---|---|---|
| [1,2] | dp[2,2] + 1×3×5 = 15+15=30 | dp[1,1] + 1×1×5 = 3+5=8 | 30 |
| [2,3] | dp[3,3] + 3×1×8 = 40+24=64 | dp[2,2] + 3×5×8 = 15+120=135 | 135 |
| [3,4] | dp[4,4] + 1×5×1 = 40+5=45 | dp[3,3] + 1×8×1 = 40+8=48 | 48 |

(Continuing this for length 3 and 4 gives the final answer.)

**Answer:** `dp[1][4] = 167` ✓

## Complexity
- **Time:** O(n³) — three nested loops (length, start, split point)
- **Space:** O(n²) — 2D DP table

## Edge Cases
- Single element → base case, return directly
- Two elements → only one split point, simple calculation
- All elements identical → still need to try all splits; symmetry doesn't always simplify
- n = 0 → return 0
- Burst Balloons: boundaries padded with 1s to avoid special-casing edges
- Game theory problems: alternating turns require careful "whose perspective" tracking
- Prefix sums often needed to compute `cost(i, j)` in O(1) instead of O(n)

## Common Mistakes
- **Wrong loop order:** Must iterate by increasing interval length, not by `i` then `j` independently.
- **Burst Balloons: treating `k` as the first balloon to burst** — The key insight is that `k` is the **last** balloon burst in range `[i, j]`, which makes the left and right subproblems independent.
- **Forgetting boundary padding:** Burst Balloons needs `arr[0] = arr[n+1] = 1` to handle edge multiplications.
- **Using wrong sentinel values:** For minimization, initialize to `Integer.MAX_VALUE`; for maximization, to `Integer.MIN_VALUE` or 0.
- **Game theory: forgetting the opponent plays optimally** — In Stone Game / Predict the Winner, the opponent minimizes your score.
- **Matrix Chain: confusing dimensions** — Matrix `i` has dimensions `p[i-1] × p[i]`; multiplication cost is `p[i-1] × p[k] × p[j]`.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Longest Palindromic Subsequence | Medium | `dp[i][j]` = LPS length in `s[i..j]`; if ends match, `dp[i+1][j-1] + 2`; else `max(dp[i+1][j], dp[i][j-1])` | https://leetcode.com/problems/longest-palindromic-subsequence/ |
| 2 | Predict the Winner | Medium | Game theory interval DP: player picks from ends; `dp[i][j]` = max advantage of the current player | https://leetcode.com/problems/predict-the-winner/ |
| 3 | Stone Game | Medium | Similar to Predict the Winner; Alice picks first from ends of piles; DP or math proof (Alice always wins) | https://leetcode.com/problems/stone-game/ |
| 4 | Minimum Cost Tree From Leaf Values | Medium | Build binary tree from leaves; `dp[i][j]` = min cost for leaves `[i..j]`; try all root splits; can also solve greedily | https://leetcode.com/problems/minimum-cost-tree-from-leaf-values/ |
| 5 | Burst Balloons | Hard | `k` = last balloon burst in `[i, j]`; pad array with 1s; coins = `arr[i-1] × arr[k] × arr[j+1]` | https://leetcode.com/problems/burst-balloons/ |
| 6 | Palindrome Partitioning II | Hard | Min cuts to make all parts palindromes; precompute palindrome table + 1D DP for cuts | https://leetcode.com/problems/palindrome-partitioning-ii/ |
| 7 | Strange Printer | Hard | `dp[i][j]` = min turns to print `s[i..j]`; if `s[i]==s[j]`, `dp[i][j-1]`; else try all splits | https://leetcode.com/problems/strange-printer/ |
| 8 | Minimum Score Triangulation of Polygon | Medium | `dp[i][j]` = min score triangulating vertices `[i..j]`; try all triangle tips k between i and j | https://leetcode.com/problems/minimum-score-triangulation-of-polygon/ |
| 9 | Matrix Chain Multiplication (GFG) | Medium | Classic interval DP: `dp[i][j]` = min multiplications for matrices `[i..j]`; try all split points | https://www.geeksforgeeks.org/problems/matrix-chain-multiplication0303/1 |
| 10 | Optimal BST (GFG) | Medium | Minimize search cost; `dp[i][j]` = min cost BST for keys `[i..j]`; try each key as root | https://www.geeksforgeeks.org/problems/optimal-binary-search-tree2214/1 |
