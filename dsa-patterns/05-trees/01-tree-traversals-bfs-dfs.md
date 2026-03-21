# Pattern: Tree Traversals — BFS & DFS

## How to Identify This Pattern

### Trigger Words
- "level order", "breadth-first", "layer by layer"
- "preorder", "inorder", "postorder", "depth-first"
- "print each level", "zigzag", "right side view"
- "visit every node", "traverse the tree"
- "average of levels", "depth of tree"

### Constraint Clues
- Need to process nodes level by level → BFS
- Need to process nodes in sorted order (BST) → Inorder DFS
- Need to build/serialize a tree → Preorder DFS
- Need to compute bottom-up aggregates (height, size) → Postorder DFS
- "Return the result in level order" or "group by depth"

### When to Use This vs Similar Patterns
| Consideration | BFS (Level Order) | DFS (Pre/In/Post) | Path-Based DFS |
|---|---|---|---|
| Traversal order | Layer by layer | Root→subtree depth | Root to leaf |
| Data structure | Queue | Recursion / Stack | Recursion + backtracking |
| Best for | Level grouping, shortest depth | Serialization, sorted order | Path sums, path enumeration |
| Space | O(w) where w = max width | O(h) where h = height | O(h) |

## Core Approach

### Variant 1: BFS — Level Order Traversal
1. Create a queue and add the root node.
2. While the queue is not empty, record its current size `levelSize`.
3. Process exactly `levelSize` nodes — these are all nodes on the current level.
4. For each node, add its non-null children to the queue.
5. Collect results per level into a list of lists.

### Variant 2: DFS — Recursive
1. **Preorder (Root → Left → Right):** Process current node, recurse left, recurse right.
2. **Inorder (Left → Root → Right):** Recurse left, process current node, recurse right.
3. **Postorder (Left → Right → Root):** Recurse left, recurse right, process current node.

### Variant 3: DFS — Iterative with Explicit Stack
1. Push the root onto a stack.
2. Pop a node, process it (preorder) or defer processing (inorder/postorder).
3. Push right child first, then left child (so left is processed first for preorder).
4. For inorder: push all left children, pop and process, then move to right.

## Java Template

### BFS — Level Order
```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        result.add(level);
    }
    return result;
}
```

### DFS — Recursive Inorder
```java
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    dfs(root, result);
    return result;
}

private void dfs(TreeNode node, List<Integer> result) {
    if (node == null) return;
    dfs(node.left, result);
    result.add(node.val);
    dfs(node.right, result);
}
```

### DFS — Iterative Inorder
```java
public List<Integer> inorderIterative(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode curr = root;

    while (curr != null || !stack.isEmpty()) {
        while (curr != null) {
            stack.push(curr);
            curr = curr.left;
        }
        curr = stack.pop();
        result.add(curr.val);
        curr = curr.right;
    }
    return result;
}
```

## Dry Run Example

**Problem:** Level Order Traversal on tree:
```
        3
       / \
      9   20
         /  \
        15   7
```

| Step | Queue (front→back) | Level | Action |
|------|--------------------|-------|--------|
| Init | [3] | — | Start |
| 1 | [9, 20] | [3] | Poll 3, enqueue 9 and 20 |
| 2 | [15, 7] | [9, 20] | Poll 9 (no children), poll 20 (enqueue 15, 7) |
| 3 | [] | [15, 7] | Poll 15, poll 7 (no children) |

**Result:** `[[3], [9, 20], [15, 7]]` ✓

**Problem:** Inorder DFS on the same tree:

| Call | Node | Action | Result so far |
|------|------|--------|---------------|
| dfs(3) | 3 | Go left → dfs(9) | — |
| dfs(9) | 9 | Go left → null, add 9, go right → null | [9] |
| back to dfs(3) | 3 | Add 3, go right → dfs(20) | [9, 3] |
| dfs(20) | 20 | Go left → dfs(15) | — |
| dfs(15) | 15 | Go left → null, add 15, go right → null | [9, 3, 15] |
| back to dfs(20) | 20 | Add 20, go right → dfs(7) | [9, 3, 15, 20] |
| dfs(7) | 7 | Go left → null, add 7, go right → null | [9, 3, 15, 20, 7] |

**Result:** `[9, 3, 15, 20, 7]` ✓

## Complexity
- **Time:** O(n) — every node is visited exactly once in all traversal variants
- **Space:** O(n) for BFS (queue can hold an entire level ≈ n/2 in a complete tree); O(h) for DFS recursion stack where h = tree height (O(log n) balanced, O(n) skewed)

## Edge Cases
- Empty tree (root is null) — return empty list
- Single node — all traversals return `[root.val]`
- Completely skewed tree (linked-list shape) — DFS recursion depth = n, BFS queue size = 1
- Complete binary tree — BFS queue reaches max width n/2 at the last level
- Tree with only left children or only right children

## Common Mistakes
- **Forgetting to check `levelSize` in BFS:** Processing nodes without bounding by level size mixes levels together.
- **Using `queue.size()` inside the for-loop condition:** The queue size changes as you enqueue children; capture it before the loop.
- **Iterative inorder — forgetting `curr = curr.right`:** After popping and processing, you must move to the right subtree.
- **Confusing preorder vs postorder iterative:** Preorder uses a stack pushing right then left; postorder can reverse a modified preorder or use two stacks.
- **Stack overflow on deep trees with recursion:** Switch to iterative for trees with depth > ~10,000.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Binary Tree Level Order Traversal | Medium | BFS with queue; capture level size before inner loop | https://leetcode.com/problems/binary-tree-level-order-traversal/ |
| 2 | Binary Tree Zigzag Level Order Traversal | Medium | BFS + alternate between appending left-to-right and right-to-left per level | https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal/ |
| 3 | Binary Tree Right Side View | Medium | BFS — the last node in each level is visible; or DFS right-first tracking depth | https://leetcode.com/problems/binary-tree-right-side-view/ |
| 4 | Binary Tree Inorder Traversal | Easy | Recursive: left → root → right; iterative: push all lefts, pop, go right | https://leetcode.com/problems/binary-tree-inorder-traversal/ |
| 5 | Binary Tree Preorder Traversal | Easy | Recursive: root → left → right; iterative: stack push right then left | https://leetcode.com/problems/binary-tree-preorder-traversal/ |
| 6 | Binary Tree Postorder Traversal | Easy | Recursive: left → right → root; iterative: reverse of modified preorder or two stacks | https://leetcode.com/problems/binary-tree-postorder-traversal/ |
| 7 | Average of Levels in Binary Tree | Easy | BFS; sum each level and divide by level size | https://leetcode.com/problems/average-of-levels-in-binary-tree/ |
| 8 | N-ary Tree Level Order Traversal | Medium | Same BFS template but iterate over `node.children` list instead of left/right | https://leetcode.com/problems/n-ary-tree-level-order-traversal/ |
| 9 | Maximum Depth of Binary Tree | Easy | DFS postorder: depth = 1 + max(left depth, right depth); or BFS counting levels | https://leetcode.com/problems/maximum-depth-of-binary-tree/ |
| 10 | Minimum Depth of Binary Tree | Easy | BFS returns at the first leaf encountered; DFS must handle one-child nodes carefully | https://leetcode.com/problems/minimum-depth-of-binary-tree/ |
