# Pattern: Cycle Detection

## How to Identify This Pattern

### Trigger Words
- "cycle", "circular dependency"
- "deadlock detection"
- "safe states", "eventually safe"
- "redundant edge" (edge that creates a cycle)
- "valid tree" (a tree has no cycles)
- "bipartite" (odd-length cycles prevent bipartiteness)

### Constraint Clues
- Need to determine if a cycle exists in a directed or undirected graph
- Need to find all nodes that are part of or lead to a cycle
- Problem asks if a graph is a valid tree (connected + no cycles)
- Need to identify which specific edge creates the cycle
- Functional graphs (each node has exactly one outgoing edge)

### When to Use This vs Similar Patterns
| Consideration | DFS Coloring (Directed) | Union-Find (Undirected) | Topological Sort | Floyd's (Linked List) |
|---|---|---|---|---|
| Graph type | Directed | Undirected | Directed (DAG check) | Functional graph |
| Finds cycle? | Yes | Yes | Yes (indirectly) | Yes |
| Finds cycle nodes? | Via "safe states" | The redundant edge | Unprocessed nodes | Meeting point |
| Complexity | O(V + E) | O(V × α(V)) | O(V + E) | O(V) |

## Core Approach

### Directed Graph — DFS with 3-Color States
1. Assign each node one of three states: `WHITE` (unvisited), `GRAY` (in current DFS path), `BLACK` (fully processed).
2. Start DFS from each WHITE node.
3. When entering a node, color it GRAY.
4. If a neighbor is GRAY → **back edge → cycle found**.
5. If a neighbor is BLACK → already processed, skip.
6. If a neighbor is WHITE → recurse.
7. After processing all neighbors, color the node BLACK.

### Undirected Graph — DFS with Parent Tracking
1. Standard DFS but track the parent of each node.
2. If a neighbor is visited **and is not the parent** → **cycle found**.
3. Alternative: Use Union-Find — if `find(u) == find(v)` before `union(u, v)`, adding edge (u,v) creates a cycle.

### Functional Graph — Floyd's Cycle Detection
1. In a functional graph (each node has exactly one outgoing edge), use slow/fast pointers.
2. Slow moves one step, fast moves two steps.
3. If they meet, a cycle exists. The meeting point helps find the cycle entry and length.

## Java Template

### Directed Graph — DFS 3-Color
```java
public boolean hasCycleDirected(int numNodes, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < numNodes; i++) graph.add(new ArrayList<>());
    for (int[] e : edges) graph.get(e[0]).add(e[1]);

    int[] color = new int[numNodes]; // 0=white, 1=gray, 2=black

    for (int i = 0; i < numNodes; i++) {
        if (color[i] == 0 && dfs(graph, i, color)) {
            return true; // cycle found
        }
    }
    return false;
}

private boolean dfs(List<List<Integer>> graph, int u, int[] color) {
    color[u] = 1; // gray — in current path
    for (int v : graph.get(u)) {
        if (color[v] == 1) return true;  // back edge → cycle
        if (color[v] == 0 && dfs(graph, v, color)) return true;
    }
    color[u] = 2; // black — fully processed
    return false;
}
```

### Find Eventual Safe States
```java
public List<Integer> eventualSafeNodes(int[][] graph) {
    int n = graph.length;
    int[] color = new int[n]; // 0=white, 1=gray, 2=black

    List<Integer> result = new ArrayList<>();
    for (int i = 0; i < n; i++) {
        if (!hasCycle(graph, i, color)) {
            result.add(i);
        }
    }
    return result;
}

private boolean hasCycle(int[][] graph, int u, int[] color) {
    if (color[u] != 0) return color[u] == 1; // gray means part of cycle
    color[u] = 1;
    for (int v : graph[u]) {
        if (hasCycle(graph, v, color)) return true;
    }
    color[u] = 2; // safe — no cycle reachable
    return false;
}
```

### Undirected Graph — DFS with Parent
```java
public boolean hasCycleUndirected(int numNodes, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < numNodes; i++) graph.add(new ArrayList<>());
    for (int[] e : edges) {
        graph.get(e[0]).add(e[1]);
        graph.get(e[1]).add(e[0]);
    }

    boolean[] visited = new boolean[numNodes];
    for (int i = 0; i < numNodes; i++) {
        if (!visited[i] && dfsUndirected(graph, i, -1, visited)) {
            return true;
        }
    }
    return false;
}

private boolean dfsUndirected(List<List<Integer>> graph, int u, int parent,
                               boolean[] visited) {
    visited[u] = true;
    for (int v : graph.get(u)) {
        if (!visited[v]) {
            if (dfsUndirected(graph, v, u, visited)) return true;
        } else if (v != parent) {
            return true; // visited neighbor that isn't parent → cycle
        }
    }
    return false;
}
```

## Dry Run Example

**Problem:** Detect cycle in directed graph with edges: `0→1, 1→2, 2→0, 2→3`.

| Step | Node | Color Before | Neighbors | Action | Color After |
|------|------|-------------|-----------|--------|-------------|
| 1 | 0 | WHITE | [1] | Mark GRAY, recurse to 1 | GRAY |
| 2 | 1 | WHITE | [2] | Mark GRAY, recurse to 2 | GRAY |
| 3 | 2 | WHITE | [0, 3] | Mark GRAY, check neighbor 0 | GRAY |
| 4 | — | — | 0 is GRAY! | **Back edge 2→0 → CYCLE** | — |

**Result:** Cycle detected: 0 → 1 → 2 → 0. ✓

**Problem:** Find Eventual Safe States — `graph = [[1,2],[2,3],[5],[0],[5],[],[]]` (7 nodes).

- Nodes 0→{1,2}, 1→{2,3}, 2→{5}, 3→{0}, 4→{5}, 5→{}, 6→{}.
- DFS from 0: 0(G)→1(G)→3(G)→0(G) = cycle! Nodes 0,1,3 unsafe.
- Node 2: 2(G)→5(G)→done(B), 2→B. Safe.
- Node 4: 4(G)→5(B)→done, 4→B. Safe.
- Nodes 5, 6: terminal → safe.

**Result:** Safe nodes = [2, 4, 5, 6]. ✓

## Complexity
- **Time:** O(V + E) — each node and edge visited at most once
- **Space:** O(V) for color/visited array + O(V) recursion stack

## Edge Cases
- Self-loop — immediate cycle (node points to itself)
- Single node with no edges — no cycle
- Disconnected graph — must check all components
- Parallel edges in undirected graph — with parent tracking, parallel edges count as a cycle
- Node with all outgoing edges (terminal-like) — no cycle from that node
- Graph is a tree — exactly V-1 edges, connected, no cycles

## Common Mistakes
- **Using 2-color (visited/not) for directed graphs:** In directed graphs, a visited node may not be on the current path. You need 3 colors (WHITE/GRAY/BLACK) to distinguish "on current path" from "fully processed."
- **Forgetting parent check in undirected graphs:** In undirected DFS, the edge back to the parent is not a cycle. You must exclude the parent when checking for visited neighbors.
- **Confusing directed and undirected cycle detection:** Different algorithms are needed. A back edge in a directed DFS means a cycle, but in undirected, any visited non-parent neighbor is a cycle.
- **Not iterating over all nodes:** In disconnected graphs, a cycle might be in an unreachable component from node 0.
- **Redundant Connection II (directed):** Requires handling two cases: a node with in-degree 2, or a cycle without in-degree 2. Much harder than the undirected version.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Course Schedule | Medium | Cycle in prerequisite graph means impossible to finish; use Kahn's or DFS coloring | https://leetcode.com/problems/course-schedule/ |
| 2 | Find Eventual Safe States | Medium | Nodes NOT part of or leading to a cycle are safe; DFS 3-color to classify | https://leetcode.com/problems/find-eventual-safe-states/ |
| 3 | Is Graph Bipartite? | Medium | Odd-length cycle prevents bipartiteness; 2-color BFS/DFS to check | https://leetcode.com/problems/is-graph-bipartite/ |
| 4 | Redundant Connection | Medium | Union-Find on undirected graph; the edge causing a cycle is redundant | https://leetcode.com/problems/redundant-connection/ |
| 5 | Redundant Connection II | Hard | Directed graph: handle node with in-degree 2 and/or cycle; try removing candidate edges | https://leetcode.com/problems/redundant-connection-ii/ |
| 6 | Longest Cycle in a Graph | Hard | Functional graph (one outgoing edge per node); DFS with timestamps to measure cycle length | https://leetcode.com/problems/longest-cycle-in-a-graph/ |
| 7 | Detect Cycles in 2D Grid | Medium | DFS on grid cells with same character; track parent to detect cycle of length ≥ 4 | https://leetcode.com/problems/detect-cycles-in-2d-grid/ |
| 8 | Graph Valid Tree | Medium | A valid tree has exactly n-1 edges, is connected, and has no cycle; Union-Find or DFS | https://leetcode.com/problems/graph-valid-tree/ |
| 9 | Circular Array Loop | Medium | Detect cycle in a functional graph where each index points to another; same-direction constraint | https://leetcode.com/problems/circular-array-loop/ |
| 10 | Find the Town Judge | Easy | The judge has in-degree n-1 and out-degree 0; scan trust relationships | https://leetcode.com/problems/find-the-town-judge/ |
