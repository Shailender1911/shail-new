# Pattern: Greedy Choice Property

## How to Identify This Pattern

### Trigger Words
- "maximize" or "minimize" something by **local decisions** (give, assign, schedule, pack)
- "minimum number of" steps / deletions / idle intervals when a **priority or ordering** is obvious
- "partition" so that each part satisfies a constraint — often process left-to-right with a greedy cutoff
- "as many as possible" with **sorting + one pass** or **heap** for the next best local option
- "change making", "distribute", "fill truck", "attend events" — feasibility checked myopically

### Constraint Clues
- After sorting (or using a fixed processing order), the **next best choice** does not depend on full future enumeration
- You can argue **exchange**: swapping an optimal solution to match the greedy pick does not worsen the objective
- Constraints are **cardinalities**, **capacities**, or **deadlines** where taking the "best available now" preserves feasibility
- Counterexample check: if a problem needs **optimal substructure with overlapping choices** (e.g., weighted), DP may replace greedy

### When to Use This vs Similar Patterns (Greedy vs DP comparison)

| Consideration | Greedy (local optimum) | Dynamic Programming |
|---------------|------------------------|---------------------|
| Decision order | Fixed by sort or sweep; one canonical "next" choice | Multiple branches; try all viable predecessors |
| Proof style | Exchange argument or "stay ahead" | Recurrence + memoization / tabulation |
| State size | O(1) or small extra structure (heap, counts) | Often O(n) or O(n²) states |
| When greedy fails | Local choice blocks a better global arrangement (dependencies) | Need to explore combinations |

**Rule of thumb:** If you can describe the algorithm as "always pick X first" after sorting, try greedy. If picking X early changes which Y values are even legal later in a non-local way, sketch a small counterexample and switch to DP.

## Core Approach

1. **Identify** what to optimize (maximize count, minimize groups, idle time, deletions, etc.).
2. **Order** the input: sort by deadline, size, end time, frequency, or process the string left-to-right.
3. **Maintain** lightweight state: counts, last position, remaining capacity, current balance.
4. **At each step**, make the **locally optimal** feasible move (e.g., serve smallest child first, place largest frequency first, extend current partition while a character still appears later).
5. **Validate** with a tiny counterexample if unsure; complexity should stay **O(n log n)** or **O(n)** for typical greedy solutions.

## Java Template

```java
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

/**
 * Generic pattern: sort, then greedy assignment.
 * Example: assign cookies — smallest cookie that satisfies each child (greatest hunger first or smallest child first).
 */
public int assignCookies(int[] g, int[] s) {
    Arrays.sort(g);
    Arrays.sort(s);
    int i = 0, j = 0;
    int count = 0;
    while (i < g.length && j < s.length) {
        if (s[j] >= g[i]) {
            count++;
            i++;
        }
        j++;
    }
    return count;
}

/**
 * Left-to-right greedy on array/string with a running invariant.
 * Example sketch: extend segment while condition holds, then commit.
 */
public void greedyScan(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n; ) {
        int j = i;
        while (j < n /* && extendCondition(i, j, arr) */) {
            j++;
        }
        // commit segment [i, j) or take action at j - 1
        i = j;
    }
}
```

## Dry Run Example

**Assign Cookies** — `g = [1,2,3]`, `s = [1,1]`.

1. Sort both (already sorted).
2. `j=0`: `s[0]=1 >= g[0]=1` → assign, `i=1`, `count=1`, `j=1`.
3. `j=1`: `s[1]=1 < g[1]=2` → skip cookie, `j=2`.
4. End → **1** child satisfied.

**Partition Labels** — `"ababcbacadefegdehijhklij"` (conceptual): scan left to right; track **last index** of each character; when `currentIndex == maxLastInWindow`, cut a partition — greedy earliest valid cut.

## Complexity
- **Time:** Usually **O(n log n)** if sorting dominates; **O(n)** for single passes with counting (e.g., frequency maps)
- **Space:** **O(1)** to **O(n)** for auxiliary arrays / heaps depending on problem

## Edge Cases
- Empty input or single element
- All values identical (ties in sort order)
- Greedy needs **long** or **boundary** arithmetic (avoid overflow on sums)
- Tasks with **cooldown** — idle slots; verify math with counts and formula, not off-by-one

## Common Mistakes
- **Wrong sort key** (e.g., by start instead of end, or by profit without feasibility).
- **Task Scheduler:** confusing `(n+1)` slots per cycle with actual idle insertion — derive from max frequency.
- **Frequency uniqueness:** deleting before duplicates collide is easier if you process **from high frequency downward**.
- **Events:** same-day overlap — sort by end time and **greedy count** non-overlapping with heap variant for room capacity.
- **Assuming greedy without proof** — always sanity-check with a small handcrafted case.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|------------|-------------|---------------|
| 1 | Assign Cookies | Easy | Sort children and cookies; greedily give smallest cookie that satisfies current child | https://leetcode.com/problems/assign-cookies/ |
| 2 | Candy | Hard | Two passes: left-to-right then right-to-left for local "higher than neighbor" constraints; sum ratings | https://leetcode.com/problems/candy/ |
| 3 | Lemonade Change | Easy | Simulate bills; greedily use $10 before $5 when giving change for $20 | https://leetcode.com/problems/lemonade-change/ |
| 4 | Task Scheduler | Medium | Bound idle time using max frequency and count of tasks tied at max; frame math or simulation | https://leetcode.com/problems/task-scheduler/ |
| 5 | Partition Labels | Medium | Record last occurrence of each char; extend window and cut when index reaches window max | https://leetcode.com/problems/partition-labels/ |
| 6 | Minimum Deletions to Make Character Frequencies Unique | Medium | Sort frequencies descending; greedily shrink down counts so all positive frequencies differ | https://leetcode.com/problems/minimum-deletions-to-make-character-frequencies-unique/ |
| 7 | Maximum Units on a Truck | Easy | Sort box types by units-per-box descending; fill truck greedily | https://leetcode.com/problems/maximum-units-on-a-truck/ |
| 8 | Reduce Array Size to The Half | Medium | Count frequencies; sort descending; take largest counts until removed elements ≥ n/2 | https://leetcode.com/problems/reduce-array-size-to-the-half/ |
| 9 | Maximum Number of Events That Can Be Attended | Medium | Sort by end time; greedy assign each event to earliest free day ≤ end (heap / tree or day sweep) | https://leetcode.com/problems/maximum-number-of-events-that-can-be-attended/ |
| 10 | Furthest Building You Can Reach | Medium | Min-heap of ladder heights; use bricks for smallest climbs; ladders for largest | https://leetcode.com/problems/furthest-building-you-can-reach/ |
