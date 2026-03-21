# Pattern: 2D DP — Grids and Longest Common Subsequence

## How to Identify This Pattern

### Trigger Words
- "grid", "matrix", "paths from top-left to bottom-right"
- "longest common subsequence", "edit distance"
- "minimum path sum", "unique paths"
- "can only move right or down"
- "transform string A into string B"
- "maximal square/rectangle"
- "interleaving", "distinct subsequences"

### Constraint Clues
- Input is a 2D grid/matrix or two strings/sequences
- The answer at cell `(i, j)` depends on adjacent cells `(i-1, j)`, `(i, j-1)`, or `(i-1, j-1)`
- Constraints like `m, n <= 1000` suggest O(m×n) DP
- Two strings of length m and n need comparison → 2D table of size `(m+1) × (n+1)`
- Movement restricted to right/down in grid problems

### When to Use This vs Similar Patterns
| Consideration | 2D Grid DP | 2D Sequence DP (LCS/Edit) | 1D DP | BFS/DFS on Grid |
|---|---|---|---|---|
| Input type | Matrix of values | Two strings/arrays | Single array/value | Grid with obstacles |
| State | `dp[i][j]` = answer for cell (i,j) | `dp[i][j]` = answer for prefixes of length i, j | `dp[i]` = answer at index i | Visited set |
| Movement | Right/down (or all directions) | Diagonal + left/up in table | Left to right | All 4 directions |
| Goal | Path cost / count | Matching / alignment | Single-dimension optimization | Reachability / shortest path |

## Core Approach

### Grid DP
1. **Define state:** `dp[i][j]` = number of paths / minimum cost to reach cell `(i, j)`.
2. **Write recurrence:** `dp[i][j] = f(dp[i-1][j], dp[i][j-1])` (from top and from left).
3. **Base cases:** First row and first column — only one way to reach each cell (or cumulative cost).
4. **Build order:** Row by row, left to right.
5. **Answer location:** `dp[m-1][n-1]` (bottom-right corner).

### Sequence DP (LCS / Edit Distance)
1. **Define state:** `dp[i][j]` = LCS length of first `i` chars of `s1` and first `j` chars of `s2`.
2. **Write recurrence:** If `s1[i-1] == s2[j-1]`: `dp[i][j] = dp[i-1][j-1] + 1`. Else: `dp[i][j] = max(dp[i-1][j], dp[i][j-1])`.
3. **Base cases:** `dp[0][j] = 0` and `dp[i][0] = 0` (empty string has LCS 0).
4. **Build order:** Row by row, left to right.
5. **Answer location:** `dp[m][n]`.

## Java Template

### Grid DP — Unique Paths
```java
public int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];

    for (int i = 0; i < m; i++) dp[i][0] = 1;
    for (int j = 0; j < n; j++) dp[0][j] = 1;

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }
    return dp[m - 1][n - 1];
}
```

### Sequence DP — Longest Common Subsequence
```java
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[m][n];
}
```

### Maximal Square
```java
public int maximalSquare(char[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    int[][] dp = new int[m + 1][n + 1];
    int maxSide = 0;

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (matrix[i - 1][j - 1] == '1') {
                dp[i][j] = Math.min(dp[i - 1][j],
                            Math.min(dp[i][j - 1], dp[i - 1][j - 1])) + 1;
                maxSide = Math.max(maxSide, dp[i][j]);
            }
        }
    }
    return maxSide * maxSide;
}
```

## Dry Run Example

**Problem:** Minimum Path Sum — grid = `[[1, 3, 1], [1, 5, 1], [4, 2, 1]]`

State: `dp[i][j]` = minimum cost path from `(0,0)` to `(i,j)`.

Recurrence: `dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])`

Fill first row (can only come from left):

| | j=0 | j=1 | j=2 |
|---|---|---|---|
| i=0 | 1 | 1+3=4 | 4+1=5 |

Fill first column (can only come from above):

| | j=0 |
|---|---|
| i=0 | 1 |
| i=1 | 1+1=2 |
| i=2 | 2+4=6 |

Fill remaining cells:

| | j=0 | j=1 | j=2 |
|---|---|---|---|
| i=0 | 1 | 4 | 5 |
| i=1 | 2 | min(4,2)+5=**7** | min(5,7)+1=**6** |
| i=2 | 6 | min(7,6)+2=**8** | min(6,8)+1=**7** |

**Answer:** `dp[2][2] = 7` (path: 1 → 3 → 1 → 1 → 1) ✓

## Complexity
- **Time:** O(m × n) — fill every cell once
- **Space:** O(m × n) for full table; O(n) with row-by-row space optimization

## Edge Cases
- Grid is a single cell → return its value
- Grid has one row or one column → only one path, sum all values
- Grid contains obstacles (Unique Paths II) → set `dp[i][j] = 0` for obstacle cells
- Empty strings in LCS → return 0
- Identical strings → LCS = length of the string, Edit Distance = 0
- One string empty in Edit Distance → answer = length of the other string (all inserts)
- Grid with negative values (uncommon on LeetCode but possible) → min path sum logic still works

## Common Mistakes
- **Forgetting to handle obstacles:** In Unique Paths II, if `grid[i][j] == 1`, set `dp[i][j] = 0`, not skip it.
- **Wrong base case initialization:** First row/column in grid DP must accumulate, not all be 1 (for min path sum).
- **Off-by-one with string indexing:** `dp[i][j]` refers to first `i` characters, but `s.charAt(i-1)` for 0-indexed strings.
- **Not considering space optimization:** Most 2D DP needs only the previous row, reducing space from O(m×n) to O(n).
- **Maximal Square: confusing area with side length** — `dp[i][j]` stores the side length, square the max side for the area.
- **Triangle problem: starting from bottom is easier** — bottom-up avoids handling variable row lengths from the top.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Unique Paths | Medium | Pure counting DP: `dp[i][j] = dp[i-1][j] + dp[i][j-1]`; also solvable with combinatorics C(m+n-2, m-1) | https://leetcode.com/problems/unique-paths/ |
| 2 | Unique Paths II | Medium | Same as Unique Paths but set `dp[i][j] = 0` wherever there's an obstacle | https://leetcode.com/problems/unique-paths-ii/ |
| 3 | Minimum Path Sum | Medium | `dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])`; initialize first row/col cumulatively | https://leetcode.com/problems/minimum-path-sum/ |
| 4 | Triangle | Medium | Bottom-up DP starting from the last row avoids boundary checks; `dp[j] = triangle[i][j] + min(dp[j], dp[j+1])` | https://leetcode.com/problems/triangle/ |
| 5 | Longest Common Subsequence | Medium | Classic 2D sequence DP; match → diagonal + 1, mismatch → max(left, top) | https://leetcode.com/problems/longest-common-subsequence/ |
| 6 | Edit Distance | Medium | Three operations (insert, delete, replace) map to three transitions: left+1, top+1, diagonal+1 (or +0 if match) | https://leetcode.com/problems/edit-distance/ |
| 7 | Maximal Square | Medium | `dp[i][j] = min(top, left, diagonal) + 1` when cell is '1'; answer is max side squared | https://leetcode.com/problems/maximal-square/ |
| 8 | Interleaving String | Medium | `dp[i][j]` = can s3[0..i+j-1] be formed by interleaving s1[0..i-1] and s2[0..j-1]; check match from either string | https://leetcode.com/problems/interleaving-string/ |
| 9 | Distinct Subsequences | Hard | Count how many times `t` appears as a subsequence of `s`; match → `dp[i-1][j-1] + dp[i-1][j]`, else → `dp[i-1][j]` | https://leetcode.com/problems/distinct-subsequences/ |
| 10 | Dungeon Game | Hard | Must DP **backwards** from bottom-right; `dp[i][j]` = min HP needed to enter cell (i,j) and survive to the end | https://leetcode.com/problems/dungeon-game/ |
