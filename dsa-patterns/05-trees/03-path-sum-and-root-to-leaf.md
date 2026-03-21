# Pattern: Path Sum & Root-to-Leaf Problems

## How to Identify This Pattern

### Trigger Words
- "path sum", "root to leaf", "root-to-leaf path"
- "maximum path sum", "any path", "downward path"
- "sum of all paths", "print all paths"
- "diameter", "longest path", "univalue path"
- "good nodes", "pseudo-palindromic"

### Constraint Clues
- Need to accumulate a value (sum, product, string) along paths
- A "path" is defined as root-to-leaf, any node-to-node, or downward-only
- Result depends on aggregating information from both subtrees at each node
- Need to track running state (current sum, current path) during DFS

### When to Use This vs Similar Patterns
| Consideration | Root-to-Leaf Path | Any-Node Path | Subtree Aggregate |
|---|---|---|---|
| Path constraint | Must start at root, end at leaf | Can start/end anywhere | Entire subtree rooted at each node |
| DFS direction | Top-down with backtracking | Bottom-up returning max gain | Bottom-up returning subtree value |
| State passing | Pass cumulative sum/path down | Return single-branch gain up | Return computed value up |
| Example | Path Sum, Binary Tree Paths | Max Path Sum, Diameter | Count Good Nodes |

## Core Approach

### Variant 1: Root-to-Leaf with Target Sum
1. DFS from root, subtracting current node's value from the remaining target.
2. At a leaf, check if remaining target equals the leaf's value.
3. If collecting all paths, maintain a list and backtrack after each recursive call.

### Variant 2: Maximum Path Sum (any node to any node)
1. At each node, compute the maximum gain from the left subtree and right subtree (clamp negative gains to 0).
2. The path through the current node = node.val + leftGain + rightGain — update the global max.
3. Return node.val + max(leftGain, rightGain) upward (a path can only fork once).

### Variant 3: Diameter / Longest Path
1. Postorder DFS returning the height of each subtree.
2. At each node, the path passing through it = leftHeight + rightHeight.
3. Update a global maximum with this path length.
4. Return 1 + max(leftHeight, rightHeight) upward.

## Java Template

### Path Sum (Root-to-Leaf)
```java
public boolean hasPathSum(TreeNode root, int targetSum) {
    if (root == null) return false;
    if (root.left == null && root.right == null) {
        return root.val == targetSum;
    }
    return hasPathSum(root.left, targetSum - root.val)
        || hasPathSum(root.right, targetSum - root.val);
}
```

### Path Sum II (Collect All Paths)
```java
public List<List<Integer>> pathSum(TreeNode root, int targetSum) {
    List<List<Integer>> result = new ArrayList<>();
    dfs(root, targetSum, new ArrayList<>(), result);
    return result;
}

private void dfs(TreeNode node, int remaining, List<Integer> path,
                 List<List<Integer>> result) {
    if (node == null) return;
    path.add(node.val);
    if (node.left == null && node.right == null && remaining == node.val) {
        result.add(new ArrayList<>(path));
    } else {
        dfs(node.left, remaining - node.val, path, result);
        dfs(node.right, remaining - node.val, path, result);
    }
    path.remove(path.size() - 1);
}
```

### Maximum Path Sum
```java
int maxSum = Integer.MIN_VALUE;

public int maxPathSum(TreeNode root) {
    maxGain(root);
    return maxSum;
}

private int maxGain(TreeNode node) {
    if (node == null) return 0;
    int leftGain = Math.max(maxGain(node.left), 0);
    int rightGain = Math.max(maxGain(node.right), 0);
    maxSum = Math.max(maxSum, node.val + leftGain + rightGain);
    return node.val + Math.max(leftGain, rightGain);
}
```

### Diameter of Binary Tree
```java
int diameter = 0;

public int diameterOfBinaryTree(TreeNode root) {
    height(root);
    return diameter;
}

private int height(TreeNode node) {
    if (node == null) return 0;
    int left = height(node.left);
    int right = height(node.right);
    diameter = Math.max(diameter, left + right);
    return 1 + Math.max(left, right);
}
```

## Dry Run Example

**Problem:** Path Sum — target = 22 on tree:
```
        5
       / \
      4   8
     /   / \
    11  13   4
   / \        \
  7   2        1
```

| Path | Running Sum | Leaf? | Sum == 22? |
|------|-------------|-------|------------|
| 5 → 4 → 11 → 7 | 5+4+11+7 = 27 | Yes | No |
| 5 → 4 → 11 → 2 | 5+4+11+2 = 22 | Yes | **Yes ✓** |
| 5 → 8 → 13 | 5+8+13 = 26 | Yes | No |
| 5 → 8 → 4 → 1 | 5+8+4+1 = 18 | Yes | No |

**Result:** `true` ✓

**Problem:** Diameter on tree:
```
        1
       / \
      2   3
     / \
    4   5
```

| Node | Left Height | Right Height | Path Through Node | Max Diameter |
|------|-------------|--------------|-------------------|--------------|
| 4 | 0 | 0 | 0 | 0 |
| 5 | 0 | 0 | 0 | 0 |
| 2 | 1 (via 4) | 1 (via 5) | 2 | 2 |
| 3 | 0 | 0 | 0 | 2 |
| 1 | 2 (via 2) | 1 (via 3) | 3 | **3** |

**Result:** Diameter = `3` (path 4→2→1→3) ✓

## Complexity
- **Time:** O(n) — each node is visited once in all variants
- **Space:** O(h) for recursion stack; O(n) worst case for skewed tree; Path Sum II also uses O(n) for storing paths

## Edge Cases
- Empty tree → return false / 0 / empty list
- Single node — it is both root and leaf; check if its value matches the target
- All negative values — max path sum must handle negative nodes; don't assume paths have positive sums
- Tree with only one child at each level — there is only one root-to-leaf path
- Target sum of 0 with node values of 0 — ensure leaf check is correct

## Common Mistakes
- **Not checking the leaf condition:** `root.left == null && root.right == null` — a node with one child is NOT a leaf; returning true there gives wrong paths.
- **Forgetting to backtrack:** In Path Sum II, if you don't `path.remove(path.size() - 1)`, paths accumulate incorrectly.
- **Max Path Sum — not clamping negative gains to 0:** A subtree with negative gain should not be included; use `Math.max(gain, 0)`.
- **Max Path Sum — returning the fork value instead of single branch:** The return value must be `node.val + max(left, right)`, not `node.val + left + right`, because a path cannot fork twice.
- **Diameter — confusing edge count vs node count:** Diameter is the number of edges, not nodes. `left + right` already gives edge count.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Path Sum | Easy | DFS subtracting node values; check remainder == 0 at leaf | https://leetcode.com/problems/path-sum/ |
| 2 | Path Sum II | Medium | DFS with backtracking; collect path when leaf sum matches | https://leetcode.com/problems/path-sum-ii/ |
| 3 | Path Sum III | Medium | Prefix-sum HashMap counts paths with target sum on any downward path | https://leetcode.com/problems/path-sum-iii/ |
| 4 | Binary Tree Maximum Path Sum | Hard | At each node: update global max with left+node+right; return single-branch max upward | https://leetcode.com/problems/binary-tree-maximum-path-sum/ |
| 5 | Sum Root to Leaf Numbers | Medium | DFS carrying currentNum = currentNum * 10 + node.val; sum at leaves | https://leetcode.com/problems/sum-root-to-leaf-numbers/ |
| 6 | Binary Tree Paths | Easy | DFS collecting path strings; append "→" between nodes; add to result at leaves | https://leetcode.com/problems/binary-tree-paths/ |
| 7 | Diameter of Binary Tree | Easy | Postorder returning height; diameter at node = leftHeight + rightHeight | https://leetcode.com/problems/diameter-of-binary-tree/ |
| 8 | Longest Univalue Path | Medium | Like diameter but only extend when child.val == parent.val; reset otherwise | https://leetcode.com/problems/longest-univalue-path/ |
| 9 | Count Good Nodes in Binary Tree | Medium | DFS passing maxSoFar; node is "good" if node.val >= maxSoFar | https://leetcode.com/problems/count-good-nodes-in-binary-tree/ |
| 10 | Pseudo-Palindromic Paths in a Binary Tree | Medium | Track digit frequency with bitmask XOR; at leaf, at most one bit set means palindrome | https://leetcode.com/problems/pseudo-palindromic-paths-in-a-binary-tree/ |
