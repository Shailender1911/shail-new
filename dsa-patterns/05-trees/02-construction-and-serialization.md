# Pattern: Tree Construction & Serialization

## How to Identify This Pattern

### Trigger Words
- "construct", "build", "rebuild the tree"
- "from preorder and inorder", "from inorder and postorder"
- "serialize", "deserialize", "encode", "decode"
- "flatten", "convert to linked list"
- "sorted array to BST", "sorted list to BST"
- "verify serialization"

### Constraint Clues
- Given two traversal arrays and asked to reconstruct the unique tree
- Need to convert a tree to a string and back without losing structure
- Input is a sorted array/list and the output must be a height-balanced BST
- Must preserve tree structure across some transformation

### When to Use This vs Similar Patterns
| Consideration | Construction | Serialization | Flatten/Convert |
|---|---|---|---|
| Input | Traversal arrays | Tree object | Tree object |
| Output | Tree object | String / Stream | Modified tree / List |
| Core technique | Divide and conquer on arrays | Preorder DFS with null markers | In-place pointer rewiring |
| Key insight | Root splits left/right subarrays | Null markers preserve structure | Reverse postorder or Morris-like |

## Core Approach

### Variant 1: Construct from Preorder + Inorder
1. The first element of preorder is always the root.
2. Find that root's index in inorder — everything left of it is the left subtree, everything right is the right subtree.
3. Recurse on the left and right slices of both arrays.
4. Use a HashMap for O(1) index lookup in inorder.

### Variant 2: Serialize / Deserialize (Preorder with null markers)
1. **Serialize:** Preorder DFS; append each node's value; append a sentinel (e.g., "null") for null children. Join with a delimiter.
2. **Deserialize:** Split the string by delimiter into a queue. Dequeue the front: if sentinel, return null; otherwise, create a node, recurse for left child, then right child.

### Variant 3: Sorted Array → Balanced BST
1. Find the middle element of the current range — this becomes the root (ensures balance).
2. Recurse on the left half for the left subtree.
3. Recurse on the right half for the right subtree.
4. Base case: start > end → return null.

## Java Template

### Construct from Preorder + Inorder
```java
public TreeNode buildTree(int[] preorder, int[] inorder) {
    Map<Integer, Integer> inMap = new HashMap<>();
    for (int i = 0; i < inorder.length; i++) {
        inMap.put(inorder[i], i);
    }
    return build(preorder, 0, preorder.length - 1,
                 inorder, 0, inorder.length - 1, inMap);
}

private TreeNode build(int[] preorder, int preStart, int preEnd,
                       int[] inorder, int inStart, int inEnd,
                       Map<Integer, Integer> inMap) {
    if (preStart > preEnd || inStart > inEnd) return null;

    int rootVal = preorder[preStart];
    TreeNode root = new TreeNode(rootVal);
    int inRoot = inMap.get(rootVal);
    int leftSize = inRoot - inStart;

    root.left = build(preorder, preStart + 1, preStart + leftSize,
                      inorder, inStart, inRoot - 1, inMap);
    root.right = build(preorder, preStart + leftSize + 1, preEnd,
                       inorder, inRoot + 1, inEnd, inMap);
    return root;
}
```

### Serialize / Deserialize
```java
public String serialize(TreeNode root) {
    StringBuilder sb = new StringBuilder();
    serializeDFS(root, sb);
    return sb.toString();
}

private void serializeDFS(TreeNode node, StringBuilder sb) {
    if (node == null) {
        sb.append("null,");
        return;
    }
    sb.append(node.val).append(",");
    serializeDFS(node.left, sb);
    serializeDFS(node.right, sb);
}

public TreeNode deserialize(String data) {
    Queue<String> queue = new LinkedList<>(Arrays.asList(data.split(",")));
    return deserializeDFS(queue);
}

private TreeNode deserializeDFS(Queue<String> queue) {
    String val = queue.poll();
    if ("null".equals(val)) return null;
    TreeNode node = new TreeNode(Integer.parseInt(val));
    node.left = deserializeDFS(queue);
    node.right = deserializeDFS(queue);
    return node;
}
```

### Sorted Array → BST
```java
public TreeNode sortedArrayToBST(int[] nums) {
    return buildBST(nums, 0, nums.length - 1);
}

private TreeNode buildBST(int[] nums, int left, int right) {
    if (left > right) return null;
    int mid = left + (right - left) / 2;
    TreeNode root = new TreeNode(nums[mid]);
    root.left = buildBST(nums, left, mid - 1);
    root.right = buildBST(nums, mid + 1, right);
    return root;
}
```

## Dry Run Example

**Problem:** Construct tree from preorder = `[3, 9, 20, 15, 7]`, inorder = `[9, 3, 15, 20, 7]`

| Step | preorder root | inorder split | Left subtree | Right subtree |
|------|---------------|---------------|--------------|---------------|
| 1 | 3 | [9] **3** [15,20,7] | preorder=[9], inorder=[9] | preorder=[20,15,7], inorder=[15,20,7] |
| 2a (left) | 9 | [] **9** [] | null | null → leaf node 9 |
| 2b (right) | 20 | [15] **20** [7] | preorder=[15], inorder=[15] | preorder=[7], inorder=[7] |
| 3a | 15 | leaf | — | — |
| 3b | 7 | leaf | — | — |

**Result:**
```
        3
       / \
      9   20
         /  \
        15   7
```
✓

## Complexity
- **Time:** O(n) for construction (with hash map for inorder index); O(n) for serialization/deserialization
- **Space:** O(n) for the hash map and recursion stack; O(n) for the serialized string

## Edge Cases
- Empty input arrays → return null
- Single element → leaf node
- Skewed tree (all left or all right children) → recursion depth = n
- Duplicate values in construction — standard problems guarantee unique values; if duplicates exist, the tree is not uniquely determined
- Sorted array of even length — choosing `left + (right - left) / 2` picks left-middle; either is valid for a balanced BST

## Common Mistakes
- **Not using a hash map for inorder index lookup:** Linear search makes construction O(n²) instead of O(n).
- **Off-by-one on `leftSize` calculation:** `leftSize = inRoot - inStart`, not `inRoot - inStart + 1`, because it counts elements strictly to the left.
- **Serialize/deserialize format mismatch:** The delimiter and null marker must be consistent between serialize and deserialize.
- **Forgetting null markers in serialization:** Without them, the tree structure is ambiguous and cannot be reconstructed.
- **Sorted list to BST — advancing the list pointer:** Use a class-level or array-wrapped pointer so recursive calls share the same advancing head.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Construct Binary Tree from Preorder and Inorder Traversal | Medium | First of preorder = root; find in inorder to split left/right; use hash map | https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/ |
| 2 | Construct Binary Tree from Inorder and Postorder Traversal | Medium | Last of postorder = root; same split logic; recurse right subtree before left | https://leetcode.com/problems/construct-binary-tree-from-inorder-and-postorder-traversal/ |
| 3 | Serialize and Deserialize Binary Tree | Hard | Preorder DFS with "null" markers; deserialize with a queue consuming tokens | https://leetcode.com/problems/serialize-and-deserialize-binary-tree/ |
| 4 | Maximum Binary Tree | Medium | Find max in range → root; recurse on left slice and right slice | https://leetcode.com/problems/maximum-binary-tree/ |
| 5 | Construct Binary Search Tree from Preorder Traversal | Medium | Use upper-bound recursion: each call consumes values less than the bound | https://leetcode.com/problems/construct-binary-search-tree-from-preorder-traversal/ |
| 6 | Convert Sorted Array to Binary Search Tree | Easy | Pick middle as root for balance; recurse on left and right halves | https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree/ |
| 7 | Convert Sorted List to Binary Search Tree | Medium | Slow/fast pointer to find mid, or simulate inorder with advancing list pointer | https://leetcode.com/problems/convert-sorted-list-to-binary-search-tree/ |
| 8 | Flatten Binary Tree to Linked List | Medium | Reverse postorder (right → left → root) with a prev pointer; or iterative with right = left trick | https://leetcode.com/problems/flatten-binary-tree-to-linked-list/ |
| 9 | Construct String from Binary Tree | Medium | Preorder with parentheses; omit right parens only when both children are null | https://leetcode.com/problems/construct-string-from-binary-tree/ |
| 10 | Verify Preorder Serialization of a Binary Tree | Medium | Track available slots: root gives 1 slot, each value consumes 1 and adds 2, each null consumes 1 | https://leetcode.com/problems/verify-preorder-serialization-of-a-binary-tree/ |
