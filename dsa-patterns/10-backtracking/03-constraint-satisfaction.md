# Pattern: Constraint Satisfaction (Boards, Grids, Partitioning)

## How to Identify This Pattern

### Trigger Words
- "place queens", "N-Queens", "no two attack"
- "solve Sudoku", "fill the board"
- "exist path in grid", "word search on board"
- "partition array into k equal-sum subsets", "form a square from sticks"
- "split string into …" with numeric or ordering constraints

### Constraint Clues
- Multiple dimensions of choice (row/column/cell) with **local** and **global** validity checks
- Search space is huge, but each decision is **reversible** (try / backtrack)
- Often needs **pruning**: early invalid partial assignments
- Word Search II benefits from **Trie** + DFS to share prefixes (still backtracking on board)

### When to Use This vs Similar Patterns
| Consideration | Backtracking CSP | BFS / Graph Search | DP |
|---|---|---|---|
| State | Partial assignment on grid/board | Layer-by-layer shortest path | Optimal substructure |
| Revisit cells | Depends (Unique Paths III visits each once) | Mark visited in graph | Table of subproblems |
| Structure | Try candidate → validate → recurse → undo | Queue of frontier states | Fill memo/table |

## Core Approach

1. **Represent state:** board, visited mask, remaining counts, current index in input.
2. **Iterate choices:** valid next queen column, next empty Sudoku cell candidates, next grid step, next subset assignment.
3. **Validate constraint** before or immediately after placing.
4. **Recurse**; on failure, **undo** the placement and try the next choice.

**Practical tips:**
- Pick the **most constrained** variable first when obvious (e.g., Sudoku cell with few candidates).
- For equal-sum partitions, **sort descending** often prunes faster (larger sticks first).

## Java Template

```java
// Generic grid DFS with backtracking (e.g., Word Search style)
public boolean dfs(char[][] board, boolean[][] vis, int r, int c, String word, int k) {
    if (k == word.length()) return true;
    if (r < 0 || r >= board.length || c < 0 || c >= board[0].length) return false;
    if (vis[r][c] || board[r][c] != word.charAt(k)) return false;

    vis[r][c] = true;
    int[][] dirs = {{1,0},{-1,0},{0,1},{0,-1}};
    for (int[] d : dirs) {
        if (dfs(board, vis, r + d[0], c + d[1], word, k + 1)) return true;
    }
    vis[r][c] = false; // unchoose
    return false;
}

// N-Queens: place row-by-row; cols / diagonals as constraint sets
// Sudoku: find next '.'; try '1'..'9'; validate row/col/box; recurse; undo
```

## Dry Run Example

**Problem:** N-Queens conceptual trace for `n = 4` (row-by-column placement).

```
Row 0: try col 0 → place Q at (0,0)
  Row 1: col 0 diag clash, col 1 clash, try col 2 → (1,2)
    Row 2: try cols ... find valid
    ...
  If dead end → remove (1,2), try next col for row 1
If all fail → remove (0,0), try col 1 for row 0

Each node = partial placement; children = valid columns for next row.
Leaf = full placement → record board snapshot.
```

## Complexity
- **N-Queens:** Roughly exponential in \(n\) with strong pruning; exact bound not needed for interviews—emphasize pruning.
- **Sudoku:** Worst-case exponential; in practice constraints collapse search.
- **Word Search:** \(O(m \cdot n \cdot 4^L)\) per word without Trie optimizations (plus board size factors).
- **Partition / stick problems:** exponential in number of sticks; bitmasks / sorting help pruning.

## Edge Cases
- Empty board or trivial `n` (e.g., `n == 1` for queens).
- Words longer than cell count; impossible paths.
- Sudoku: already invalid input — return early per spec.
- Numeric string splitting: leading zeros, overflow, single-digit segments.
- Word Search II: overlapping paths — revisit allowed only if you unmark on backtrack (classic).

## Common Mistakes
- **Not undoing** visited flags or board mutations when returning false.
- **Confusing Word Search I vs II:** II needs efficient prefix checks (Trie) to avoid timeouts.
- **N-Queens II:** confusing with counting — same search, no board materialization.
- **Partition to K Equal Sum Subsets:** naive bitmask assignment without pruning — TLE on typical constraints.
- **Unique Paths III:** must visit **every** free cell exactly once — different from standard "any path" DFS.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | N-Queens | Hard | Place by row; track columns and two diagonal directions | https://leetcode.com/problems/n-queens/ |
| 2 | N-Queens II | Hard | Same search as N-Queens but count solutions only | https://leetcode.com/problems/n-queens-ii/ |
| 3 | Sudoku Solver | Hard | Pick empty cell; try digits; validate 3×3 + row/col; backtrack | https://leetcode.com/problems/sudoku-solver/ |
| 4 | Word Search | Medium | DFS + visited toggle; prune on first char mismatch | https://leetcode.com/problems/word-search/ |
| 5 | Word Search II | Hard | Trie of words; DFS from each cell; remove words to prune branches | https://leetcode.com/problems/word-search-ii/ |
| 6 | Unique Paths III | Hard | DFS over empty squares; count Hamiltonian paths covering all non-obstacle cells | https://leetcode.com/problems/unique-paths-iii/ |
| 7 | Matchsticks to Square | Medium | Backtrack assign each stick to one of 4 sides; equal target length | https://leetcode.com/problems/matchsticks-to-square/ |
| 8 | Partition to K Equal Sum Subsets | Medium | Multiset partition with bitmask / used array; sort desc for pruning | https://leetcode.com/problems/partition-to-k-equal-sum-subsets/ |
| 9 | Additive Number | Medium | Try first two numbers then verify remaining string extends sum chain | https://leetcode.com/problems/additive-number/ |
| 10 | Splitting a String Into Descending Consecutive Values | Medium | Backtrack splits; enforce strictly decreasing consecutive values | https://leetcode.com/problems/splitting-a-string-into-descending-consecutive-values/ |
