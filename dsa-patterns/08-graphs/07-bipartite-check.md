# Pattern: Bipartite Check (Graph 2-Coloring)

## How to Identify This Pattern

### Trigger Words
- "bipartite", "two groups", "two sets"
- "divide into two parties/teams"
- "coloring", "two colors"
- "odd cycle" (bipartite ↔ no odd-length cycles)
- "matching", "pairing elements from two groups"
- "possible bipartition", "can we split"

### Constraint Clues
- Need to check if a graph can be split into two disjoint sets where every edge goes between the sets
- Conflict/dislike relationships between elements that should be in different groups
- Need to assign one of two labels to every node such that no two adjacent nodes share the same label
- Graph coloring with exactly 2 colors
- Problems about even/odd-length cycles

### When to Use This vs Similar Patterns
| Consideration | Bipartite Check | Cycle Detection | Graph Coloring (k>2) | Union-Find |
|---|---|---|---|---|
| Colors/groups | Exactly 2 | N/A | k colors | Arbitrary groups |
| Technique | BFS/DFS 2-coloring | DFS 3-state | Backtracking (NP-hard for k≥3) | Merge-based |
| Key insight | No odd cycles | Back edge = cycle | NP-complete for k≥3 | Connectivity |
| Complexity | O(V + E) | O(V + E) | Exponential for k≥3 | O(V × α(V)) |

## Core Approach

### BFS 2-Coloring
1. Initialize a `color[]` array with -1 (uncolored) for all nodes.
2. For each uncolored node (handles disconnected graphs):
   a. Color it 0 and add to a queue.
   b. While the queue is not empty:
      - Dequeue node `u`.
      - For each neighbor `v`:
        - If uncolored: color it `1 - color[u]` and enqueue.
        - If same color as `u`: **not bipartite** → return false.
3. If all nodes are colored without conflict → **bipartite**.

### DFS 2-Coloring
1. Same color array initialization.
2. For each uncolored node, start DFS with color 0.
3. In DFS, color the current node. For each neighbor:
   - If uncolored: recurse with the opposite color.
   - If same color: **not bipartite**.

### Bipartite + BFS for Group Assignment
For problems like "Divide Nodes Into Maximum Number of Groups":
1. First check if the graph is bipartite.
2. For bipartite components, BFS from every node to find the maximum number of levels (layers).
3. The answer for each component is the maximum BFS depth.

## Java Template

### BFS 2-Coloring
```java
public boolean isBipartite(int[][] graph) {
    int n = graph.length;
    int[] color = new int[n];
    Arrays.fill(color, -1);

    for (int i = 0; i < n; i++) {
        if (color[i] != -1) continue;

        Queue<Integer> queue = new LinkedList<>();
        queue.offer(i);
        color[i] = 0;

        while (!queue.isEmpty()) {
            int u = queue.poll();
            for (int v : graph[u]) {
                if (color[v] == -1) {
                    color[v] = 1 - color[u];
                    queue.offer(v);
                } else if (color[v] == color[u]) {
                    return false;
                }
            }
        }
    }
    return true;
}
```

### DFS 2-Coloring
```java
public boolean isBipartiteDFS(int[][] graph) {
    int n = graph.length;
    int[] color = new int[n];
    Arrays.fill(color, -1);

    for (int i = 0; i < n; i++) {
        if (color[i] == -1 && !dfs(graph, i, 0, color)) {
            return false;
        }
    }
    return true;
}

private boolean dfs(int[][] graph, int u, int c, int[] color) {
    color[u] = c;
    for (int v : graph[u]) {
        if (color[v] == -1) {
            if (!dfs(graph, v, 1 - c, color)) return false;
        } else if (color[v] == c) {
            return false;
        }
    }
    return true;
}
```

### Possible Bipartition (Dislike Groups)
```java
public boolean possibleBipartition(int n, int[][] dislikes) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i <= n; i++) graph.add(new ArrayList<>());
    for (int[] d : dislikes) {
        graph.get(d[0]).add(d[1]);
        graph.get(d[1]).add(d[0]);
    }

    int[] color = new int[n + 1];
    Arrays.fill(color, -1);

    for (int i = 1; i <= n; i++) {
        if (color[i] != -1) continue;
        Queue<Integer> queue = new LinkedList<>();
        queue.offer(i);
        color[i] = 0;

        while (!queue.isEmpty()) {
            int u = queue.poll();
            for (int v : graph.get(u)) {
                if (color[v] == -1) {
                    color[v] = 1 - color[u];
                    queue.offer(v);
                } else if (color[v] == color[u]) {
                    return false;
                }
            }
        }
    }
    return true;
}
```

## Dry Run Example

**Problem:** Is Graph Bipartite? — `graph = [[1,3],[0,2],[1,3],[0,2]]` (4 nodes, 0-indexed).

Edges: 0—1, 0—3, 1—2, 2—3.

| Step | Queue | Dequeue | Color Neighbor | color[] | Conflict? |
|------|-------|---------|----------------|---------|-----------|
| Init | [0] | — | color[0]=0 | [0,-1,-1,-1] | — |
| 1 | [] | 0 | 1→color 1, 3→color 1 | [0,1,-1,1] | No |
| 2 | [1,3] | 1 | 0→already 0 (≠1 ✓), 2→color 0 | [0,1,0,1] | No |
| 3 | [3,2] | 3 | 0→already 0 (≠1 ✓), 2→already 0 (≠1 ✓) | [0,1,0,1] | No |
| 4 | [2] | 2 | 1→already 1 (≠0 ✓), 3→already 1 (≠0 ✓) | [0,1,0,1] | No |

Groups: {0, 2} and {1, 3}. No conflicts.
**Result:** Bipartite = true. ✓

**Counter-example:** `graph = [[1,2,3],[0,2],[0,1,3],[0,2]]` — edge 1-2 exists.

| Step | Dequeue | Action | color[] | Conflict? |
|------|---------|--------|---------|-----------|
| Init | — | color[0]=0 | [0,-1,-1,-1] | — |
| 1 | 0 | 1→1, 2→1, 3→1 | [0,1,1,1] | No |
| 2 | 1 | check 2: color[2]=1 == color[1]=1 | — | **YES!** |

**Result:** Not bipartite (odd cycle 0-1-2-0). ✓

## Complexity
- **Time:** O(V + E) — standard BFS/DFS traversal
- **Space:** O(V) for the color array and queue/recursion stack

## Edge Cases
- Disconnected graph — must check each component independently; graph can be bipartite overall even with multiple components
- Single node — trivially bipartite
- No edges — every node is its own component; bipartite
- Self-loop — immediately not bipartite (node must be both colors)
- Odd-length cycle — not bipartite
- Even-length cycle — bipartite
- Complete graph K3 or larger — not bipartite (triangle is an odd cycle)

## Common Mistakes
- **Only checking one component:** Disconnected graphs require checking every component. Always loop over all nodes.
- **Confusing "not bipartite" with "has a cycle":** Even cycles are fine in bipartite graphs. Only odd cycles break bipartiteness.
- **Using visited instead of color:** A simple `visited` boolean is not enough. You need the actual color (0 or 1) to verify consistency.
- **1-indexed vs 0-indexed:** Possible Bipartition uses 1-indexed nodes. Size arrays accordingly.
- **Treating the problem as directed:** Bipartite check is for undirected graphs. Always add edges in both directions.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Is Graph Bipartite? | Medium | BFS/DFS 2-coloring; if any neighbor has the same color, return false | https://leetcode.com/problems/is-graph-bipartite/ |
| 2 | Possible Bipartition | Medium | Model dislikes as edges; check if the dislike graph is bipartite | https://leetcode.com/problems/possible-bipartition/ |
| 3 | Divide Nodes Into the Maximum Number of Groups | Hard | Check bipartiteness per component; BFS from every node to find max levels per component | https://leetcode.com/problems/divide-nodes-into-the-maximum-number-of-groups/ |
| 4 | Shortest Cycle in a Graph | Hard | BFS from every node; track the shortest cycle length found | https://leetcode.com/problems/shortest-cycle-in-a-graph/ |
| 5 | Flower Planting With No Adjacent | Medium | Greedy coloring with 4 colors; for each node pick any color not used by neighbors | https://leetcode.com/problems/flower-planting-with-no-adjacent/ |
| 6 | Coloring A Border | Medium | BFS/DFS to identify border cells of a connected component; recolor them | https://leetcode.com/problems/coloring-a-border/ |
| 7 | Check if There is a Valid Path in a Grid | Medium | Model cells as nodes with connections based on street types; BFS/DFS or Union-Find for connectivity | https://leetcode.com/problems/check-if-there-is-a-valid-path-in-a-grid/ |
| 8 | The Maze | Medium | BFS where the ball rolls until hitting a wall; each state is a stopping position | https://leetcode.com/problems/the-maze/ |
| 9 | Odd-Even Jump | Hard | DP with sorted map; determine if jumps from each index can reach the end | https://leetcode.com/problems/odd-even-jump/ |
| 10 | Find if Path Exists in Graph | Easy | Simple BFS/DFS or Union-Find to check connectivity between source and destination | https://leetcode.com/problems/find-if-path-exists-in-graph/ |
