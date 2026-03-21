# DSA Pattern Identification Guide

## How to Use This Guide

When you encounter a new problem, follow this decision tree to identify the likely pattern. Then open the specific pattern file for the approach, template, and practice problems.

---

## Step 1: What Data Structure is the Input?

| Input Type | Go To |
|-----------|-------|
| Array / String | [Step 2A: Array/String Problems](#step-2a-arraystring-problems) |
| Linked List | [Step 2B: Linked List Problems](#step-2b-linked-list-problems) |
| Tree / Binary Tree | [Step 2C: Tree Problems](#step-2c-tree-problems) |
| Graph / Matrix (grid) | [Step 2D: Graph Problems](#step-2d-graph-problems) |
| No specific input (design) | [Step 2E: Design Problems](#step-2e-design-problems) |

---

## Step 2A: Array/String Problems

```
Is the array sorted (or can sorting help)?
├── YES
│   ├── "Find pair/triplet with target sum" → TWO POINTERS
│   ├── "Find element / boundary / insertion point" → BINARY SEARCH
│   ├── "Merge two sorted arrays" → TWO POINTERS (merge variant)
│   └── "Minimize/maximize some value given a constraint" → BINARY SEARCH ON ANSWER
│
├── NO — Is it about a contiguous subarray/substring?
│   ├── YES
│   │   ├── "Fixed-size window" (e.g., max sum of k elements) → SLIDING WINDOW (fixed)
│   │   ├── "Variable-size window with condition" (longest/shortest) → SLIDING WINDOW (variable)
│   │   ├── "Subarray sum equals K / divisible by K" → PREFIX SUMS + HASHMAP
│   │   └── "Maximum subarray sum" → KADANE'S ALGORITHM
│   │
│   ├── NO — Is it about counting / grouping / lookup?
│   │   ├── "Find duplicates / frequency / anagrams" → HASHMAPS & COUNTING
│   │   ├── "Two Sum (unsorted)" → HASHMAP
│   │   └── "Group elements by property" → HASHMAP
│   │
│   └── NO — Other array operations?
│       ├── "Rearrange / partition / sort with constraint" → SORTING + CUSTOM COMPARATORS
│       ├── "Merge overlapping intervals" → INTERVALS (sort + merge)
│       ├── "Remove duplicates in-place" → TWO POINTERS
│       ├── "Rotate array" → REVERSAL TECHNIQUE
│       └── "Next permutation / rearrangement" → SPECIFIC ALGORITHM
```

---

## Step 2B: Linked List Problems

```
What operation is needed?
├── "Detect cycle / find cycle start" → FAST & SLOW POINTERS
├── "Find middle element" → FAST & SLOW POINTERS
├── "Check if palindrome" → FAST & SLOW + REVERSAL
├── "Reverse the list (or part of it)" → DUMMY NODE & REVERSAL
├── "Merge two/k sorted lists" → MERGE & RECURSION
├── "Remove nth node from end" → TWO POINTERS (with gap)
├── "Intersection of two lists" → TWO POINTERS
└── "Swap/reorder nodes" → DUMMY NODE TECHNIQUE
```

---

## Step 2C: Tree Problems

```
What are you asked to do?
├── "Visit all nodes / level by level" → TREE TRAVERSALS (BFS/DFS)
│   ├── "Level order / zigzag / right side view" → BFS (Queue)
│   └── "Preorder / inorder / postorder" → DFS (Recursion/Stack)
│
├── "Build tree from traversal data" → CONSTRUCTION & SERIALIZATION
├── "Find path sum / all paths / max path" → PATH SUM & ROOT-TO-LEAF
├── "Validate BST / find kth element" → BST VALIDATION & PROPERTIES
├── "Check symmetric / mirror / identical" → MIRROR & SYMMETRY
├── "Find lowest common ancestor" → LCA
└── "Calculate height / diameter / balance" → DFS (post-order)
```

---

## Step 2D: Graph Problems

```
What are you asked to do?
├── "Explore all connected nodes / count islands" → BFS or DFS TRAVERSAL
├── "Find shortest path"
│   ├── Unweighted graph → BFS
│   ├── Weighted graph (no negative) → DIJKSTRA
│   └── Weighted graph (with negative) → BELLMAN-FORD
│
├── "Detect cycle"
│   ├── Directed graph → DFS with coloring / TOPOLOGICAL SORT
│   └── Undirected graph → UNION-FIND or DFS
│
├── "Order tasks with dependencies" → TOPOLOGICAL SORT
├── "Group into two sets / check bipartite" → BIPARTITE CHECK (BFS/DFS coloring)
├── "Find connected components / check connectivity" → UNION-FIND or BFS/DFS
├── "Minimum spanning tree" → KRUSKAL (Union-Find) or PRIM
└── "Find if path exists" → BFS / DFS / UNION-FIND
```

---

## Step 2E: Design Problems

```
What structure do you need?
├── "Stack with O(1) min/max" → TWO STACKS / MONOTONIC STACK
├── "Queue from stacks (or vice versa)" → STACK DESIGN
├── "LRU/LFU Cache" → HASHMAP + DOUBLY LINKED LIST
├── "Implement Trie" → TRIE
├── "Priority-based operations" → HEAP
└── "Efficient range queries" → PREFIX SUMS / SEGMENT TREE
```

---

## Trigger Word Glossary

These words/phrases in problem statements strongly hint at specific patterns:

| Trigger Words | Pattern | Example Problem |
|--------------|---------|-----------------|
| "sorted array", "find efficiently" | **Binary Search** | Search Insert Position |
| "pair with sum", "two numbers" | **Two Pointers** (sorted) or **HashMap** (unsorted) | Two Sum |
| "contiguous subarray", "window" | **Sliding Window** | Maximum Sum Subarray of Size K |
| "longest substring without repeating" | **Sliding Window** (variable) | Longest Substring Without Repeating Characters |
| "subarray sum equals K" | **Prefix Sums + HashMap** | Subarray Sum Equals K |
| "maximum subarray" | **Kadane's Algorithm** | Maximum Subarray |
| "top K", "kth largest/smallest" | **Heap** | Kth Largest Element |
| "merge intervals", "overlapping" | **Intervals** (sort + sweep) | Merge Intervals |
| "next greater element" | **Monotonic Stack** | Next Greater Element |
| "valid parentheses", "balanced" | **Stack** | Valid Parentheses |
| "level order", "breadth-first" | **BFS** (Queue) | Binary Tree Level Order Traversal |
| "all paths", "root to leaf" | **DFS** (Backtracking) | Path Sum II |
| "number of islands", "connected" | **BFS/DFS** on grid | Number of Islands |
| "shortest path", "minimum steps" | **BFS** (unweighted) / **Dijkstra** (weighted) | Word Ladder |
| "course schedule", "prerequisites" | **Topological Sort** | Course Schedule |
| "detect cycle" | **Fast & Slow** (LL) / **DFS coloring** (graph) | Linked List Cycle |
| "minimum cost", "maximum profit" | **DP** or **Greedy** | Coin Change |
| "how many ways" | **DP** | Climbing Stairs |
| "subsequence", "subset" | **DP** or **Backtracking** | Longest Increasing Subsequence |
| "permutation", "combination" | **Backtracking** | Permutations |
| "partition into groups" | **Backtracking** or **DP** | Partition Equal Subset Sum |
| "schedule", "select maximum" | **Greedy** | Activity Selection |
| "in-place", "constant space" | **Two Pointers** or **Bit Manipulation** | Move Zeroes |
| "palindrome" | **Two Pointers** or **DP** | Longest Palindromic Substring |
| "anagram", "frequency" | **HashMap / Counting** | Group Anagrams |
| "linked list cycle" | **Fast & Slow Pointers** | Linked List Cycle |
| "merge K sorted" | **Heap** | Merge K Sorted Lists |
| "median" | **Two Heaps** | Find Median from Data Stream |
| "LCA", "common ancestor" | **LCA Algorithm** | Lowest Common Ancestor |
| "validate BST" | **BST Properties** (inorder) | Validate Binary Search Tree |

---

## Pattern Comparison: When to Use What

### Sliding Window vs Two Pointers vs Prefix Sums

| Feature | Sliding Window | Two Pointers | Prefix Sums |
|---------|---------------|-------------|-------------|
| **Input** | Array/String | Sorted array (usually) | Array |
| **Goal** | Contiguous subarray/substring | Find pair/triplet | Subarray sum queries |
| **Window** | Expands right, shrinks left | Both move inward or same direction | Precompute, then query |
| **When** | "Longest/shortest substring with condition" | "Two numbers that sum to X" | "Subarray sum equals K" |
| **Time** | O(n) | O(n) | O(n) precompute, O(1) query |

### BFS vs DFS (Graphs/Trees)

| Feature | BFS | DFS |
|---------|-----|-----|
| **Data structure** | Queue | Stack / Recursion |
| **Explores** | Level by level | Depth-first |
| **Best for** | Shortest path (unweighted), level-order | Exhaustive search, path finding, cycle detection |
| **Space** | O(width of tree/graph) | O(height of tree/graph) |

### Greedy vs DP

| Feature | Greedy | Dynamic Programming |
|---------|--------|-------------------|
| **Makes** | Locally optimal choice | Considers all subproblems |
| **Guarantees optimal?** | Only with greedy-choice property | Always (if correct recurrence) |
| **Time** | Usually O(n log n) | Usually O(n^2) or O(n * capacity) |
| **When** | Activity selection, Huffman, fractional knapsack | 0/1 knapsack, LCS, edit distance |
| **Key test** | "Does choosing the best now always lead to global best?" | "Do subproblems overlap?" |

### HashMap vs Sorting

| Feature | HashMap | Sorting |
|---------|---------|---------|
| **Time** | O(n) average | O(n log n) |
| **Space** | O(n) | O(1) to O(n) |
| **When** | Need O(1) lookup, counting, grouping | Need order, merge intervals, custom ordering |

---

## Recommended Study Order

Based on LeetCode profile (92 solved, strong Arrays, weak Trees/Graphs/DP):

### Phase 1: Strengthen Foundations (Weeks 1-3)
1. Two Pointers (reinforce)
2. Sliding Window (reinforce)
3. Prefix Sums (new)
4. Hashmaps & Counting (reinforce)
5. Kadane's Algorithm (new)
6. Binary Search — Classic (reinforce)
7. Binary Search — Answer Space (new)

### Phase 2: New Data Structures (Weeks 4-6)
8. Stacks — Monotonic Stack
9. Stacks — Parentheses & Expressions
10. Linked Lists — Fast & Slow
11. Linked Lists — Reversal
12. Heaps — Top K & Kth Element
13. Heaps — Two Heaps

### Phase 3: Trees (Weeks 7-9)
14. Tree Traversals (BFS/DFS)
15. Path Sum & Root-to-Leaf
16. BST Validation & Properties
17. Mirror & Symmetry
18. Construction & Serialization
19. LCA

### Phase 4: Graphs (Weeks 10-12)
20. BFS Traversal
21. DFS Traversal
22. Union-Find
23. Topological Sort
24. Shortest Path
25. Cycle Detection

### Phase 5: DP & Advanced (Weeks 13-16)
26. 1D DP Basics
27. 2D DP (Grid & LCS)
28. DP on Strings
29. Knapsack Variants
30. Interval DP
31. DP on Trees

### Phase 6: Problem-Solving Patterns (Weeks 17-18)
32. Greedy
33. Backtracking — Subsets & Combinations
34. Backtracking — Permutations
35. Intervals

---

## The Pattern Recognition Process

When you see a new problem, follow these 4 steps:

1. **Read the problem statement** — highlight keywords (sorted, contiguous, shortest path, etc.)
2. **Check constraints** — n <= 10^4 suggests O(n^2) is OK; n <= 10^5 needs O(n log n); n <= 10^6 needs O(n)
3. **Match to pattern** — use the decision tree above
4. **Verify** — does the pattern's approach actually solve this problem? If not, try the next closest pattern

### Constraint-Based Complexity Guide

| Constraint (n) | Max Acceptable Complexity | Likely Patterns |
|----------------|--------------------------|-----------------|
| n <= 10 | O(n!) or O(2^n) | Backtracking, brute force |
| n <= 20 | O(2^n) | Backtracking, bitmask DP |
| n <= 100 | O(n^3) | DP, Floyd-Warshall |
| n <= 1,000 | O(n^2) | DP, nested loops |
| n <= 10,000 | O(n^2) borderline | DP, two pointers |
| n <= 100,000 | O(n log n) | Sorting, binary search, heap |
| n <= 1,000,000 | O(n) | Sliding window, two pointers, prefix sums, hashmap |
| n <= 10^9 | O(log n) or O(1) | Binary search, math |
