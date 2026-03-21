# Pattern: DFS Traversal (Graphs & Grids)

## How to Identify This Pattern

### Trigger Words
- "connected components", "number of islands"
- "explore all paths", "all paths from source to target"
- "flood fill", "connected region"
- "surrounded", "enclosed", "boundary"
- "count", "area of region"
- "reachability", "can reach"

### Constraint Clues
- Need to explore all nodes reachable from a source (not necessarily shortest path)
- Counting connected components or measuring component properties (area, perimeter)
- Checking reachability or connectivity between nodes
- Problem involves exploring/marking entire regions in a grid
- Recursion depth is manageable (grid ≤ 200×200, graph ≤ 10^4 nodes)

### When to Use This vs Similar Patterns
| Consideration | DFS | BFS | Union-Find |
|---|---|---|---|
| Shortest path? | No guarantee | Yes (unweighted) | No |
| Best for | Connectivity, paths, components | Shortest path, level-order | Dynamic connectivity queries |
| Space | O(V) stack depth | O(V) queue width | O(V) parent array |
| Implementation | Recursive (simple) or iterative | Iterative with queue | Array-based |

## Core Approach

### DFS on a Grid (Island-style)
1. Iterate over every cell in the grid.
2. When you find an unvisited land cell, start a DFS from it.
3. In DFS, mark the current cell as visited (change its value or use a visited array).
4. Recurse into all valid neighbors (4-directional or 8-directional).
5. Each DFS call from the main loop discovers one connected component.

### DFS on an Adjacency List
1. Maintain a `boolean[] visited` array.
2. For each unvisited node, call DFS.
3. In DFS, mark the node visited, then recurse into all unvisited neighbors.
4. Track any required information (path, component size, etc.) during recursion.

### DFS for All Paths
1. Use backtracking: add the current node to the path before recursing, remove it after.
2. When you reach the target, record a copy of the current path.
3. No visited set is needed if the graph is a DAG; otherwise use visited to avoid cycles.

## Java Template

### DFS on a Grid (Number of Islands)
```java
public int numIslands(char[][] grid) {
    int count = 0;
    for (int i = 0; i < grid.length; i++) {
        for (int j = 0; j < grid[0].length; j++) {
            if (grid[i][j] == '1') {
                dfs(grid, i, j);
                count++;
            }
        }
    }
    return count;
}

private void dfs(char[][] grid, int r, int c) {
    if (r < 0 || r >= grid.length || c < 0 || c >= grid[0].length
            || grid[r][c] != '1') {
        return;
    }
    grid[r][c] = '0'; // mark visited
    dfs(grid, r + 1, c);
    dfs(grid, r - 1, c);
    dfs(grid, r, c + 1);
    dfs(grid, r, c - 1);
}
```

### DFS on an Adjacency List (Clone Graph)
```java
public Node cloneGraph(Node node) {
    if (node == null) return null;
    Map<Node, Node> cloned = new HashMap<>();
    return dfs(node, cloned);
}

private Node dfs(Node node, Map<Node, Node> cloned) {
    if (cloned.containsKey(node)) return cloned.get(node);

    Node copy = new Node(node.val);
    cloned.put(node, copy);
    for (Node neighbor : node.neighbors) {
        copy.neighbors.add(dfs(neighbor, cloned));
    }
    return copy;
}
```

### DFS for All Paths (Source to Target in DAG)
```java
public List<List<Integer>> allPathsSourceTarget(int[][] graph) {
    List<List<Integer>> result = new ArrayList<>();
    List<Integer> path = new ArrayList<>();
    path.add(0);
    dfs(graph, 0, path, result);
    return result;
}

private void dfs(int[][] graph, int node, List<Integer> path,
                 List<List<Integer>> result) {
    if (node == graph.length - 1) {
        result.add(new ArrayList<>(path));
        return;
    }
    for (int next : graph[node]) {
        path.add(next);
        dfs(graph, next, path, result);
        path.remove(path.size() - 1);
    }
}
```

## Dry Run Example

**Problem:** Max Area of Island — find the largest island (connected 1s) in a grid.

Grid:
```
0  0  1  0  0
0  1  1  1  0
0  0  1  0  0
```

| Step | Cell | Action | Area So Far | Visited Cells |
|------|------|--------|-------------|---------------|
| 1 | (0,0) | grid=0, skip | — | — |
| 2 | (0,2) | grid=1, start DFS | 1 | (0,2) |
| 3 | → (1,2) | grid=1, continue DFS | 2 | (0,2),(1,2) |
| 4 | → → (2,2) | grid=1, continue DFS | 3 | (0,2),(1,2),(2,2) |
| 5 | → → → dead ends | all neighbors 0 or OOB | 3 | backtrack |
| 6 | → (1,1) | grid=1, continue DFS | 4 | + (1,1) |
| 7 | → → dead ends | all neighbors visited or 0 | 4 | backtrack |
| 8 | → (1,3) | grid=1, continue DFS | 5 | + (1,3) |
| 9 | → → dead ends | all neighbors 0 or OOB | 5 | backtrack |

**Result:** Maximum area = 5. ✓

## Complexity
- **Time:** O(V + E) for graph DFS; O(M × N) for grid DFS
- **Space:** O(V) for recursion stack and visited set; O(M × N) for grid DFS in the worst case (long snake-shaped island causes deep recursion)

## Edge Cases
- Empty grid or graph — return 0
- Grid with all water (no land) — return 0 for island problems
- Grid with all land (one giant island) — recursion depth = M × N, may cause stack overflow for very large grids; consider iterative DFS
- Single node graph — the node itself is the answer
- Disconnected graph — must iterate over all nodes to find all components
- Boundary cells in "Surrounded Regions" — regions touching the border are NOT surrounded

## Common Mistakes
- **Stack overflow on large grids:** Java's default stack size can overflow on grids ≥ 500×500. Use iterative DFS with an explicit stack or increase stack size.
- **Forgetting to mark visited before recursing:** This causes infinite loops in graphs with cycles.
- **Modifying input when you shouldn't:** Some problems expect the grid to remain unchanged; use a separate visited array.
- **Not copying the path in "all paths" problems:** Storing a reference to the path list means all entries share the same list. Always do `new ArrayList<>(path)`.
- **Confusing grid[r][c] = '0' vs grid[r][c] = 0:** Character grids use `'0'`/`'1'`, integer grids use `0`/`1`.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Clone Graph | Medium | DFS with a hashmap to track original→clone mapping; handles cycles via memoization | https://leetcode.com/problems/clone-graph/ |
| 2 | Number of Islands | Medium | DFS from each unvisited '1' cell; sink the island by marking cells '0' | https://leetcode.com/problems/number-of-islands/ |
| 3 | Surrounded Regions | Medium | DFS from border 'O' cells to mark safe zones; flip remaining 'O' to 'X' | https://leetcode.com/problems/surrounded-regions/ |
| 4 | Max Area of Island | Medium | DFS returns area count for each island; track the global maximum | https://leetcode.com/problems/max-area-of-island/ |
| 5 | Number of Enclaves | Medium | DFS from border land cells to mark reachable land; count remaining land cells | https://leetcode.com/problems/number-of-enclaves/ |
| 6 | Keys and Rooms | Medium | DFS from room 0 using keys as edges; check if all rooms are visited | https://leetcode.com/problems/keys-and-rooms/ |
| 7 | All Paths From Source to Target | Medium | DFS with backtracking on a DAG; no visited set needed since DAG has no cycles | https://leetcode.com/problems/all-paths-from-source-to-target/ |
| 8 | Number of Closed Islands | Medium | DFS on land cells; an island is closed if no cell touches the grid boundary | https://leetcode.com/problems/number-of-closed-islands/ |
| 9 | Count Sub Islands | Medium | DFS on grid2 islands; a sub-island must have every land cell also be land in grid1 | https://leetcode.com/problems/count-sub-islands/ |
| 10 | Island Perimeter | Easy | Count edges: each land cell contributes 4 minus number of land neighbors | https://leetcode.com/problems/island-perimeter/ |
