# Pattern: Meeting Rooms and Scheduling

## How to Identify This Pattern

### Trigger Words
- "meeting rooms", "can attend all meetings", "minimum rooms", "minimum platforms"
- "calendar" booking, "double booking", "K-booking", maximum concurrent events
- "car pooling", capacity along a trip timeline
- "employee free time", common free intervals across schedules
- "queries" asking for shortest interval covering each query point
- "maximum events" attendable in a day (intervals with choice)
- "courses" with duration and deadline — schedule as many as possible

### Constraint Clues
- Each item has a **time window** `[start, end)` or `[start, end]` — confirm inclusivity
- Question is about **compatibility** (no overlap) or **resource count** (max overlap)
- `n` up to 10⁵–10⁶ suggests **O(n log n)** sorting + heap or sweep, not O(n²) all pairs

### When to Use This vs Similar Patterns
| Consideration | Meeting rooms / sweep | Merge intervals | Greedy by deadline |
|---|---|---|---|
| Question | Can schedule? How many rooms? | Canonical merged form | Pick max tasks by end/deadline |
| Key structure | Chronological sweep + min-heap of end times | Sort by start, merge | Sort courses/events by deadline or end |
| Classic signal | "minimum meeting rooms" | "merge overlapping" | "maximum events/courses" |

## Core Approach

### Can attend all meetings (no overlap)
1. Sort intervals by **start** time.
2. Ensure each `start >= previous end` (strict `>` if half-open `[a,b)` vs `[b,c)`).

### Minimum meeting rooms (max concurrent meetings)
1. Sort intervals by **start**.
2. Use a **min-heap** (or multiset) of **end** times of ongoing meetings.
3. For each meeting, pop all ended meetings (`earliestEnd <= start`). Push current `end`.
4. Heap size is the number of rooms in use; track maximum.

### Car pooling (capacity on a line)
1. **Difference array / sweep:** at each `start` add passengers, at each `end` subtract (watch inclusive/exclusive end).
2. Or sort **events** `(time, delta)` and sweep cumulative sum ≤ capacity.

### Calendar series (I / II / III)
- **I:** track last booked interval; check overlap.
- **II:** overlap at most once — sweep overlaps with a structure or count overlaps per point.
- **III:** k-overlap — line sweep with active count or interval tree logic.

### Employee free time
1. Merge each employee’s intervals, then **gap** between consecutive busy blocks across a merged timeline (common pattern: merge all, then find holes).

### Minimum interval to include each query
- Sort intervals by start; for each query use binary search + track best candidate by smallest length (or sweep with structure).

## Java Template

```java
import java.util.*;

// Minimum meeting rooms: sort by start, min-heap of end times
public int minMeetingRooms(int[][] intervals) {
    if (intervals.length == 0) return 0;
    Arrays.sort(intervals, Comparator.comparingInt(a -> a[0]));
    PriorityQueue<Integer> pq = new PriorityQueue<>();
    int max = 0;
    for (int[] m : intervals) {
        while (!pq.isEmpty() && pq.peek() <= m[0]) {
            pq.poll();
        }
        pq.offer(m[1]);
        max = Math.max(max, pq.size());
    }
    return max;
}

// Can attend all meetings: sort by start, check end ordering
public boolean canAttendMeetings(int[][] intervals) {
    Arrays.sort(intervals, Comparator.comparingInt(a -> a[0]));
    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] < intervals[i - 1][1]) return false;
    }
    return true;
}
```

## Dry Run Example

**Meeting Rooms II** — intervals: `[0,30], [5,10], [15,20]` sorted by start.

| Meeting | Heap (ends) after evicting finished | Rooms in use | Max |
|---------|--------------------------------------|--------------|-----|
| [0,30] | [30] | 1 | 1 |
| [5,10] | [10,30] (`30 > 5`, keep both) | 2 | 2 |
| [15,20] | pop 10 (`10 <= 15`), then [20,30] | 2 | 2 |

Answer: **2** rooms.

**Car pooling** style sweep — trips `[[2,1,5],[3,3,7]]` as events: `(1,+2), (5,-2), (3,+3), (7,-3)` sorted by time (with tie-break: drop before add at same timestamp if end is exclusive).

| Time | Delta | Passengers |
|------|-------|------------|
| 1 | +2 | 2 |
| 3 | +3 | 5 |
| 5 | -2 | 3 |
| 7 | -3 | 0 |

Check `capacity` at each step.

## Complexity
- **Sort + heap (meeting rooms II):** **O(n log n)** time, **O(n)** space for heap.
- **Sweep with TreeMap / events list:** **O(n log n)** time typical.
- **Per-query binary search problems:** **O((n + q) log n)** with preprocessing.

## Edge Cases
- Empty schedule; one meeting
- Identical start times — heap must support multiple concurrent ends
- **Touching** meetings `[0,5]` and `[5,10]` — usually non-overlapping if half-open; overlapping if closed intervals — read carefully
- Events spanning midnight or date boundaries — normalize to timestamps
- Large coordinate range — coordinate compression for difference-array approaches

## Common Mistakes
- Using **meeting rooms II** heap pattern when the problem needs **merge-intervals** style answer (different outputs).
- Wrong **end exclusivity** in car pooling / sweep (off-by-one at drop-off).
- For **My Calendar II**, forgetting that **triple** overlap is disallowed, not double booking of the same interval twice only.
- Not sorting by **start** before heap sweep; unsorted intervals break the eviction rule.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Meeting Rooms | Easy | Sort by start; overlap iff `curr.start < prev.end` | https://leetcode.com/problems/meeting-rooms/ |
| 2 | Meeting Rooms II | Medium | Sort by start; min-heap of end times; max heap size | https://leetcode.com/problems/meeting-rooms-ii/ |
| 3 | Car Pooling | Medium | Event sweep: +cap at start, −cap at end; prefix sum ≤ capacity | https://leetcode.com/problems/car-pooling/ |
| 4 | My Calendar I | Medium | Book if no overlap with existing; tree map or list + binary search | https://leetcode.com/problems/my-calendar-i/ |
| 5 | My Calendar II | Medium | Track booked intervals; allow double-book but detect triple overlap | https://leetcode.com/problems/my-calendar-ii/ |
| 6 | My Calendar III | Hard | Line sweep: count overlapping bookings; max is k-booking | https://leetcode.com/problems/my-calendar-iii/ |
| 7 | Employee Free Time | Hard | Merge each person’s busy intervals; gaps in union are free slots | https://leetcode.com/problems/employee-free-time/ |
| 8 | Minimum Interval to Include Each Query | Hard | Sort intervals; for each query binary search candidates, pick smallest span | https://leetcode.com/problems/minimum-interval-to-include-each-query/ |
| 9 | Maximum Number of Events That Can Be Attended | Medium | Greedy by early end / day capacity; often sort + heap or day sweep | https://leetcode.com/problems/maximum-number-of-events-that-can-be-attended/ |
| 10 | Course Schedule III | Hard | Sort by `lastDay`; min-heap of durations; swap out longest if over deadline | https://leetcode.com/problems/course-schedule-iii/ |
