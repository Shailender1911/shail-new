# Pattern: Activity Selection & Interval Greedy

## How to Identify This Pattern

### Trigger Words
- "non-overlapping", "maximum number of intervals", "schedule", "meetings", "appointments"
- "minimum removals" to avoid overlap, "burst balloons" with one shot per vertical line
- "cover" a segment or timeline with clips, taps, or jumps
- "merge" or "choose" intervals subject to a greedy ordering (often by end time)
- "reconstruct" order from constraints that behave like nested or chained intervals

### Constraint Clues
- Input is a list of intervals `[start, end]` or equivalent ranges on a line
- Objective is **max count** of compatible choices or **min count** of resources (arrows, removals)
- Greedy proof often: picking the interval that **ends earliest** leaves maximum room for the rest
- Variants: 2D intervals treated as 1D (balloons), "jump" reach as interval coverage, gas as circular net flow

### When to Use This vs Similar Patterns (Greedy vs DP comparison)

| Situation | Greedy (intervals / jumps) | Dynamic Programming |
|-----------|----------------------------|---------------------|
| Structure | Choosing one interval does not create overlapping subproblems with incompatible earlier choices once you sort | Optimal substructure is unclear; local choice can block a better global plan |
| Ordering | A clear sort key (end time, start time) makes the next choice obvious | State must remember *which* intervals were taken or exact position |
| Objective | Maximize count / minimize resources with **exchange argument** | Minimize cost with dependencies (e.g., weighted interval scheduling with weights) |
| Typical risk | Assuming sort by start always works | Using greedy when counterexamples exist (weighted intervals → DP or binary search on DP) |

**Rule of thumb:** If the classic "activity selection" exchange argument applies (earliest finishing compatible interval), use greedy. If intervals have weights or complex dependencies, consider DP on sorted endpoints.

## Core Approach

1. **Model** the problem as intervals on a line (or map balloons, video clips, taps, jumps onto start/end semantics).
2. **Sort** by a deliberate key — most often **ascending end time** (activity selection). Sometimes **ascending start** with a different greedy invariant (e.g., jump games track farthest reach).
3. **Scan** in sorted order: maintain the **last chosen end** (or current coverage boundary). For each candidate, if it **does not conflict** with the last choice (start ≥ lastEnd, or equivalent), **take** it and update lastEnd.
4. For **minimum arrows / merges**, count one resource per greedy "shot" or group when the scan crosses a boundary.
5. For **jump-style** problems, iterate positions and repeatedly extend **farthest reachable** index within the current jump window; advance jumps when you step past the end of the current window.

## Java Template

```java
import java.util.Arrays;

/**
 * Classic activity selection: maximum number of non-overlapping intervals.
 * Intervals[i] = [start, end]. Sort by end ascending.
 */
public int maxNonOverlappingIntervals(int[][] intervals) {
    if (intervals.length == 0) return 0;
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[1], b[1]));
    int count = 0;
    int lastEnd = Integer.MIN_VALUE;
    for (int[] iv : intervals) {
        if (iv[0] >= lastEnd) {
            count++;
            lastEnd = iv[1];
        }
    }
    return count;
}

/**
 * Jump Game II flavor: minimum jumps to reach last index (greedy on farthest reach).
 */
public int jumpGameII(int[] nums) {
    int n = nums.length;
    if (n <= 1) return 0;
    int jumps = 0;
    int end = 0;      // farthest index reachable with current number of jumps
    int farthest = 0;
    for (int i = 0; i < n - 1; i++) {
        farthest = Math.max(farthest, i + nums[i]);
        if (i == end) {
            jumps++;
            end = farthest;
        }
    }
    return jumps;
}
```

## Dry Run Example

**Problem:** Activity selection — intervals `[[1,2],[2,3],[3,4]]`, maximize count (non-overlap with `start >= previousEnd`).

1. Sort by end: `[[1,2],[2,3],[3,4]]` (already sorted).
2. Take `[1,2]` → `lastEnd = 2`, count = 1.
3. Take `[2,3]` because `2 >= 2` → `lastEnd = 3`, count = 2.
4. Take `[3,4]` because `3 >= 3` → `lastEnd = 4`, count = 3.

**Arrows / balloons:** Sort by `x_end`; shoot at `end` of first balloon; skip all balloons with `start <= shot` until `start > shot`, then repeat.

## Complexity
- **Time:** O(n log n) for sorting + O(n) scan → **O(n log n)** dominated by sort
- **Space:** O(1) extra beyond the input array (sort may be O(log n) stack for mergesort in library)

## Edge Cases
- Empty list or single interval
- All intervals identical or fully nested
- Intervals touching at one point — verify whether that counts as overlap (`<=` vs `<` on endpoints)
- Jump array with `0` at a stuck position (Jump Game / II)
- Circular gas (sum constraint) vs interval greedy on a line

## Common Mistakes
- **Sorting by start only** when end-time greedy is required (or vice versa) — pick the sort key that matches the proof.
- **Off-by-one on overlap:** mixing open/closed intervals; always tie-break consistently with the problem.
- **Jump Game II:** BFS from every index is slower; the **farthest-in-window** greedy is O(n).
- **Confusing "minimum removals" with "maximum non-overlapping":** answer is often `n - maxNonOverlapping`.
- **Taps / stitching:** forgetting to map the problem to interval coverage on `[0, target]` with the correct coordinate system.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|------------|-------------|---------------|
| 1 | Non-overlapping Intervals | Medium | Sort by end; greedily keep intervals that start after last end; removals = n − kept | https://leetcode.com/problems/non-overlapping-intervals/ |
| 2 | Minimum Number of Arrows to Burst Balloons | Medium | Sort by balloon end; one arrow at each new end when current arrow can't reach next start | https://leetcode.com/problems/minimum-number-of-arrows-to-burst-balloons/ |
| 3 | Jump Game | Medium | Track farthest reachable index; if you can't pass current index, fail | https://leetcode.com/problems/jump-game/ |
| 4 | Jump Game II | Medium | Greedy layers: extend farthest within current jump window; increment jump when index hits window end | https://leetcode.com/problems/jump-game-ii/ |
| 5 | Video Stitching | Medium | Sort clips by start; greedily extend coverage with clip that starts ≤ current and maximizes end | https://leetcode.com/problems/video-stitching/ |
| 6 | Minimum Number of Taps to Open to Water a Garden | Hard | Convert tap ranges to intervals on [0, n]; greedy cover like stitching / minimum arrows variant | https://leetcode.com/problems/minimum-number-of-taps-to-open-to-water-a-garden/ |
| 7 | Gas Station | Medium | If total gas ≥ total cost, a start exists; sweep tracking tank — reset candidate start when tank drops | https://leetcode.com/problems/gas-station/ |
| 8 | Queue Reconstruction by Height | Medium | Sort people by height desc, then k asc; insert each at index k — greedy placement respects front-count | https://leetcode.com/problems/queue-reconstruction-by-height/ |
| 9 | Boats to Save People | Medium | Sort weights; two-pointer: try heaviest + lightest if within limit; else heaviest alone | https://leetcode.com/problems/boats-to-save-people/ |
| 10 | Bag of Tokens | Medium | Sort tokens; greedy two-pointer — spend power on face-up smallest when score matters, face-down largest to regain power | https://leetcode.com/problems/bag-of-tokens/ |
