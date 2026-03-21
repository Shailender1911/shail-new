# Pattern: Lowest Common Ancestor (LCA)

## How to Identify This Pattern

### Trigger Words
- "lowest common ancestor", "LCA"
- "nearest common parent", "first shared ancestor"
- "distance between two nodes"
- "all nodes at distance K"
- "deepest leaves", "smallest subtree containing all deepest"
- "ancestor", "descendant relationship"

### Constraint Clues
- Two target nodes are given and you must find where their paths from root diverge
- Need to compute distance between two nodes (distance = depth(p) + depth(q) − 2·depth(LCA))
- Problem involves relationships between nodes at different depths
- Need to find a node that is an ancestor of a specific set of nodes

### When to Use This vs Similar Patterns
| Consideration | LCA in BST | LCA in Binary Tree | LCA of Deepest | Distance / K-away |
|---|---|---|---|---|
| Tree type | BST | General | General | General |
| Key insight | Compare values to go left/right | Postorder null-propagation | Depth-aware DFS | LCA + BFS from target |
| Complexity | O(h) | O(n) | O(n) | O(n) |

## Core Approach

### Variant 1: LCA in BST
1. Start at root. If both `p` and `q` are smaller than root, go left.
2. If both are larger, go right.
3. Otherwise, current root is the split point — it is the LCA.

### Variant 2: LCA in General Binary Tree
1. Postorder DFS: if the current node is null, `p`, or `q`, return it.
2. Recurse left and right.
3. If both left and right return non-null, current node is the LCA.
4. If only one side returns non-null, propagate that result upward.

### Variant 3: All Nodes at Distance K
1. Find the target node and build a parent-pointer map (or convert tree to graph).
2. BFS from the target node, expanding to left, right, and parent.
3. Track visited nodes to avoid revisiting. Collect all nodes at distance K.

### Variant 4: LCA of Deepest Leaves
1. DFS returning both depth and LCA candidate.
2. At each node, get (depth, LCA) from left and right subtrees.
3. If left depth > right depth, propagate left result; if right > left, propagate right.
4. If depths are equal, current node is the LCA of the deepest leaves at this subtree.

## Java Template

### LCA of BST
```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    while (root != null) {
        if (p.val < root.val && q.val < root.val) {
            root = root.left;
        } else if (p.val > root.val && q.val > root.val) {
            root = root.right;
        } else {
            return root;
        }
    }
    return null;
}
```

### LCA of Binary Tree
```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    if (left != null && right != null) return root;
    return left != null ? left : right;
}
```

### All Nodes Distance K
```java
public List<Integer> distanceK(TreeNode root, TreeNode target, int k) {
    Map<TreeNode, TreeNode> parentMap = new HashMap<>();
    buildParentMap(root, null, parentMap);

    Queue<TreeNode> queue = new LinkedList<>();
    Set<TreeNode> visited = new HashSet<>();
    queue.offer(target);
    visited.add(target);

    int distance = 0;
    while (!queue.isEmpty()) {
        if (distance == k) {
            List<Integer> result = new ArrayList<>();
            for (TreeNode node : queue) result.add(node.val);
            return result;
        }
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            for (TreeNode neighbor : new TreeNode[]{node.left, node.right, parentMap.get(node)}) {
                if (neighbor != null && !visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.offer(neighbor);
                }
            }
        }
        distance++;
    }
    return new ArrayList<>();
}

private void buildParentMap(TreeNode node, TreeNode parent, Map<TreeNode, TreeNode> map) {
    if (node == null) return;
    map.put(node, parent);
    buildParentMap(node.left, node, map);
    buildParentMap(node.right, node, map);
}
```

### LCA of Deepest Leaves
```java
public TreeNode lcaDeepestLeaves(TreeNode root) {
    return dfs(root).node;
}

private Result dfs(TreeNode node) {
    if (node == null) return new Result(null, 0);
    Result left = dfs(node.left);
    Result right = dfs(node.right);
    if (left.depth > right.depth) return new Result(left.node, left.depth + 1);
    if (right.depth > left.depth) return new Result(right.node, right.depth + 1);
    return new Result(node, left.depth + 1);
}

record Result(TreeNode node, int depth) {}
```

## Dry Run Example

**Problem:** LCA of Binary Tree — find LCA of nodes 5 and 1:
```
        3
       / \
      5   1
     / \ / \
    6  2 0  8
      / \
     7   4
```

| Call | Node | Left result | Right result | Return |
|------|------|-------------|--------------|--------|
| dfs(6) | 6 | null | null | null |
| dfs(7) | 7 | null | null | null |
| dfs(4) | 4 | null | null | null |
| dfs(2) | 2 | null (7≠p,q) | null (4≠p,q) | null |
| dfs(5) | 5 | **5==p → return 5** | | |
| Actually: root==p | 5 | — | — | return 5 |
| dfs(0) | 0 | null | null | null |
| dfs(8) | 8 | null | null | null |
| dfs(1) | 1 | **1==q → return 1** | | |
| dfs(3) | 3 | left=5 (non-null) | right=1 (non-null) | **return 3** |

**Result:** LCA(5, 1) = `3` ✓

**Problem:** LCA of BST — find LCA of 2 and 8:
```
        6
       / \
      2   8
     / \ / \
    0  4 7  9
      / \
     3   5
```

| Step | Root | p=2, q=8 | Action |
|------|------|----------|--------|
| 1 | 6 | 2 < 6 and 8 > 6 | Split point → return 6 |

**Result:** LCA(2, 8) = `6` ✓ (found in O(1) since root is the split)

## Complexity
- **Time:** O(h) for BST LCA; O(n) for general binary tree LCA and distance K
- **Space:** O(h) for recursion stack; O(n) for parent map in distance K problems

## Edge Cases
- One of the target nodes is the root → the root is the LCA
- One node is an ancestor of the other → the ancestor node is the LCA
- Both nodes are the same → the node itself is the LCA
- Nodes are in different subtrees → LCA is the first common split point
- Single-node tree with both targets equal to root → return root
- Target node not in tree — standard LCA assumes both nodes exist; if not guaranteed, do a two-pass check

## Common Mistakes
- **Binary Tree LCA — assuming BST ordering:** The general LCA algorithm must check identity (`root == p`), not value comparison.
- **BST LCA — using `<=` instead of `<`:** If `p.val == root.val`, root is the LCA; don't continue searching.
- **Distance K — forgetting parent pointers:** A tree only has child pointers; you must build a parent map or convert to a graph to go "upward."
- **Distance K — not tracking visited nodes:** Without a visited set, BFS will revisit parent→child→parent in a loop.
- **LCA of Deepest Leaves — confusing depth counting:** Be consistent about whether depth is 0-indexed or 1-indexed; mismatches cause wrong LCA selection.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Lowest Common Ancestor of a Binary Search Tree | Medium | BST property: both smaller → go left, both larger → go right, else split point | https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/ |
| 2 | Lowest Common Ancestor of a Binary Tree | Medium | Postorder: if left and right both non-null, current node is LCA | https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/ |
| 3 | Lowest Common Ancestor of Deepest Leaves | Medium | Return (depth, LCA) pairs; equal depths at a node means that node is the LCA | https://leetcode.com/problems/lowest-common-ancestor-of-deepest-leaves/ |
| 4 | All Nodes Distance K in Binary Tree | Medium | Build parent map, then BFS from target node expanding in 3 directions | https://leetcode.com/problems/all-nodes-distance-k-in-binary-tree/ |
| 5 | Find Distance in a Binary Tree | Medium | Distance = depth(p) + depth(q) − 2 × depth(LCA) | https://leetcode.com/problems/find-distance-in-a-binary-tree/ |
| 6 | Maximum Difference Between Node and Ancestor | Medium | DFS passing current path min and max; diff = max(|node - min|, |node - max|) | https://leetcode.com/problems/maximum-difference-between-node-and-ancestor/ |
| 7 | Count Nodes Equal to Average of Subtree | Medium | Postorder returning (sum, count); check if node.val == sum/count | https://leetcode.com/problems/count-nodes-equal-to-average-of-subtree/ |
| 8 | Smallest Subtree with all the Deepest Nodes | Medium | Identical to LCA of deepest leaves; track depth from both sides | https://leetcode.com/problems/smallest-subtree-with-all-the-deepest-nodes/ |
| 9 | Binary Tree Cameras | Hard | Greedy postorder: leaves should not have cameras; their parents should; track 3 states | https://leetcode.com/problems/binary-tree-cameras/ |
| 10 | Step-By-Step Directions From a Binary Tree Node to Another | Medium | Find LCA, build paths LCA→p and LCA→q; replace LCA→p path with "U"s | https://leetcode.com/problems/step-by-step-directions-from-a-binary-tree-node-to-another/ |
