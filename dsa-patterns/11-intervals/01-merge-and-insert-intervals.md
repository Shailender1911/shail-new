# Pattern: Merge and Insert Intervals

## How to Identify This Pattern

### Trigger Words
- "merge overlapping intervals", "combine intervals", "union of intervals"
- "insert a new interval" into a sorted, non-overlapping list
- "intersection" of two lists of intervals
- "minimum removals" so intervals do not overlap (or become non-overlapping)
- "covered interval", "subset interval", "one interval completely inside another"
- "burst balloons", "arrows", "minimum arrows" (intervals on a line)
- "summary of consecutive numbers", "ranges as strings"
- "missing values" between bounds given sorted unique integers
- "add/remove/query range" on a number line; disjoint intervals in a stream

### Constraint Clues
- Input is a list of `[start, end]` pairs (closed intervals unless stated otherwise)
- Intervals may be unsorted and overlapping; output often should be merged or non-overlapping
- Bounds are integers; `n` can be large — aim for **O(n log n)** sort + **O(n)** scan when possible
- Dynamic updates (add/remove) suggest ordered map / tree structure or lazy merging

### When to Use This vs Similar Patterns
| Consideration | Merge / insert intervals | Meeting rooms / sweep line | Greedy by end time |
|---|---|---|---|
| Core object | Closed ranges on 1D axis | Events with start/end + counts | Often sort by end, count conflicts |
| Typical sort key | By **start** (then linear merge) | By **start** (heap for concurrent count) | By **end** for "maximum non-overlapping" |
| Output | Merged list, insert result, intersections | Max overlap, feasibility | Min arrows, min removals |

## Core Approach

### Merge overlapping intervals
1. If intervals are not guaranteed sorted, **sort by start** (tie-break by end if needed).
2. Keep a **current** merged interval; for each next interval `[s, e]`:
   - If `s <= current.end` (overlap or touch, if problem counts touching as merge), extend `current.end = max(current.end, e)`.
   - Else push `current` to the answer and start a new `current = [s, e]`.
3. Do not forget to push the last `current`.

### Insert into sorted non-overlapping intervals
1. Scan linearly (or binary search by start): append intervals fully before the new interval.
2. **Merge** the new interval with every overlapping interval while advancing.
3. Append intervals fully after the merged block.

### Two-pointer intersection of two sorted lists
1. Compare `a[i]` and `b[j]`; the intersection is `[max(starts), min(ends)]` if `max(starts) <= min(ends)`.
2. Drop the interval with the **smaller end** (it cannot intersect the other list further).

### Minimum arrows / non-overlapping removals (greedy by end)
1. Sort by **end** (or process merged by end).
2. Greedily keep intervals that end earliest; count shots or removals when a new interval starts **after** the last kept end (strict inequality depends on open vs closed — read the statement).

## Java Template

```java
import java.util.*;

// Merge overlapping intervals: sort by start, linear scan
public int[][] merge(int[][] intervals) {
    if (intervals.length <= 1) return intervals;
    Arrays.sort(intervals, Comparator.comparingInt(a -> a[0]));
    List<int[]> merged = new ArrayList<>();
    int[] cur = intervals[0];
    for (int i = 1; i < intervals.length; i++) {
        int[] n = intervals[i];
        if (n[0] <= cur[1]) { // overlap (use < if touching should NOT merge)
            cur[1] = Math.max(cur[1], n[1]);
        } else {
            merged.add(cur);
            cur = n;
        }
    }
    merged.add(cur);
    return merged.toArray(new int[0][]);
}

// Insert interval into sorted disjoint intervals → merge-style extension
public int[][] insert(int[][] intervals, int[] newInterval) {
    List<int[]> ans = new ArrayList<>();
    int i = 0, n = intervals.length;
    int s = newInterval[0], e = newInterval[1];
    while (i < n && intervals[i][1] < s) {
        ans.add(intervals[i++]);
    }
    while (i < n && intervals[i][0] <= e) {
        s = Math.min(s, intervals[i][0]);
        e = Math.max(e, intervals[i][1]);
        i++;
    }
    ans.add(new int[]{s, e});
    while (i < n) ans.add(intervals[i++]);
    return ans.toArray(new int[0][]);
}
```

## Dry Run Example

**Merge Intervals** — input: `[1,3], [2,6], [8,10], [15,18]` (after sort by start: same order).

| Step | Current merged | Next interval | Action |
|------|----------------|---------------|--------|
| Init | `[1,3]` | `[2,6]` | `2 <= 3` → extend end → `[1,6]` |
| 2 | `[1,6]` | `[8,10]` | `8 > 6` → push `[1,6]`, current = `[8,10]` |
| 3 | `[8,10]` | `[15,18]` | `15 > 10` → push `[8,10]`, current = `[15,18]` |
| Done | — | — | push `[15,18]` → result: `[1,6], [8,10], [15,18]` |

**Interval list intersections** — `A = [0,2], [5,10]` and `B = [1,5], [8,12]` (two pointers).

| i,j | Compare | Intersection | Advance |
|-----|---------|--------------|---------|
| 0,0 | [0,2] vs [1,5] | [1,2] | A ends first → i++ |
| 1,0 | [5,10] vs [1,5] | [5,5] | B ends first → j++ |
| 1,1 | [5,10] vs [8,12] | [8,10] | A ends first → i++ |

## Complexity
- **Merge / insert scan:** **O(n log n)** time if sorting is required, **O(n)** extra for the result list; **O(1)** or **O(n)** auxiliary depending on in-place constraints.
- **Two-pointer intersections:** **O(m + n)** time, **O(1)** extra besides output.
- **TreeMap / ordered-map range problems:** **O(log n)** per update/query typical.

## Edge Cases
- Empty list or single interval
- **Touching** intervals `[1,2], [2,3]` — merge or not per problem definition
- Interval completely inside another — merge collapses to outer; "remove covered" uses containment logic
- Very large coordinates — use `long` if cross-multiplying or accumulating
- Duplicate intervals — usually harmless after sort

## Common Mistakes
- Sorting by **end** when **start** order is required for linear merge (or vice versa for arrow/greedy problems).
- Using `<=` vs `<` for overlap; misreading "touching" as overlap for balloons vs merge.
- Off-by-one in intersection: need `max(start) <= min(end)` for a non-empty intersection.
- For insert, forgetting to merge **all** overlapping intervals in one pass before appending the rest.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Merge Intervals | Medium | Sort by start; extend `end` while `next.start <= cur.end` | https://leetcode.com/problems/merge-intervals/ |
| 2 | Insert Interval | Medium | Linear (or binary) scan; one merge pass for overlapping block | https://leetcode.com/problems/insert-interval/ |
| 3 | Interval List Intersections | Medium | Two pointers on sorted lists; advance interval with smaller `end` | https://leetcode.com/problems/interval-list-intersections/ |
| 4 | Non-overlapping Intervals | Medium | Sort by end; greedy keep; count removals when `start < lastEnd` | https://leetcode.com/problems/non-overlapping-intervals/ |
| 5 | Remove Covered Intervals | Medium | Sort by start asc, end desc; sweep keeping maximal `end` so far | https://leetcode.com/problems/remove-covered-intervals/ |
| 6 | Minimum Number of Arrows to Burst Balloons | Medium | Sort by end; one arrow per cluster when `start > lastShotEnd` | https://leetcode.com/problems/minimum-number-of-arrows-to-burst-balloons/ |
| 7 | Summary Ranges | Easy | Scan sorted array; break range when `nums[i] != nums[i-1] + 1` | https://leetcode.com/problems/summary-ranges/ |
| 8 | Missing Ranges | Medium | Walk the sorted range `[lower, upper]`; emit gaps between consecutive values | https://leetcode.com/problems/missing-ranges/ |
| 9 | Range Module | Hard | `TreeMap` (or treap) of disjoint intervals; merge/split on add, erase on remove | https://leetcode.com/problems/range-module/ |
| 10 | Data Stream as Disjoint Intervals | Hard | Ordered map of start → end; merge neighbors on each `addNum` | https://leetcode.com/problems/data-stream-as-disjoint-intervals/ |
