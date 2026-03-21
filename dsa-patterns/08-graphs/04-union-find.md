# Pattern: Union-Find (Disjoint Set Union)

## How to Identify This Pattern

### Trigger Words
- "connected components", "number of provinces/groups"
- "merge", "union", "group together"
- "redundant connection", "extra edge"
- "are they in the same group/set?"
- "equivalence relation" (if a~b and b~c then a~c)
- "swap operations" that create equivalence classes

### Constraint Clues
- Need to dynamically merge sets and query connectivity
- Problem involves grouping elements by equivalence (same row, same column, same equation, etc.)
- Adding edges one by one and need to detect when a cycle forms
- Need to count connected components as edges are added
- O(n) or O(n × α(n)) time required — Union-Find is nearly O(1) per operation

### When to Use This vs Similar Patterns
| Consideration | Union-Find | DFS/BFS | Adjacency Matrix |
|---|---|---|---|
| Dynamic edge additions | Excellent (near O(1) per merge) | Must re-traverse | Must re-traverse |
| Static connectivity | Works but DFS simpler | Simple and clean | Too much space |
| Counting components | Decrement counter on union | Count DFS calls | Expensive |
| Detecting redundant edges | Natural (find before union) | Possible but complex | Not practical |

## Core Approach

### Union-Find with Path Compression + Union by Rank
1. **Initialize:** Create `parent[i] = i` and `rank[i] = 0` for all nodes.
2. **Find(x):** Follow parent pointers to root. Apply **path compression**: set every node on the path to point directly to the root.
3. **Union(x, y):**
   a. Find roots of x and y.
   b. If same root, they're already connected — return false (redundant edge).
   c. **Union by rank:** Attach the shorter tree under the taller one. If ranks are equal, pick either and increment its rank.
4. **Component count:** Start at N, decrement by 1 for each successful union.

## Java Template

### Union-Find Class
```java
class UnionFind {
    private int[] parent;
    private int[] rank;
    private int components;

    public UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        components = n;
        for (int i = 0; i < n; i++) {
            parent[i] = i;
        }
    }

    public int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]); // path compression
        }
        return parent[x];
    }

    public boolean union(int x, int y) {
        int rootX = find(x), rootY = find(y);
        if (rootX == rootY) return false; // already connected

        if (rank[rootX] < rank[rootY]) {
            parent[rootX] = rootY;
        } else if (rank[rootX] > rank[rootY]) {
            parent[rootY] = rootX;
        } else {
            parent[rootY] = rootX;
            rank[rootX]++;
        }
        components--;
        return true;
    }

    public boolean connected(int x, int y) {
        return find(x) == find(y);
    }

    public int getComponents() {
        return components;
    }
}
```

### Usage: Number of Provinces
```java
public int findCircleNum(int[][] isConnected) {
    int n = isConnected.length;
    UnionFind uf = new UnionFind(n);
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            if (isConnected[i][j] == 1) {
                uf.union(i, j);
            }
        }
    }
    return uf.getComponents();
}
```

### Usage: Redundant Connection
```java
public int[] findRedundantConnection(int[][] edges) {
    int n = edges.length;
    UnionFind uf = new UnionFind(n + 1); // 1-indexed nodes
    for (int[] edge : edges) {
        if (!uf.union(edge[0], edge[1])) {
            return edge; // this edge creates a cycle
        }
    }
    return new int[0];
}
```

## Dry Run Example

**Problem:** Number of Provinces — `isConnected = [[1,1,0],[1,1,0],[0,0,1]]`, n = 3.

| Step | Operation | parent[] | rank[] | Components |
|------|-----------|----------|--------|------------|
| Init | — | [0, 1, 2] | [0, 0, 0] | 3 |
| 1 | union(0, 1) | [0, 0, 2] | [1, 0, 0] | 2 |
| 2 | isConnected[0][2]=0 | skip | — | 2 |
| 3 | isConnected[1][2]=0 | skip | — | 2 |

**Result:** 2 provinces (nodes 0,1 form one group; node 2 is alone). ✓

**Problem:** Redundant Connection — edges = `[[1,2],[1,3],[2,3]]`.

| Step | Edge | find(u), find(v) | Action | parent[] |
|------|------|-------------------|--------|----------|
| Init | — | — | — | [0, 1, 2, 3] |
| 1 | [1,2] | 1, 2 (different) | union → merge | [0, 1, 1, 3] |
| 2 | [1,3] | 1, 3 (different) | union → merge | [0, 1, 1, 1] |
| 3 | [2,3] | find(2)→1, find(3)→1 (same!) | **redundant!** | return [2,3] |

**Result:** `[2, 3]` is the redundant connection. ✓

## Complexity
- **Time:** O(α(n)) per find/union operation, where α is the inverse Ackermann function (effectively constant). Total: O(n × α(n)) for n operations.
- **Space:** O(n) for parent and rank arrays.

## Edge Cases
- Single node — 1 component, no unions needed
- Already fully connected — all unions succeed, result is 1 component
- Multiple redundant edges — return the last one in the edge list (Redundant Connection problem)
- 0-indexed vs 1-indexed nodes — ensure parent array is sized correctly
- Self-loops — `union(x, x)` should return false (same root)
- Directed graph (Redundant Connection II) — requires a different approach with in-degree tracking

## Common Mistakes
- **Forgetting path compression:** Without it, find degrades to O(n) per call on skewed trees.
- **Not using union by rank:** Without it, trees can become tall chains, making find slow.
- **Off-by-one in node indexing:** If nodes are 1-indexed, allocate parent of size `n + 1`.
- **Using union result incorrectly:** `union()` returns false when nodes are already connected — this IS the signal for redundant edges and cycle detection.
- **Not making a deep copy in Accounts Merge:** When grouping emails, you must map every email to a root account index, then collect all emails belonging to the same root.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Number of Provinces | Medium | Union-Find on adjacency matrix; count remaining components | https://leetcode.com/problems/number-of-provinces/ |
| 2 | Redundant Connection | Medium | Union edges one by one; the edge that connects two already-connected nodes is redundant | https://leetcode.com/problems/redundant-connection/ |
| 3 | Accounts Merge | Medium | Union emails sharing an account; group by root and collect all emails per group | https://leetcode.com/problems/accounts-merge/ |
| 4 | Most Stones Removed with Same Row or Column | Medium | Union stones sharing a row or column; answer = total stones − number of components | https://leetcode.com/problems/most-stones-removed-with-same-row-or-column/ |
| 5 | Satisfiability of Equality Equations | Medium | Union variables in '==' equations; then check '!=' equations for contradictions | https://leetcode.com/problems/satisfiability-of-equality-equations/ |
| 6 | Smallest String With Swaps | Medium | Union indices that can be swapped; within each component, sort characters greedily | https://leetcode.com/problems/smallest-string-with-swaps/ |
| 7 | Number of Connected Components in an Undirected Graph | Medium | Standard Union-Find; count components after processing all edges | https://leetcode.com/problems/number-of-connected-components-in-an-undirected-graph/ |
| 8 | Regions Cut By Slashes | Medium | Upscale each cell to 3×3 grid; slashes become walls; count connected empty regions | https://leetcode.com/problems/regions-cut-by-slashes/ |
| 9 | Making A Large Island | Hard | Label islands with Union-Find; for each 0-cell, check unique adjacent island sizes | https://leetcode.com/problems/making-a-large-island/ |
| 10 | Minimize Hamming Distance After Swap Operations | Medium | Union swappable indices; within each component, match elements by frequency counting | https://leetcode.com/problems/minimize-hamming-distance-after-swap-operations/ |
