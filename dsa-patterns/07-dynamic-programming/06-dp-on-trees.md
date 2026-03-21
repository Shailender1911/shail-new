# Pattern: DP on Trees (Post-Order Traversal DP)

## How to Identify This Pattern

### Trigger Words
- "binary tree", "rooted tree" with optimization/counting
- "maximum path sum", "longest path"
- "rob houses arranged in a tree"
- "minimum cameras", "minimum operations on a tree"
- "number of unique BSTs"
- "distribute coins", "split tree"
- "diameter of tree"

### Constraint Clues
- Input is a tree (binary tree or general rooted tree)
- The answer for a node depends on the answers of its children (bottom-up)
- Need to make a decision at each node: include/exclude, place camera or not
- Answer combines information from left and right subtrees
- n ≤ 10⁵ nodes → O(n) tree DP is expected
- Problem involves "choosing a subset of nodes" subject to tree constraints (e.g., no two adjacent)

### When to Use This vs Similar Patterns
| Consideration | Tree DP | 1D DP | Graph DP | Greedy on Tree |
|---|---|---|---|---|
| Structure | Tree (acyclic, rooted) | Linear array | General graph | Tree |
| Traversal | Post-order (bottom-up) | Left to right | Topological sort | Any order |
| State | `dp[node]` with multiple values | `dp[i]` | `dp[node]` | Local decision |
| Dependencies | Children → parent | `i-1, i-2, ...` | Predecessors | None |
| When to use | Optimal structure on tree | Sequential decisions | DAG optimization | Greedy proof exists |

## Core Approach

1. **Define state:** For each node, define what information to compute. Often two values:
   - `dp[node][0]` = best answer if this node is **not** selected
   - `dp[node][1]` = best answer if this node **is** selected
   - Or return multiple values from each DFS call (e.g., "max path ending here" vs "max path anywhere in subtree").

2. **Write recurrence:** Combine children's results:
   - If node is selected: children cannot be selected → use `dp[child][0]`
   - If node is not selected: children can be selected or not → use `max(dp[child][0], dp[child][1])`

3. **Base cases:** Leaf nodes or null nodes return base values (0, -infinity, etc.).

4. **Build order:** **Post-order traversal** — solve children first, then compute the parent's answer.

5. **Answer location:** `dp[root]` or `max(dp[root][0], dp[root][1])`, or a global variable updated during traversal.

## Java Template

### Tree DP with Two States (House Robber III)
```java
public int rob(TreeNode root) {
    int[] result = dfs(root);
    return Math.max(result[0], result[1]);
}

// returns [notRob, rob]
private int[] dfs(TreeNode node) {
    if (node == null) return new int[]{0, 0};

    int[] left = dfs(node.left);
    int[] right = dfs(node.right);

    int notRob = Math.max(left[0], left[1]) + Math.max(right[0], right[1]);
    int rob = node.val + left[0] + right[0];

    return new int[]{notRob, rob};
}
```

### Path Sum Through Nodes (Binary Tree Maximum Path Sum)
```java
int maxSum;

public int maxPathSum(TreeNode root) {
    maxSum = Integer.MIN_VALUE;
    dfs(root);
    return maxSum;
}

// returns max sum of path starting from node going downward
private int dfs(TreeNode node) {
    if (node == null) return 0;

    int left = Math.max(0, dfs(node.left));
    int right = Math.max(0, dfs(node.right));

    // Path through this node (could be the global answer)
    maxSum = Math.max(maxSum, left + right + node.val);

    // Return max single-branch path for parent's use
    return node.val + Math.max(left, right);
}
```

### Three-State Tree DP (Binary Tree Cameras)
```java
int cameras;

public int minCameraCover(TreeNode root) {
    cameras = 0;
    // If root is not covered, it needs a camera
    if (dfs(root) == 0) cameras++;
    return cameras;
}

// 0 = not covered, 1 = has camera, 2 = covered (no camera)
private int dfs(TreeNode node) {
    if (node == null) return 2; // null nodes are considered covered

    int left = dfs(node.left);
    int right = dfs(node.right);

    if (left == 0 || right == 0) {
        // A child is not covered → must place camera here
        cameras++;
        return 1;
    }
    if (left == 1 || right == 1) {
        // A child has a camera → this node is covered
        return 2;
    }
    // Both children are covered but neither has a camera →
    // this node is NOT covered (parent must handle it)
    return 0;
}
```

### Counting Structurally Unique BSTs (Catalan Number DP)
```java
public int numTrees(int n) {
    int[] dp = new int[n + 1];
    dp[0] = 1;
    dp[1] = 1;

    for (int nodes = 2; nodes <= n; nodes++) {
        for (int root = 1; root <= nodes; root++) {
            dp[nodes] += dp[root - 1] * dp[nodes - root];
        }
    }
    return dp[n];
}
```

## Dry Run Example

**Problem:** House Robber III

```
     3
    / \
   2   3
    \   \
     3   1
```

Post-order DFS returns `[notRob, rob]` for each node:

| Node | Left child result | Right child result | notRob | rob | Result |
|---|---|---|---|---|---|
| Leaf 3 (left→right child of 2) | [0,0] | [0,0] | 0 | 3 | [0, 3] |
| Node 2 | [0,0] | [0,3] | max(0,0)+max(0,3)=3 | 2+0+0=2 | [3, 2] |
| Leaf 1 (right child of 3) | [0,0] | [0,0] | 0 | 1 | [0, 1] |
| Node 3 (right child of root) | [0,0] | [0,1] | max(0,0)+max(0,1)=1 | 3+0+0=3 | [1, 3] |
| Root 3 | [3,2] | [1,3] | max(3,2)+max(1,3)=3+3=6 | 3+3+1=7 | [6, 7] |

**Answer:** `max(6, 7) = 7` (rob root 3 + leaf 3 + right child 1 = 3 + 3 + 1 = 7) ✓

## Complexity
- **Time:** O(n) — visit each node exactly once in post-order
- **Space:** O(h) for recursion stack, where h = tree height (O(log n) balanced, O(n) skewed)

## Edge Cases
- Empty tree (null root) → return 0 or appropriate base value
- Single node → answer is just that node's value (or 1 camera, etc.)
- Skewed tree (linked list shape) → recursion depth = n; may need iterative approach or increased stack size
- All values are negative (Max Path Sum) → must pick at least one node; don't default to 0
- Tree with only left or only right children → ensure null checks don't skip valid subtrees
- Unique BSTs with n = 0 → 1 (empty tree is valid)
- Large trees (n = 10⁵) → ensure O(n) time; avoid recomputation by returning arrays/pairs from DFS

## Common Mistakes
- **Forgetting to clamp negative subtree contributions to 0:** In Max Path Sum, a subtree with negative sum should be ignored (take 0 instead), or it worsens the path.
- **Confusing "path through node" with "path starting at node":** Max Path Sum's global answer allows `left + node + right`, but the value returned to the parent must be a single-branch path.
- **Not handling the root separately in camera problems:** After DFS, if the root returns "not covered," you must add one more camera.
- **House Robber III: re-traversing subtrees:** Naive recursion without memoization leads to O(2ⁿ). Return pairs `[rob, notRob]` to avoid recomputation.
- **Diameter: forgetting to track global max:** The diameter may not pass through the root. Update a global variable inside DFS.
- **Unique BSTs: confusing with Unique BSTs II** — Unique BSTs (I) counts trees (Catalan number DP). Unique BSTs II generates all trees (backtracking, return lists of TreeNodes).

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Diameter of Binary Tree | Easy | DFS returns depth; diameter at each node = leftDepth + rightDepth; track global max | https://leetcode.com/problems/diameter-of-binary-tree/ |
| 2 | Count Good Nodes in Binary Tree | Medium | DFS with max-so-far; a node is "good" if its value ≥ all ancestors; count during traversal | https://leetcode.com/problems/count-good-nodes-in-binary-tree/ |
| 3 | Unique Binary Search Trees | Medium | Catalan number DP: `dp[n] = Σ dp[k-1] * dp[n-k]` for each root k; no actual tree needed | https://leetcode.com/problems/unique-binary-search-trees/ |
| 4 | House Robber III | Medium | Two-state tree DP: `[notRob, rob]` per node; rob = val + children's notRob; notRob = max of each child's states | https://leetcode.com/problems/house-robber-iii/ |
| 5 | Binary Tree Maximum Path Sum | Hard | DFS returns max single-branch sum; global max considers left + node + right; clamp negatives to 0 | https://leetcode.com/problems/binary-tree-maximum-path-sum/ |
| 6 | Binary Tree Cameras | Hard | Three-state greedy/DP: 0=not covered, 1=has camera, 2=covered; place cameras at parents of uncovered leaves | https://leetcode.com/problems/binary-tree-cameras/ |
| 7 | Unique Binary Search Trees II | Medium | Recursive generation: for each root k, build all left trees from [1..k-1] and right trees from [k+1..n]; combine | https://leetcode.com/problems/unique-binary-search-trees-ii/ |
| 8 | Distribute Coins in Binary Tree | Medium | Post-order: each node returns excess coins (positive = surplus, negative = deficit); total moves = Σ|excess| at each edge | https://leetcode.com/problems/distribute-coins-in-binary-tree/ |
| 9 | Maximum Product of Splitted Binary Tree | Medium | First DFS to get total sum; second DFS tries each edge cut: product = subtreeSum × (total - subtreeSum); track max | https://leetcode.com/problems/maximum-product-of-splitted-binary-tree/ |
| 10 | Longest Path With Different Adjacent Characters | Hard | DFS returns longest single-branch path with unique adjacent chars; combine top-2 children branches + 1 for global max | https://leetcode.com/problems/longest-path-with-different-adjacent-characters/ |
