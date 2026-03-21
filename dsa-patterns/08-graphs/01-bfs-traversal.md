# Pattern: BFS Traversal (Graphs & Grids)

## How to Identify This Pattern

### Trigger Words
- "shortest path" (in unweighted graph or grid)
- "minimum number of steps/moves/operations"
- "level-by-level", "layer by layer"
- "nearest", "closest", "minimum distance"
- "spread", "expand", "propagate" (like rotting, fire, infection)
- "reachable within k steps"

### Constraint Clues
- Grid or graph with uniform edge weights (all edges cost 1)
- Need the **shortest** path, not just any path
- Multi-source propagation (multiple starting points spreading simultaneously)
- Need to process nodes level by level (e.g., tree level-order traversal on a graph)
- Matrix dimensions ≤ 200×200 (BFS on grid is feasible)

### When to Use This vs Similar Patterns
| Consideration | BFS | DFS | Dijkstra |
|---|---|---|---|
| Edge weights | All equal (unweighted) | N/A (connectivity) | Weighted (non-negative) |
| Guarantees shortest path? | Yes (unweighted) | No | Yes (weighted) |
| Data structure | Queue | Stack / recursion | Priority Queue |
| Best for | Shortest path, level-order | Connectivity, cycle detection | Weighted shortest path |

## Core Approach

### Standard BFS (Single Source)
1. Create a queue and add the source node. Mark it as visited.
2. While the queue is not empty:
   a. Dequeue the front node.
   b. Process it (check if it's the target, record distance, etc.).
   c. For each unvisited neighbor, mark it visited and enqueue it.
3. The first time you reach the target, you have the shortest path.

### Multi-Source BFS
1. Add **all** source nodes to the queue simultaneously (distance = 0).
2. Process exactly like standard BFS — the wavefront expands from all sources at once.
3. Each cell's distance is the minimum distance to any source.

### BFS on a Grid
1. Directions array: `{{0,1},{0,-1},{1,0},{-1,0}}` for 4-directional movement.
2. Use a `boolean[][] visited` or modify the grid in-place to avoid revisiting.
3. Each level of the BFS corresponds to one "step" in the grid.

## Java Template

### BFS on a Grid (Shortest Path)
```java
public int bfsGrid(int[][] grid) {
    int rows = grid.length, cols = grid[0].length;
    boolean[][] visited = new boolean[rows][cols];
    Queue<int[]> queue = new LinkedList<>();
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};

    queue.offer(new int[]{0, 0});
    visited[0][0] = true;
    int steps = 0;

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] cell = queue.poll();
            int r = cell[0], c = cell[1];

            if (r == rows - 1 && c == cols - 1) return steps;

            for (int[] d : dirs) {
                int nr = r + d[0], nc = c + d[1];
                if (nr >= 0 && nr < rows && nc >= 0 && nc < cols
                        && !visited[nr][nc] && grid[nr][nc] == 0) {
                    visited[nr][nc] = true;
                    queue.offer(new int[]{nr, nc});
                }
            }
        }
        steps++;
    }
    return -1;
}
```

### BFS on an Adjacency List (Graph)
```java
public int bfsGraph(Map<Integer, List<Integer>> graph, int source, int target) {
    Set<Integer> visited = new HashSet<>();
    Queue<Integer> queue = new LinkedList<>();
    queue.offer(source);
    visited.add(source);
    int level = 0;

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int node = queue.poll();
            if (node == target) return level;
            for (int neighbor : graph.getOrDefault(node, List.of())) {
                if (visited.add(neighbor)) {
                    queue.offer(neighbor);
                }
            }
        }
        level++;
    }
    return -1;
}
```

## Dry Run Example

**Problem:** Rotting Oranges — given a grid where `2` = rotten, `1` = fresh, `0` = empty, find the minimum minutes until all oranges are rotten. Each minute, rotten oranges rot 4-directional neighbors.

Grid:
```
2  1  1
1  1  0
0  1  1
```

| Minute | Queue (rotten cells) | Grid State | Fresh Left |
|--------|---------------------|------------|------------|
| 0 | [(0,0)] | `2 1 1 / 1 1 0 / 0 1 1` | 6 |
| 1 | [(0,1),(1,0)] | `2 2 1 / 2 1 0 / 0 1 1` | 4 |
| 2 | [(0,2),(1,1)] | `2 2 2 / 2 2 0 / 0 1 1` | 2 |
| 3 | [(2,1)] | `2 2 2 / 2 2 0 / 0 2 1` | 1 |
| 4 | [(2,2)] | `2 2 2 / 2 2 0 / 0 2 2` | 0 |

**Result:** 4 minutes. All fresh oranges are now rotten. ✓

## Complexity
- **Time:** O(V + E) for graph BFS; O(M × N) for grid BFS where M×N is the grid size
- **Space:** O(V) for the visited set and queue; O(M × N) for grid BFS

## Edge Cases
- Graph is disconnected — some nodes may never be reached; check if all targets were visited
- Source equals target — return 0 immediately
- Empty grid or graph — return 0 or -1 depending on the problem
- Multi-source BFS with no sources — edge case for "Rotting Oranges" when there are no rotten oranges but fresh ones exist → return -1
- Grid with all blocked cells — no path exists
- Single cell grid — source is the destination

## Common Mistakes
- **Marking visited when dequeuing instead of when enqueuing:** This causes duplicate entries in the queue and TLE. Always mark visited at enqueue time.
- **Forgetting level-size loop:** Without `int size = queue.size()` and looping `size` times, you lose track of BFS levels and can't count steps correctly.
- **Not handling multi-source BFS:** Problems like Rotting Oranges and 01 Matrix require adding ALL sources to the queue before starting. Don't BFS from each source independently.
- **Confusing 4-directional vs 8-directional movement:** Shortest Path in Binary Matrix uses 8 directions; most grid problems use 4.
- **Off-by-one in step counting:** Decide whether the source counts as step 0 or step 1 based on the problem definition.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Number of Islands | Medium | BFS from each unvisited land cell; each BFS call discovers one island | https://leetcode.com/problems/number-of-islands/ |
| 2 | Rotting Oranges | Medium | Multi-source BFS from all rotten oranges; count levels as minutes | https://leetcode.com/problems/rotting-oranges/ |
| 3 | Word Ladder | Hard | Each word is a node; BFS finds shortest transformation sequence | https://leetcode.com/problems/word-ladder/ |
| 4 | 01 Matrix | Medium | Multi-source BFS from all 0-cells; expand outward to fill distances | https://leetcode.com/problems/01-matrix/ |
| 5 | Shortest Path in Binary Matrix | Medium | BFS with 8-directional movement; first arrival at bottom-right is shortest | https://leetcode.com/problems/shortest-path-in-binary-matrix/ |
| 6 | Walls and Gates | Medium | Multi-source BFS from all gates; propagate distances to empty rooms | https://leetcode.com/problems/walls-and-gates/ |
| 7 | Open the Lock | Medium | BFS on state space; each state is a 4-digit combination, neighbors differ by ±1 on one wheel | https://leetcode.com/problems/open-the-lock/ |
| 8 | Minimum Knight Moves | Medium | BFS from origin; exploit symmetry by working in the first quadrant | https://leetcode.com/problems/minimum-knight-moves/ |
| 9 | Pacific Atlantic Water Flow | Medium | Multi-source BFS from ocean borders inward; intersect cells reachable from both oceans | https://leetcode.com/problems/pacific-atlantic-water-flow/ |
| 10 | Flood Fill | Easy | Simple BFS/DFS from starting cell; change color of all connected same-color cells | https://leetcode.com/problems/flood-fill/ |
