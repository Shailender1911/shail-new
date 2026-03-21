# Pattern: Shortest Path — Dijkstra & Bellman-Ford

## How to Identify This Pattern

### Trigger Words
- "minimum cost", "minimum weight path"
- "shortest path in weighted graph"
- "cheapest", "least expensive"
- "network delay", "time for signal to reach all nodes"
- "at most K stops/edges"
- "maximum probability" (inverted to minimum via log or product)

### Constraint Clues
- Graph has **weighted edges** (non-uniform costs)
- Need the shortest/cheapest path between two nodes
- Constraints mention non-negative weights → Dijkstra; negative weights allowed → Bellman-Ford
- "At most K edges" → modified Bellman-Ford or BFS with state (node, stops)
- Graph given as edge list with weights

### When to Use This vs Similar Patterns
| Consideration | Dijkstra | Bellman-Ford | BFS (unweighted) |
|---|---|---|---|
| Edge weights | Non-negative | Any (handles negative) | All equal (weight 1) |
| Negative cycles? | Cannot detect | Detects them | N/A |
| Time complexity | O((V+E) log V) | O(V × E) | O(V + E) |
| Data structure | Min-heap (PriorityQueue) | Relax all edges V-1 times | Queue |
| "At most K stops" | Modified with state | Natural (run K iterations) | Natural |

## Core Approach

### Dijkstra's Algorithm
1. Initialize `dist[]` with ∞ for all nodes, `dist[source] = 0`.
2. Add `(0, source)` to a min-heap (priority queue).
3. While the heap is not empty:
   a. Extract `(cost, u)` with minimum cost.
   b. If `cost > dist[u]`, skip (already found a better path).
   c. For each neighbor `(v, weight)` of `u`:
      - If `dist[u] + weight < dist[v]`, update `dist[v]` and push `(dist[v], v)`.
4. `dist[target]` is the shortest path. If still ∞, no path exists.

### Bellman-Ford Algorithm
1. Initialize `dist[]` with ∞ for all nodes, `dist[source] = 0`.
2. Repeat `V - 1` times:
   a. For every edge `(u, v, weight)`:
      - If `dist[u] + weight < dist[v]`, update `dist[v] = dist[u] + weight`.
3. (Optional) Run one more iteration to detect negative cycles — if any distance decreases, a negative cycle exists.

### Modified Bellman-Ford (K stops)
1. Same as Bellman-Ford but run only `K + 1` iterations instead of `V - 1`.
2. Use a copy of the distance array from the previous iteration to prevent "chain relaxations" within a single round.

## Java Template

### Dijkstra (Adjacency List)
```java
public int[] dijkstra(Map<Integer, List<int[]>> graph, int n, int source) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[source] = 0;

    // {cost, node}
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, source});

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int cost = curr[0], u = curr[1];

        if (cost > dist[u]) continue;

        for (int[] edge : graph.getOrDefault(u, List.of())) {
            int v = edge[0], w = edge[1];
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.offer(new int[]{dist[v], v});
            }
        }
    }
    return dist;
}
```

### Bellman-Ford (Edge List)
```java
public int[] bellmanFord(int[][] edges, int n, int source) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[source] = 0;

    for (int i = 0; i < n - 1; i++) {
        for (int[] e : edges) {
            int u = e[0], v = e[1], w = e[2];
            if (dist[u] != Integer.MAX_VALUE && dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
            }
        }
    }
    return dist;
}
```

### Dijkstra on Grid (Path with Minimum Effort)
```java
public int minimumEffortPath(int[][] heights) {
    int rows = heights.length, cols = heights[0].length;
    int[][] effort = new int[rows][cols];
    for (int[] row : effort) Arrays.fill(row, Integer.MAX_VALUE);
    effort[0][0] = 0;

    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, 0, 0}); // {effort, row, col}
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int e = curr[0], r = curr[1], c = curr[2];

        if (r == rows - 1 && c == cols - 1) return e;
        if (e > effort[r][c]) continue;

        for (int[] d : dirs) {
            int nr = r + d[0], nc = c + d[1];
            if (nr >= 0 && nr < rows && nc >= 0 && nc < cols) {
                int newEffort = Math.max(e, Math.abs(heights[nr][nc] - heights[r][c]));
                if (newEffort < effort[nr][nc]) {
                    effort[nr][nc] = newEffort;
                    pq.offer(new int[]{newEffort, nr, nc});
                }
            }
        }
    }
    return 0;
}
```

## Dry Run Example

**Problem:** Network Delay Time — find the time for a signal to reach all nodes from source node `2`.

Edges: `[[2,1,1],[2,3,1],[3,4,1]]`, n = 4, source = 2.

| Step | PQ State | Extract | Process Neighbors | dist[] |
|------|----------|---------|-------------------|--------|
| Init | [(0,2)] | — | — | [∞, ∞, 0, ∞] |
| 1 | [] | (0,2) | →1 cost=1, →3 cost=1 | [∞, 1, 0, 1] |
| 2 | [(1,1),(1,3)] | (1,1) | no outgoing edges | [∞, 1, 0, 1] |
| 3 | [(1,3)] | (1,3) | →4 cost=2 | [∞, 1, 0, 1, 2] |
| 4 | [(2,4)] | (2,4) | no outgoing edges | [∞, 1, 0, 1, 2] |

dist = [∞, 1, 0, 1, 2] → max (ignoring index 0) = 2.
**Result:** Network delay time = 2. ✓

## Complexity
- **Dijkstra:**
  - **Time:** O((V + E) log V) with binary heap
  - **Space:** O(V + E) for adjacency list and distance array
- **Bellman-Ford:**
  - **Time:** O(V × E)
  - **Space:** O(V) for distance array

## Edge Cases
- Disconnected graph — some nodes remain at ∞; check for unreachable nodes
- Self-loops — Dijkstra handles them naturally (self-loop adds cost but won't improve distance)
- Negative weights — Dijkstra gives wrong results; must use Bellman-Ford
- Source == target — distance is 0
- Parallel edges between same pair — take the one with minimum weight
- "At most K stops" — need state = (node, stops_remaining) to avoid mixing paths with different stop counts

## Common Mistakes
- **Using Dijkstra with negative weights:** Dijkstra's greedy assumption fails. Use Bellman-Ford instead.
- **Not skipping stale entries in the priority queue:** Without `if (cost > dist[u]) continue;`, you process outdated entries and waste time.
- **Integer overflow with `dist[u] + w`:** If `dist[u]` is `Integer.MAX_VALUE`, adding `w` overflows. Always check `dist[u] != Integer.MAX_VALUE` before relaxing.
- **Bellman-Ford with K stops — chain relaxation:** When limiting to K stops, you must use a snapshot of the previous round's distances to prevent relaxing through multiple edges in one round.
- **Confusing 0-indexed vs 1-indexed nodes:** LeetCode problems use both; read the problem carefully.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Network Delay Time | Medium | Standard Dijkstra; answer is the maximum distance across all nodes | https://leetcode.com/problems/network-delay-time/ |
| 2 | Cheapest Flights Within K Stops | Medium | Bellman-Ford with K+1 rounds; or BFS with (node, stops, cost) state | https://leetcode.com/problems/cheapest-flights-within-k-stops/ |
| 3 | Path with Minimum Effort | Medium | Dijkstra on grid; edge weight = absolute height difference; minimize max effort along path | https://leetcode.com/problems/path-with-minimum-effort/ |
| 4 | Swim in Rising Water | Hard | Dijkstra on grid; minimize the maximum elevation along the path | https://leetcode.com/problems/swim-in-rising-water/ |
| 5 | Path with Maximum Probability | Medium | Modified Dijkstra with max-heap; multiply probabilities instead of adding costs | https://leetcode.com/problems/path-with-maximum-probability/ |
| 6 | Find the City With the Smallest Number of Neighbors at a Threshold Distance | Medium | Run Dijkstra from every city; count reachable cities within threshold | https://leetcode.com/problems/find-the-city-with-the-smallest-number-of-neighbors-at-a-threshold-distance/ |
| 7 | Design Graph With Shortest Path Calculator | Hard | Maintain adjacency list; run Dijkstra on each query | https://leetcode.com/problems/design-graph-with-shortest-path-calculator/ |
| 8 | Minimum Cost to Make at Least One Valid Path in a Grid | Hard | 0-1 BFS (deque); cost 0 if following arrow, cost 1 to change direction | https://leetcode.com/problems/minimum-cost-to-make-at-least-one-valid-path-in-a-grid/ |
| 9 | Shortest Path Visiting All Nodes | Hard | BFS with bitmask state (visited_mask, current_node); shortest path visiting all nodes | https://leetcode.com/problems/shortest-path-visiting-all-nodes/ |
| 10 | Path with Maximum Gold | Medium | DFS/backtracking on grid; collect maximum gold without revisiting cells | https://leetcode.com/problems/path-with-maximum-gold/ |
