# Pattern: Mirror, Symmetry & Inversion

## How to Identify This Pattern

### Trigger Words
- "symmetric", "mirror", "mirror image"
- "invert", "flip", "swap left and right"
- "same tree", "identical", "equal structure"
- "subtree of another tree", "check if one tree contains another"
- "merge two trees", "balanced", "complete"
- "leaf-similar", "univalued"

### Constraint Clues
- Comparing two trees (or two subtrees of the same tree) structurally
- Transforming a tree by swapping children
- Checking a structural property (balanced, complete, univalued) across all nodes
- Both trees must be traversed simultaneously in a coordinated manner

### When to Use This vs Similar Patterns
| Consideration | Dual-Tree Comparison | Single-Tree Transform | Structural Property Check |
|---|---|---|---|
| Input | Two roots / one root mirrored | One root | One root |
| Core technique | Parallel DFS on two nodes | DFS swapping children | DFS returning boolean / height |
| Example | Symmetric Tree, Same Tree | Invert Binary Tree | Balanced, Complete, Univalued |
| Key question | Do corresponding nodes match? | What transforms are needed? | Does every node satisfy the rule? |

## Core Approach

### Variant 1: Symmetric / Same Tree (Dual-Pointer DFS)
1. Compare two nodes simultaneously: `left` and `right` (or `tree1` and `tree2`).
2. Both null → match. One null → mismatch. Values differ → mismatch.
3. For symmetry: recurse with `(left.left, right.right)` and `(left.right, right.left)`.
4. For same tree: recurse with `(left.left, right.left)` and `(left.right, right.right)`.

### Variant 2: Invert Binary Tree
1. If node is null, return null.
2. Swap `node.left` and `node.right`.
3. Recurse on both children.
4. Return the node (now inverted).

### Variant 3: Balanced / Complete Check
1. **Balanced:** Postorder DFS returning height. If `|leftHeight - rightHeight| > 1` at any node, mark unbalanced. Use -1 as a sentinel for "unbalanced subtree."
2. **Complete:** BFS level order. Once a non-full node (missing a child) is seen, all subsequent nodes must be leaves.

## Java Template

### Symmetric Tree
```java
public boolean isSymmetric(TreeNode root) {
    if (root == null) return true;
    return isMirror(root.left, root.right);
}

private boolean isMirror(TreeNode t1, TreeNode t2) {
    if (t1 == null && t2 == null) return true;
    if (t1 == null || t2 == null) return false;
    return t1.val == t2.val
        && isMirror(t1.left, t2.right)
        && isMirror(t1.right, t2.left);
}
```

### Invert Binary Tree
```java
public TreeNode invertTree(TreeNode root) {
    if (root == null) return null;
    TreeNode left = invertTree(root.left);
    TreeNode right = invertTree(root.right);
    root.left = right;
    root.right = left;
    return root;
}
```

### Balanced Binary Tree
```java
public boolean isBalanced(TreeNode root) {
    return checkHeight(root) != -1;
}

private int checkHeight(TreeNode node) {
    if (node == null) return 0;
    int left = checkHeight(node.left);
    if (left == -1) return -1;
    int right = checkHeight(node.right);
    if (right == -1) return -1;
    if (Math.abs(left - right) > 1) return -1;
    return 1 + Math.max(left, right);
}
```

### Subtree of Another Tree
```java
public boolean isSubtree(TreeNode root, TreeNode subRoot) {
    if (root == null) return false;
    if (isSameTree(root, subRoot)) return true;
    return isSubtree(root.left, subRoot) || isSubtree(root.right, subRoot);
}

private boolean isSameTree(TreeNode p, TreeNode q) {
    if (p == null && q == null) return true;
    if (p == null || q == null) return false;
    return p.val == q.val
        && isSameTree(p.left, q.left)
        && isSameTree(p.right, q.right);
}
```

## Dry Run Example

**Problem:** Is Symmetric on tree:
```
        1
       / \
      2   2
     / \ / \
    3  4 4  3
```

| Compare | t1 | t2 | Match? | Next comparisons |
|---------|----|-----|--------|------------------|
| 1 | 2 (left of root) | 2 (right of root) | val==val ✓ | (2.left, 2.right) and (2.right, 2.left) |
| 2a | 3 (left.left) | 3 (right.right) | val==val ✓ | both children null → ✓ |
| 2b | 4 (left.right) | 4 (right.left) | val==val ✓ | both children null → ✓ |

**Result:** `true` ✓

**Problem:** Invert Binary Tree:
```
Before:          After:
    4              4
   / \            / \
  2   7          7   2
 / \ / \        / \ / \
1  3 6  9      9  6 3  1
```

| Node | Before swap | After swap |
|------|-------------|------------|
| 4 | left=2, right=7 | left=7, right=2 |
| 2 | left=1, right=3 | left=3, right=1 |
| 7 | left=6, right=9 | left=9, right=6 |

**Result:** All children swapped at every level ✓

## Complexity
- **Time:** O(n) for all variants — each node is visited once (Subtree check is O(m·n) in worst case where m and n are sizes of both trees)
- **Space:** O(h) for recursion stack where h is tree height

## Edge Cases
- Both trees are null → symmetric / same / balanced
- One tree is null and the other is not → not symmetric / not same
- Single-node tree → always symmetric, balanced, univalued
- Skewed tree — height difference = n, so not balanced
- All nodes have the same value — symmetric only if structure is also mirrored
- Subtree check — subRoot is larger than root → always false

## Common Mistakes
- **Symmetric: comparing `(left.left, right.left)` instead of `(left.left, right.right)`:** Symmetry mirrors, so outer nodes compare with outer, inner with inner.
- **Balanced: computing height separately then checking difference:** This is O(n²); use the -1 sentinel pattern for O(n).
- **Invert: only swapping at one level:** Must recurse into both subtrees after swapping.
- **Subtree check: comparing only values without structure:** Two trees with the same values but different shapes are not the same tree.
- **Same Tree: forgetting the base case when one is null:** Must check both-null before one-null to avoid NullPointerException.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Symmetric Tree | Easy | Mirror-compare: match (left.left, right.right) and (left.right, right.left) | https://leetcode.com/problems/symmetric-tree/ |
| 2 | Invert Binary Tree | Easy | Postorder swap: invert children first, then swap left and right | https://leetcode.com/problems/invert-binary-tree/ |
| 3 | Same Tree | Easy | Parallel DFS: both null → true, one null → false, compare vals and recurse | https://leetcode.com/problems/same-tree/ |
| 4 | Subtree of Another Tree | Easy | For each node in root, check isSameTree(node, subRoot); O(m·n) | https://leetcode.com/problems/subtree-of-another-tree/ |
| 5 | Flip Equivalent Binary Trees | Medium | At each pair, either children match directly or after flipping; recurse both ways | https://leetcode.com/problems/flip-equivalent-binary-trees/ |
| 6 | Check Completeness of a Binary Tree | Medium | BFS; once a null child is seen, no more non-null nodes should appear | https://leetcode.com/problems/check-completeness-of-a-binary-tree/ |
| 7 | Univalued Binary Tree | Easy | DFS: every node's value must equal the root's value | https://leetcode.com/problems/univalued-binary-tree/ |
| 8 | Merge Two Binary Trees | Easy | Parallel DFS: if both exist, sum values; if one is null, use the other | https://leetcode.com/problems/merge-two-binary-trees/ |
| 9 | Leaf-Similar Trees | Easy | Collect leaf sequences of both trees via DFS; compare the sequences | https://leetcode.com/problems/leaf-similar-trees/ |
| 10 | Balanced Binary Tree | Easy | Postorder height check: return -1 sentinel if |left - right| > 1 | https://leetcode.com/problems/balanced-binary-tree/ |
