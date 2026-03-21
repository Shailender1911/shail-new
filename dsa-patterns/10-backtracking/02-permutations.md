# Pattern: Permutations

## How to Identify This Pattern

### Trigger Words
- "all permutations", "every ordering", "rearrange"
- "next lexicographic order", "k-th permutation"
- "arrange 1..n" with adjacency or divisibility rules
- "tiles" / "letters" — multiset permutations with pruning
- "superstring" or "scramble" — string structure built from permutations or splits (still permutation/search flavored)

### Constraint Clues
- Answer count grows like \(n!\) (or bounded multiset factorial)
- Need to **mark used elements** or **swap** within a range; order of inclusion matters
- Duplicates require skipping **identical choices at the same depth** (often sort + `!used[i-1]` when equal)

### When to Use This vs Similar Patterns
| Consideration | Permutations | Subsets / Combinations | Next Permutation (iterative) |
|---|---|---|---|
| Order | Every ordering matters | Order ignored (use `start`) | Single successor in sorted order |
| State | `used[]` or swap-based | `path` + `start` index | In-place pivot + reverse suffix |
| Duplicates | Skip same value at same tree level | Skip same value when not taking consecutive duplicate | N/A (handles multiset carefully) |

## Core Approach

**DFS with `used[]` (most common):**
1. When `path.size() == n`, record a copy of `path`.
2. For each index `i`, if not used, optionally skip duplicates (sorted input).
3. Mark `used[i]`, recurse, unmark.

**Swap-based (in-place permutations):** fix position `start`, swap `start` with `j >= start`, recurse `start+1`, swap back.

**Next Permutation (LeetCode 31):** find rightmost ascent, pivot, swap with rightmost larger on suffix, reverse suffix — \(O(n)\), no full enumeration.

## Java Template

```java
import java.util.*;

// Permutations with used[] (distinct elements)
public List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> res = new ArrayList<>();
    boolean[] used = new boolean[nums.length];
    dfs(nums, used, new ArrayList<>(), res);
    return res;
}

private void dfs(int[] nums, boolean[] used, List<Integer> path, List<List<Integer>> res) {
    if (path.size() == nums.length) {
        res.add(new ArrayList<>(path));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        // For duplicates (sorted): skip if same as prev and prev not used in this branch
        // if (i > 0 && nums[i] == nums[i-1] && !used[i-1]) continue;
        used[i] = true;
        path.add(nums[i]);
        dfs(nums, used, path, res);
        path.remove(path.size() - 1);
        used[i] = false;
    }
}
```

## Dry Run Example

**Problem:** `permute([1,2])` — recursion tree (choose position left-to-right).

```
                path []
        /-----------------------\
    pick 0 (1)              pick 1 (2)
       |                        |
    path [1]                 path [2]
    /      \                  /      \
 pick 1(2)  (none left)   pick 0(1)  ...
    |                        |
 path [1,2] ✓             path [2,1] ✓

Used bits:
dfs: {} -> try i=0 -> used {0} -> dfs -> try i=1 -> used {0,1} -> record [1,2]
backtrack -> try i=1 first at root -> used {1} -> ... -> record [2,1]
```

## Complexity
- **Time:** \(O(n \cdot n!)\) to emit all permutations for \(n\) distinct elements (each leaf + copy).
- **Space:** \(O(n)\) recursion + `used` + path; output excluded.

## Edge Cases
- `n == 0` or `n == 1` — still return the correct singleton/empty structure per problem statement.
- Duplicate values — must avoid duplicate permutations (sort + pruning rule).
- Next permutation on last permutation wraps to sorted ascending (problem definition).
- Large \(n\) for counting-only problems — use math / DP, not full enumeration.

## Common Mistakes
- **Set `used` incorrectly** with duplicates — wrong pruning duplicates valid branches or keeps invalid ones.
- **Shallow copy** of `path` into results.
- **Next permutation:** forgetting to reverse the suffix after swap, or wrong pivot when array is descending (full reverse).
- **Beautiful Arrangement / constrained permutations:** checking constraint at placement time vs only at leaves (should prune early).
- **K-th permutation:** trying to list all permutations for large \(n\) — use factorial number system / constructive approach.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Permutations | Medium | `used[]` DFS or swap-backtrack; snapshot at full length | https://leetcode.com/problems/permutations/ |
| 2 | Permutations II | Medium | Sort; skip `nums[i] == nums[i-1]` if previous equal not used | https://leetcode.com/problems/permutations-ii/ |
| 3 | Next Permutation | Medium | Rightmost ascent; swap with successor in suffix; reverse suffix | https://leetcode.com/problems/next-permutation/ |
| 4 | Beautiful Arrangement | Medium | Place from index 1..n; prune when value fails divisibility rule | https://leetcode.com/problems/beautiful-arrangement/ |
| 5 | Count Numbers with Unique Digits | Medium | Count permutations of digits for lengths 0..n (DP / combinatorics) | https://leetcode.com/problems/count-numbers-with-unique-digits/ |
| 6 | Letter Tile Possibilities | Medium | Frequency map; try each letter at depth; aggregate non-empty strings | https://leetcode.com/problems/letter-tile-possibilities/ |
| 7 | Permutation Sequence | Hard | Factorial numbering / constructive k-th permutation without full enum | https://leetcode.com/problems/permutation-sequence/ |
| 8 | Letter Case Permutation | Easy | At each char: branch lower vs upper when letter; DFS string build | https://leetcode.com/problems/letter-case-permutation/ |
| 9 | Scramble String | Hard | Binary tree split DP + memo; test if two strings are scramble-equivalent | https://leetcode.com/problems/scramble-string/ |
| 10 | Shortest Superstring | Hard | TSP-style bitmask DP over subsets (exact small \(n\)); not naive \(n!\) | https://leetcode.com/problems/shortest-superstring/ |
