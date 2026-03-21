# Pattern: Topological Sort

## How to Identify This Pattern

### Trigger Words
- "order of tasks", "course prerequisites"
- "dependency ordering", "build order"
- "sequence", "scheduling"
- "valid ordering", "can finish all courses"
- "alien dictionary", "ordering of characters"
- "longest path in DAG"

### Constraint Clues
- Graph is **directed** and should be **acyclic** (DAG) for a valid topological order
- Dependencies or prerequisites exist between elements
- Need to determine a valid processing order respecting all dependencies
- Need to detect if a valid ordering is impossible (cycle exists)
- "In-degree" concept applies — nodes with 0 in-degree can be processed first

### When to Use This vs Similar Patterns
| Consideration | Topological Sort (Kahn's BFS) | Topological Sort (DFS) | Regular BFS/DFS |
|---|---|---|---|
| Best for | Ordering + cycle detection | Ordering via post-order | Connectivity, shortest path |
| Detects cycles? | Yes (not all nodes processed) | Yes (back edge = cycle) | Not directly |
| Gives all valid orderings? | One valid order | One valid order | N/A |
| Parallel processing? | Nodes at same level can run in parallel | Not naturally | N/A |

## Core Approach

### Kahn's Algorithm (BFS-based)
1. Compute **in-degree** for every node.
2. Add all nodes with in-degree 0 to a queue.
3. While the queue is not empty:
   a. Dequeue node `u` and add it to the result order.
   b. For each neighbor `v` of `u`, decrement `v`'s in-degree.
   c. If `v`'s in-degree becomes 0, enqueue `v`.
4. If the result contains all nodes → valid topological order.
5. If the result has fewer nodes → **cycle exists** (some nodes never reached in-degree 0).

### DFS-based Topological Sort
1. Maintain a visited array with 3 states: `UNVISITED`, `IN_PROGRESS`, `DONE`.
2. For each unvisited node, start DFS.
3. In DFS:
   a. Mark node as `IN_PROGRESS`.
   b. Recurse into all neighbors. If a neighbor is `IN_PROGRESS` → **cycle detected**.
   c. After all neighbors are processed, mark node as `DONE` and push onto a stack.
4. The stack (reversed) gives the topological order.

## Java Template

### Kahn's Algorithm (BFS)
```java
public int[] topologicalSort(int numNodes, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    int[] inDegree = new int[numNodes];
    for (int i = 0; i < numNodes; i++) graph.add(new ArrayList<>());

    for (int[] e : edges) {
        graph.get(e[0]).add(e[1]);
        inDegree[e[1]]++;
    }

    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numNodes; i++) {
        if (inDegree[i] == 0) queue.offer(i);
    }

    int[] order = new int[numNodes];
    int idx = 0;
    while (!queue.isEmpty()) {
        int u = queue.poll();
        order[idx++] = u;
        for (int v : graph.get(u)) {
            if (--inDegree[v] == 0) {
                queue.offer(v);
            }
        }
    }
    return idx == numNodes ? order : new int[0]; // empty = cycle
}
```

### Course Schedule (Can Finish?)
```java
public boolean canFinish(int numCourses, int[][] prerequisites) {
    List<List<Integer>> graph = new ArrayList<>();
    int[] inDegree = new int[numCourses];
    for (int i = 0; i < numCourses; i++) graph.add(new ArrayList<>());

    for (int[] pre : prerequisites) {
        graph.get(pre[1]).add(pre[0]);
        inDegree[pre[0]]++;
    }

    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) {
        if (inDegree[i] == 0) queue.offer(i);
    }

    int count = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        count++;
        for (int next : graph.get(course)) {
            if (--inDegree[next] == 0) queue.offer(next);
        }
    }
    return count == numCourses;
}
```

### DFS-based (with cycle detection)
```java
public int[] topologicalSortDFS(int numNodes, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < numNodes; i++) graph.add(new ArrayList<>());
    for (int[] e : edges) graph.get(e[0]).add(e[1]);

    int[] state = new int[numNodes]; // 0=unvisited, 1=in-progress, 2=done
    Deque<Integer> stack = new ArrayDeque<>();

    for (int i = 0; i < numNodes; i++) {
        if (state[i] == 0 && hasCycle(graph, i, state, stack)) {
            return new int[0]; // cycle detected
        }
    }

    int[] order = new int[numNodes];
    for (int i = 0; i < numNodes; i++) order[i] = stack.pop();
    return order;
}

private boolean hasCycle(List<List<Integer>> graph, int u,
                         int[] state, Deque<Integer> stack) {
    state[u] = 1;
    for (int v : graph.get(u)) {
        if (state[v] == 1) return true; // back edge = cycle
        if (state[v] == 0 && hasCycle(graph, v, state, stack)) return true;
    }
    state[u] = 2;
    stack.push(u);
    return false;
}
```

## Dry Run Example

**Problem:** Course Schedule II — `numCourses = 4`, prerequisites = `[[1,0],[2,0],[3,1],[3,2]]`.

Graph: 0→1, 0→2, 1→3, 2→3.

| Step | Queue | Dequeue | Update In-degrees | Order So Far |
|------|-------|---------|-------------------|-------------|
| Init | [0] | — | inDeg = [0,1,1,2] | [] |
| 1 | [] | 0 | inDeg[1]→0, inDeg[2]→0 | [0] |
| 2 | [1,2] | 1 | inDeg[3]→1 | [0, 1] |
| 3 | [2] | 2 | inDeg[3]→0 | [0, 1, 2] |
| 4 | [3] | 3 | — | [0, 1, 2, 3] |

4 nodes processed == numCourses → valid order.
**Result:** `[0, 1, 2, 3]` ✓

## Complexity
- **Time:** O(V + E) — each node and edge is processed once
- **Space:** O(V + E) — adjacency list + in-degree array + queue

## Edge Cases
- No edges — all nodes have in-degree 0; any permutation is a valid order
- Single node — trivially ordered
- Cycle in the graph — topological sort is impossible; return empty or false
- Disconnected graph — multiple components; Kahn's still works because all in-degree 0 nodes are enqueued
- Self-loop — immediate cycle
- Multiple valid orderings — Kahn's with a regular queue gives one valid order; use a min-heap for lexicographically smallest

## Common Mistakes
- **Reversing prerequisite direction:** In `[a, b]` meaning "b is prerequisite of a", the edge goes from `b → a`. Getting this backwards breaks the algorithm.
- **Forgetting to check for cycles:** If the result has fewer nodes than expected, a cycle exists. Don't return a partial order.
- **Using topological sort on undirected graphs:** Topological sort is only defined for directed graphs.
- **Not building the graph correctly for Alien Dictionary:** Each pair of adjacent words gives at most one edge. Compare character by character until the first difference.
- **Missing the "prefix" edge case in Alien Dictionary:** If word1 is a prefix of word2 and word1 comes after word2, the ordering is invalid.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Course Schedule | Medium | Kahn's BFS; if all courses processed → no cycle → can finish | https://leetcode.com/problems/course-schedule/ |
| 2 | Course Schedule II | Medium | Same as above but return the actual topological order | https://leetcode.com/problems/course-schedule-ii/ |
| 3 | Alien Dictionary | Hard | Build a graph from adjacent word pairs; topological sort gives character order | https://leetcode.com/problems/alien-dictionary/ |
| 4 | Minimum Height Trees | Medium | Iteratively remove leaf nodes (in-degree 1); remaining 1-2 nodes are roots of MHTs | https://leetcode.com/problems/minimum-height-trees/ |
| 5 | Parallel Courses | Medium | Kahn's BFS tracking levels; answer is the number of BFS levels (critical path length) | https://leetcode.com/problems/parallel-courses/ |
| 6 | Find All Possible Recipes from Given Supplies | Medium | Model recipes and ingredients as a dependency graph; topological sort to resolve which recipes can be made | https://leetcode.com/problems/find-all-possible-recipes-from-given-supplies/ |
| 7 | Longest Increasing Path in a Matrix | Hard | Each cell depends on smaller neighbors; DFS with memoization is equivalent to topological sort on the DAG | https://leetcode.com/problems/longest-increasing-path-in-a-matrix/ |
| 8 | Sort Items by Groups Respecting Dependencies | Hard | Two-level topological sort: first sort groups, then sort items within each group | https://leetcode.com/problems/sort-items-by-groups-respecting-dependencies/ |
| 9 | Sequence Reconstruction | Medium | Check if topological order is unique (queue never has more than 1 element) and matches the given sequence | https://leetcode.com/problems/sequence-reconstruction/ |
| 10 | Build a Matrix With Conditions | Hard | Two independent topological sorts (row order and column order); place values accordingly | https://leetcode.com/problems/build-a-matrix-with-conditions/ |
