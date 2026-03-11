# DSA Pattern Decision Tree

> Before solving ANY problem, ask yourself these questions in order.
> This is your mental framework for pattern recognition during interviews.

---

## The Decision Tree

```
START: Read the problem

1. Is the input SORTED (or can be sorted)?
   ├── YES → Binary Search, Two Pointers, or Merge approach
   └── NO → Continue to Q2

2. Are you looking for OPTIMIZATION (max/min/shortest/longest)?
   ├── Subarray/substring of specific size? → Sliding Window
   ├── Overall max/min with choices? → Dynamic Programming
   ├── Locally optimal → globally optimal? → Greedy
   └── NO → Continue to Q3

3. Do you need FREQUENCIES, DUPLICATES, or LOOKUPS?
   ├── YES → HashMap / HashSet / Counting Array
   └── NO → Continue to Q4

4. Is it about SUBSTRINGS or SUBARRAYS?
   ├── Fixed size? → Sliding Window
   ├── Variable size with condition? → Sliding Window + Two Pointers
   ├── Sum equals K? → Prefix Sum + HashMap
   └── NO → Continue to Q5

5. Is the data a TREE or GRAPH?
   ├── Tree? → DFS (recursive) or BFS (level-order)
   ├── Graph with shortest path? → BFS
   ├── Graph with connectivity? → DFS / Union-Find
   ├── Graph with ordering? → Topological Sort
   └── NO → Continue to Q6

6. Do you need ALL combinations, permutations, or subsets?
   ├── YES → Backtracking
   └── NO → Continue to Q7

7. Is it about MATCHING, PARSING, or NESTING?
   ├── Parentheses/brackets? → Stack
   ├── Next greater/smaller? → Monotonic Stack
   ├── Expression evaluation? → Stack
   └── NO → Continue to Q8

8. Does the problem have OVERLAPPING SUBPROBLEMS?
   ├── YES → Dynamic Programming
   │   ├── Linear sequence? → 1D DP
   │   ├── Two sequences? → 2D DP
   │   ├── Choices with weight/value? → Knapsack DP
   │   └── Range based? → Interval DP
   └── NO → Continue to Q9

9. Do you need the K-th largest/smallest or TOP K?
   ├── YES → Heap (Priority Queue)
   └── NO → Continue to Q10

10. Are you dealing with INTERVALS?
    ├── Merge overlapping? → Sort by start, merge
    ├── Find max overlap? → Sort + sweep line / heap
    ├── Insert interval? → Binary search / linear scan
    └── NO → Brute force or rethink the pattern
```

---

## Pattern Cheat Sheet

### 1. Two Pointers
**When:** Sorted array, find pairs, palindrome check, remove duplicates
**Template:**
```java
int left = 0, right = arr.length - 1;
while (left < right) {
    if (condition met) return result;
    else if (need larger) left++;
    else right--;
}
```
**Classic problems:** Two Sum II, 3Sum, Container With Most Water, Trapping Rain Water

### 2. Sliding Window
**When:** Subarray/substring with condition, fixed or variable size window
**Template (variable size):**
```java
int left = 0;
for (int right = 0; right < n; right++) {
    // expand: add arr[right] to window

    while (window is invalid) {
        // shrink: remove arr[left] from window
        left++;
    }

    // update answer
}
```
**Classic problems:** Longest Substring Without Repeating Chars, Minimum Window Substring

### 3. Binary Search
**When:** Sorted data, search space can be halved, "minimum X such that condition"
**Template:**
```java
int lo = min, hi = max;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (condition(mid)) hi = mid;
    else lo = mid + 1;
}
return lo;
```
**Classic problems:** Search in Rotated Array, Koko Eating Bananas, Find Peak Element

### 4. HashMap / HashSet
**When:** Need O(1) lookup, counting frequencies, finding duplicates
**Template:**
```java
Map<Key, Value> map = new HashMap<>();
for (element : arr) {
    if (map.containsKey(target - element)) return result;
    map.put(element, index);
}
```
**Classic problems:** Two Sum, Group Anagrams, Longest Consecutive Sequence

### 5. Stack
**When:** Matching brackets, next greater element, evaluate expressions
**Template (monotonic stack):**
```java
Deque<Integer> stack = new ArrayDeque<>();
for (int i = 0; i < n; i++) {
    while (!stack.isEmpty() && arr[stack.peek()] < arr[i]) {
        int idx = stack.pop();
        result[idx] = arr[i]; // next greater element
    }
    stack.push(i);
}
```
**Classic problems:** Valid Parentheses, Daily Temperatures, Largest Rectangle in Histogram

### 6. BFS (Breadth-First Search)
**When:** Shortest path (unweighted), level-order traversal, spreading/rotting
**Template:**
```java
Queue<int[]> queue = new LinkedList<>();
queue.offer(start);
visited.add(start);
int level = 0;

while (!queue.isEmpty()) {
    int size = queue.size();
    for (int i = 0; i < size; i++) {
        int[] curr = queue.poll();
        for (int[] neighbor : getNeighbors(curr)) {
            if (!visited.contains(neighbor)) {
                visited.add(neighbor);
                queue.offer(neighbor);
            }
        }
    }
    level++;
}
```
**Classic problems:** Number of Islands, Rotting Oranges, Word Ladder

### 7. DFS (Depth-First Search)
**When:** Explore all paths, tree traversals, connected components, cycle detection
**Template:**
```java
void dfs(int node, boolean[] visited) {
    visited[node] = true;
    for (int neighbor : graph[node]) {
        if (!visited[neighbor]) {
            dfs(neighbor, visited);
        }
    }
}
```
**Classic problems:** Number of Islands, Clone Graph, Course Schedule

### 8. Backtracking
**When:** Generate all combinations/permutations/subsets, constraint satisfaction
**Template:**
```java
void backtrack(List<Integer> current, int start) {
    result.add(new ArrayList<>(current)); // or check if valid

    for (int i = start; i < n; i++) {
        current.add(nums[i]);       // choose
        backtrack(current, i + 1);  // explore
        current.remove(current.size() - 1); // un-choose
    }
}
```
**Classic problems:** Subsets, Permutations, Combination Sum, N-Queens

### 9. Dynamic Programming
**When:** Overlapping subproblems + optimal substructure, counting ways, optimization
**Template (1D):**
```java
int[] dp = new int[n + 1];
dp[0] = base_case;
for (int i = 1; i <= n; i++) {
    dp[i] = recurrence_relation(dp[i-1], dp[i-2], ...);
}
return dp[n];
```
**Classic problems:** Climbing Stairs, Coin Change, Longest Increasing Subsequence, House Robber

### 10. Heap / Priority Queue
**When:** K-th largest/smallest, merge K sorted lists, streaming median
**Template:**
```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>(); // or maxHeap with (a, b) -> b - a
for (int num : nums) {
    minHeap.offer(num);
    if (minHeap.size() > k) minHeap.poll();
}
return minHeap.peek(); // k-th largest
```
**Classic problems:** Kth Largest Element, Merge K Sorted Lists, Find Median from Data Stream

### 11. Greedy
**When:** Local optimal choice leads to global optimal, interval scheduling
**Key insight:** If you can prove that a greedy choice is always safe, use it.
**Classic problems:** Jump Game, Gas Station, Merge Intervals, Meeting Rooms II

### 12. Prefix Sum
**When:** Range sum queries, subarray sum equals K
**Template:**
```java
int[] prefix = new int[n + 1];
for (int i = 0; i < n; i++) {
    prefix[i + 1] = prefix[i] + nums[i];
}
// sum of range [l, r] = prefix[r+1] - prefix[l]
```
**Classic problems:** Subarray Sum Equals K, Range Sum Query

### 13. Topological Sort
**When:** Task ordering with prerequisites, dependency resolution
**Template (Kahn's BFS):**
```java
int[] inDegree = new int[n];
// calculate in-degrees
Queue<Integer> queue = new LinkedList<>();
for (int i = 0; i < n; i++) {
    if (inDegree[i] == 0) queue.offer(i);
}
List<Integer> order = new ArrayList<>();
while (!queue.isEmpty()) {
    int node = queue.poll();
    order.add(node);
    for (int neighbor : graph[node]) {
        if (--inDegree[neighbor] == 0) queue.offer(neighbor);
    }
}
```
**Classic problems:** Course Schedule, Course Schedule II

### 14. Union-Find
**When:** Group connectivity, detect cycles in undirected graph, number of components
**Template:**
```java
int[] parent, rank;

int find(int x) {
    if (parent[x] != x) parent[x] = find(parent[x]); // path compression
    return parent[x];
}

void union(int x, int y) {
    int px = find(x), py = find(y);
    if (px == py) return;
    if (rank[px] < rank[py]) parent[px] = py;
    else if (rank[px] > rank[py]) parent[py] = px;
    else { parent[py] = px; rank[px]++; }
}
```
**Classic problems:** Number of Connected Components, Redundant Connection

### 15. Trie (Prefix Tree)
**When:** Prefix-based search, autocomplete, word dictionary
**Template:**
```java
class TrieNode {
    TrieNode[] children = new TrieNode[26];
    boolean isEnd;
}

void insert(String word) {
    TrieNode node = root;
    for (char c : word.toCharArray()) {
        if (node.children[c - 'a'] == null)
            node.children[c - 'a'] = new TrieNode();
        node = node.children[c - 'a'];
    }
    node.isEnd = true;
}
```
**Classic problems:** Implement Trie, Word Search II, Design Add and Search Words

---

## Time Complexity Quick Reference

| Pattern | Typical Time | Typical Space |
|---------|-------------|---------------|
| Two Pointers | O(n) | O(1) |
| Sliding Window | O(n) | O(k) |
| Binary Search | O(log n) | O(1) |
| HashMap | O(n) | O(n) |
| Stack | O(n) | O(n) |
| BFS/DFS | O(V + E) | O(V) |
| Backtracking | O(2^n) or O(n!) | O(n) |
| DP (1D) | O(n) | O(n) |
| DP (2D) | O(n*m) | O(n*m) |
| Heap operations | O(n log k) | O(k) |
| Trie | O(L) per word | O(TOTAL_CHARS) |
| Union-Find | O(α(n)) ≈ O(1) | O(n) |
| Topological Sort | O(V + E) | O(V) |

---

## Interview Day Quick Review

Before walking into a coding interview, review this:

1. **Read carefully** — understand what's being asked before coding
2. **Ask clarifying questions** — input size, edge cases, constraints
3. **Think patterns first** — use the decision tree above
4. **Talk through your approach** before writing code
5. **Start with brute force** — then optimize
6. **Write clean code** — good variable names, helper functions
7. **Test with examples** — walk through your code with the given example
8. **Analyze complexity** — time and space, mention it proactively
