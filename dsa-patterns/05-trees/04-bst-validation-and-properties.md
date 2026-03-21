# Pattern: BST Validation & Properties

## How to Identify This Pattern

### Trigger Words
- "binary search tree", "BST"
- "validate BST", "is valid BST"
- "kth smallest", "kth largest"
- "inorder successor", "inorder predecessor"
- "search", "insert", "delete" in a BST
- "range sum", "trim", "convert BST"

### Constraint Clues
- The tree is explicitly a BST (left < root < right)
- Need to leverage sorted (inorder) property for efficiency
- Operations should run in O(h) time instead of O(n)
- "Sorted order" or "next greater/smaller" in a tree context

### When to Use This vs Similar Patterns
| Consideration | BST Properties | General Tree DFS | Inorder Traversal |
|---|---|---|---|
| Key constraint | left < root < right | No ordering | Produces sorted output on BST |
| Search complexity | O(h) | O(n) | O(n) |
| When to use | Searching, validating, range queries | Any tree problem | Kth element, validation, conversion |
| Typical approach | Compare and go left/right | Visit all nodes | Full traversal or early stop |

## Core Approach

### Variant 1: BST Validation (Range Check)
1. Pass a valid range `(min, max)` to each node — initially `(-∞, +∞)`.
2. Check that `node.val` is within `(min, max)`.
3. Recurse left with updated range `(min, node.val)`.
4. Recurse right with updated range `(node.val, max)`.
5. All constraints satisfied → valid BST.

### Variant 2: Kth Smallest via Inorder
1. Perform an inorder traversal (left → root → right).
2. Maintain a counter; decrement at each visited node.
3. When the counter reaches 0, the current node is the kth smallest.
4. Early termination avoids traversing the entire tree.

### Variant 3: Search / Insert / Delete
1. **Search:** Compare target with node, go left or right accordingly.
2. **Insert:** Search for the position; when null is reached, create the new node.
3. **Delete:** Find the node. If it has two children, replace with inorder successor (smallest in right subtree), then delete the successor.

## Java Template

### Validate BST
```java
public boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

private boolean validate(TreeNode node, long min, long max) {
    if (node == null) return true;
    if (node.val <= min || node.val >= max) return false;
    return validate(node.left, min, node.val)
        && validate(node.right, node.val, max);
}
```

### Kth Smallest Element
```java
int count, result;

public int kthSmallest(TreeNode root, int k) {
    count = k;
    inorder(root);
    return result;
}

private void inorder(TreeNode node) {
    if (node == null) return;
    inorder(node.left);
    count--;
    if (count == 0) {
        result = node.val;
        return;
    }
    inorder(node.right);
}
```

### Delete Node in BST
```java
public TreeNode deleteNode(TreeNode root, int key) {
    if (root == null) return null;
    if (key < root.val) {
        root.left = deleteNode(root.left, key);
    } else if (key > root.val) {
        root.right = deleteNode(root.right, key);
    } else {
        if (root.left == null) return root.right;
        if (root.right == null) return root.left;
        TreeNode successor = findMin(root.right);
        root.val = successor.val;
        root.right = deleteNode(root.right, successor.val);
    }
    return root;
}

private TreeNode findMin(TreeNode node) {
    while (node.left != null) node = node.left;
    return node;
}
```

## Dry Run Example

**Problem:** Validate BST on tree:
```
        5
       / \
      1   4
         / \
        3   6
```

| Node | Range (min, max) | Valid? | Reason |
|------|-------------------|--------|--------|
| 5 | (-∞, +∞) | Yes | — |
| 1 | (-∞, 5) | Yes | 1 < 5 |
| 4 | (5, +∞) | **No** | 4 < 5, violates lower bound |

**Result:** `false` ✓ (node 4 is in right subtree of 5 but 4 < 5)

**Problem:** Kth Smallest, k = 3 on BST:
```
        5
       / \
      3   6
     / \
    2   4
   /
  1
```

Inorder traversal: `1, 2, 3, 4, 5, 6`

| Visit | Node | Count | Found? |
|-------|------|-------|--------|
| 1 | 1 | 3→2 | No |
| 2 | 2 | 2→1 | No |
| 3 | 3 | 1→0 | **Yes → return 3** |

**Result:** `3` ✓

## Complexity
- **Time:** O(h) for search/insert/delete in a balanced BST (O(log n)); O(n) worst case for skewed tree; O(n) for validation and kth smallest (may traverse all)
- **Space:** O(h) for recursion stack

## Edge Cases
- Empty tree → valid BST by definition (return true)
- Single node → always valid
- `Integer.MIN_VALUE` or `Integer.MAX_VALUE` as node values — use `long` for range bounds to avoid overflow
- BST with duplicate values — standard BST definition requires strict inequality; clarify with the problem
- Deleting the root node — ensure re-rooting logic works
- Kth smallest where k equals tree size — must traverse the entire tree

## Common Mistakes
- **Using `int` bounds for validation:** Node values can be `Integer.MIN_VALUE` or `Integer.MAX_VALUE`; use `long` or `TreeNode` references for bounds.
- **Using `<=` instead of `<` for BST validation:** BSTs require strict inequality; equal values in left or right subtree make it invalid.
- **Not considering duplicates in BST operations:** Some problems allow duplicates; ensure consistent placement (e.g., always in right subtree).
- **Delete — forgetting the two-children case:** Must replace with inorder successor (or predecessor), not just remove the node.
- **Kth smallest — not short-circuiting:** After finding the answer, continuing the traversal wastes time; return early.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Validate Binary Search Tree | Medium | Pass (min, max) range down; use long to avoid integer boundary issues | https://leetcode.com/problems/validate-binary-search-tree/ |
| 2 | Kth Smallest Element in a BST | Medium | Inorder traversal gives sorted order; count down k and stop early | https://leetcode.com/problems/kth-smallest-element-in-a-bst/ |
| 3 | Inorder Successor in BST | Medium | If node has right child, go right then all left; otherwise track from root going left | https://leetcode.com/problems/inorder-successor-in-bst/ |
| 4 | Search in a Binary Search Tree | Easy | Compare with root; go left if smaller, right if larger; O(h) | https://leetcode.com/problems/search-in-a-binary-search-tree/ |
| 5 | Delete Node in a BST | Medium | Three cases: leaf, one child, two children (replace with inorder successor) | https://leetcode.com/problems/delete-node-in-a-bst/ |
| 6 | Convert BST to Greater Tree | Medium | Reverse inorder (right → root → left) with running sum | https://leetcode.com/problems/convert-bst-to-greater-tree/ |
| 7 | Trim a Binary Search Tree | Medium | If root < low, trim left subtree entirely (return trim of right); analogous for high | https://leetcode.com/problems/trim-a-binary-search-tree/ |
| 8 | Range Sum of BST | Easy | DFS pruning: skip left if node.val <= low, skip right if node.val >= high | https://leetcode.com/problems/range-sum-of-bst/ |
| 9 | Closest Binary Search Tree Value | Easy | Track closest while binary-searching; update when |node.val - target| improves | https://leetcode.com/problems/closest-binary-search-tree-value/ |
| 10 | Two Sum IV - Input is a BST | Easy | Inorder to sorted array + two pointers; or use HashSet during DFS | https://leetcode.com/problems/two-sum-iv-input-is-a-bst/ |
