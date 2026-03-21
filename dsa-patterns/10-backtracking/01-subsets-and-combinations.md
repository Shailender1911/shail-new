# Pattern: Subsets & Combinations

## How to Identify This Pattern

### Trigger Words
- "all subsets", "power set", "every combination"
- "pick k elements", "choose n from m", "combinations of size k"
- "sum to target" with reuse or without reuse of elements
- "partition into groups" where order within the group does not matter (combination-style)
- "generate all valid strings" built by picking one option per step (digits → letters, parentheses pairs)

### Constraint Clues
- Count of answers is exponential (often \(2^n\) subsets or \(\binom{n}{k}\) combinations)
- Input size is small enough that brute-force enumeration is expected (typically \(n \le 20\) or similar)
- Duplicates in input require **skip-after-use** or **skip-equal-neighbors** logic to avoid duplicate combinations
- Problems distinguish **multiset with reuse** (Combination Sum) vs **use each index once** (Combination Sum II)

### When to Use This vs Similar Patterns
| Consideration | Subsets / Combinations | Permutations | DP |
|---|---|---|---|
| Order of picked elements | Does **not** matter (sort index / start from `start` to avoid permutations of the same multiset) | Order **matters** | Optimal substructure, overlapping subproblems |
| Duplicate handling | Sort + skip same value at same depth; or track frequency maps | Sort + skip used-at-position vs used-value carefully | Often tabulation instead of enumeration |
| Goal | Enumerate all valid **choices** up to size / sum | Enumerate all **orderings** | Count or optimize without listing all |

## Core Approach

Backtracking builds a **partial solution** (current list) and recurses on **remaining choices**.

1. **Choose:** Add a candidate to the partial solution (or branch on a mapping choice).
2. **Explore:** Recurse with updated state (next index, remaining sum, remaining picks).
3. **Unchoose:** Remove the candidate (restore state) so sibling branches see a clean slate.

**Combination / subset shape (avoid duplicate combinations):**
- Loop `i` from `start` to `n-1` so you only extend with elements **at or after** `start`.
- For duplicates: sort the array; skip `i > start && nums[i] == nums[i-1]` when `i` was not "continued" from the same depth (classic skip for Combination Sum II / Subsets II).

**Target-sum combinations:**
- **Reuse allowed:** recurse with same `i` (or any interpretation the problem specifies).
- **Each number once:** always recurse with `i + 1`.

## Java Template

```java
import java.util.*;

// Subsets: each element in or out (or loop start..end — equivalent with correct indexing)
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> res = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), res);
    return res;
}

private void backtrack(int[] nums, int start, List<Integer> path, List<List<Integer>> res) {
    res.add(new ArrayList<>(path)); // every node is a subset
    for (int i = start; i < nums.length; i++) {
        path.add(nums[i]);
        backtrack(nums, i + 1, path, res);
        path.remove(path.size() - 1);
    }
}

// Combinations: choose k numbers from 1..n
public List<List<Integer>> combine(int n, int k) {
    List<List<Integer>> res = new ArrayList<>();
    combineHelper(1, n, k, new ArrayList<>(), res);
    return res;
}

private void combineHelper(int start, int n, int k, List<Integer> path, List<List<Integer>> res) {
    if (path.size() == k) {
        res.add(new ArrayList<>(path));
        return;
    }
    for (int i = start; i <= n; i++) {
        path.add(i);
        combineHelper(i + 1, n, k, path, res);
        path.remove(path.size() - 1);
    }
}
```

## Dry Run Example

**Problem:** Subsets of `[1, 2]` (index-based combination tree).

```
                        []
                   /          \
              add 1            skip (loop continues)
                |                |
              [1]               (same level: i=1 adds 2 from [])
            /      \                |
      add 2    stop deeper     ... (omitted for space)
        |
      [1,2]

Recursion tree (conceptual):
dfs(start=0): path []
  i=0: take 1 → dfs(1): path [1]
    i=1: take 2 → dfs(2): path [1,2] → record
    i=1: after return, path [1]
  i=0: after return, path []
  i=1: take 2 → dfs(2): path [2] → record
Every time we enter dfs, we snapshot `path` into answers → [], [1], [1,2], [2].
```

## Complexity
- **Time:** Often \(O(k \cdot \text{number of answers})\) for building lists; subset enumeration is \(O(n \cdot 2^n)\) in typical implementations.
- **Space:** \(O(n)\) recursion depth plus \(O(n)\) for the current path (output space excluded).

## Edge Cases
- Empty input → answer is `[[]]` for subsets (one empty subset) unless the problem states otherwise.
- Duplicates in input → need explicit dedup strategy at each recursion depth.
- Target sum problems: distinguish reuse vs single-use; sort may be required for pruning or duplicate skipping.
- Integer overflow when summing — use `long` if constraints are tight.

## Common Mistakes
- **Accidentally generating permutations:** using `used[]` on values without a `start` index, or re-adding earlier indices.
- **Copying references:** adding `path` itself to results instead of `new ArrayList<>(path)`.
- **Off-by-one in combinations:** wrong bounds for `1..n` vs `0..n-1`.
- **Duplicate combinations:** forgetting to sort + skip equal neighbors (Subsets II / Combination Sum II).
- **Infinite recursion in reuse-allowed sum problems:** forgetting to pass the same index `i` when reuse is allowed.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Subsets | Medium | Record at every node; `for (i = start; i < n; i++)` pick loop | https://leetcode.com/problems/subsets/ |
| 2 | Subsets II | Medium | Sort; skip `nums[i] == nums[i-1]` when `i > start` | https://leetcode.com/problems/subsets-ii/ |
| 3 | Combination Sum | Medium | Reuse allowed → recurse with same `i`; prune when `sum > target` | https://leetcode.com/problems/combination-sum/ |
| 4 | Combination Sum II | Medium | Each candidate once → `i + 1`; sort + skip duplicates | https://leetcode.com/problems/combination-sum-ii/ |
| 5 | Combination Sum III | Medium | Fixed length `k` and range `1..9`; combination template with sum | https://leetcode.com/problems/combination-sum-iii/ |
| 6 | Letter Combinations of a Phone Number | Medium | DFS per digit; branch on 3–4 letters per key | https://leetcode.com/problems/letter-combinations-of-a-phone-number/ |
| 7 | Combinations | Medium | Choose `k` from `1..n` with `start` index loop | https://leetcode.com/problems/combinations/ |
| 8 | Generate Parentheses | Medium | Track `open`/`close`; only add `)` if `close < open` | https://leetcode.com/problems/generate-parentheses/ |
| 9 | Palindrome Partitioning | Medium | Extend prefix palindrome cuts; backtrack remainder | https://leetcode.com/problems/palindrome-partitioning/ |
| 10 | Restore IP Addresses | Medium | Build 4 segments from string; validate octet ranges (0–255, no bad leading zeros) | https://leetcode.com/problems/restore-ip-addresses/ |
