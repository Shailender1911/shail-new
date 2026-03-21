# Pattern: Two Heaps and Median Maintenance

## How to Identify This Pattern

### Trigger Words
- "find median", "running median", "median from data stream"
- "sliding window median"
- "balance two halves", "split into two groups"
- "maximize minimum", "minimize maximum"
- "schedule", "assign resources optimally"
- "alternating arrangement", "no two adjacent same"

### Constraint Clues
- Need the median after every insertion (online / streaming data)
- Must maintain a dynamic partition of elements into a lower half and upper half
- Need to track the maximum of one group and minimum of another simultaneously
- Problem involves balancing workload, capital, or resources between two sides
- Constraint says O(log n) per insertion with O(1) median query

### When to Use This vs Similar Patterns
| Consideration | Two Heaps | Sorted Array/List | Balanced BST | Segment Tree |
|---|---|---|---|---|
| Insert + Median query | O(log n) + O(1) | O(n) + O(1) | O(log n) + O(log n) | O(log n) + O(log n) |
| Supports removal? | Hard (lazy deletion) | O(n) | O(log n) | O(log n) |
| Implementation ease | Simple | Trivial | Complex | Complex |
| Sliding window? | Needs lazy deletion | Too slow | Works well | Works well |
| Best for | Streaming median | Small N | Sliding window | Range queries |

## Core Approach

### Variant 1: Streaming Median (Find Median from Data Stream)
1. Maintain two heaps:
   - **maxHeap** (lower half) — stores the smaller half; root = largest of the lower half.
   - **minHeap** (upper half) — stores the larger half; root = smallest of the upper half.
2. **Invariant:** `maxHeap.size() == minHeap.size()` or `maxHeap.size() == minHeap.size() + 1`.
3. To add a number:
   a. If `maxHeap` is empty or `num ≤ maxHeap.peek()`, add to `maxHeap`.
   b. Otherwise, add to `minHeap`.
   c. **Rebalance:** if `maxHeap.size() > minHeap.size() + 1`, move maxHeap's root to minHeap.
   d. If `minHeap.size() > maxHeap.size()`, move minHeap's root to maxHeap.
4. **Median:** if sizes are equal, average of both roots; otherwise, maxHeap's root.

### Variant 2: Sliding Window Median
1. Use the same two-heap structure as above.
2. When the window slides, **add** the new element and **remove** the outgoing element.
3. Removal uses **lazy deletion**: mark the element as removed, and only physically remove it when it surfaces as a root during rebalancing.
4. Maintain a `balance` counter to track how many pending deletions are in each heap.

### Variant 3: Maximize Capital (IPO-style)
1. Use a **minHeap** sorted by cost to find affordable projects.
2. Use a **maxHeap** sorted by profit to pick the most profitable affordable project.
3. For each round: move all newly affordable projects from minHeap to maxHeap, then pick the max-profit project.

## Java Template

### Find Median from Data Stream
```java
class MedianFinder {
    private PriorityQueue<Integer> maxHeap; // lower half
    private PriorityQueue<Integer> minHeap; // upper half

    public MedianFinder() {
        maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        minHeap = new PriorityQueue<>();
    }

    public void addNum(int num) {
        if (maxHeap.isEmpty() || num <= maxHeap.peek()) {
            maxHeap.offer(num);
        } else {
            minHeap.offer(num);
        }

        // Rebalance: maxHeap can have at most 1 extra element
        if (maxHeap.size() > minHeap.size() + 1) {
            minHeap.offer(maxHeap.poll());
        } else if (minHeap.size() > maxHeap.size()) {
            maxHeap.offer(minHeap.poll());
        }
    }

    public double findMedian() {
        if (maxHeap.size() > minHeap.size()) {
            return maxHeap.peek();
        }
        return (maxHeap.peek() + minHeap.peek()) / 2.0;
    }
}
```

### IPO / Maximize Capital
```java
public int findMaximizedCapital(int k, int w, int[] profits, int[] capital) {
    int n = profits.length;
    int[][] projects = new int[n][2];
    for (int i = 0; i < n; i++) {
        projects[i] = new int[]{capital[i], profits[i]};
    }
    Arrays.sort(projects, Comparator.comparingInt(a -> a[0]));

    PriorityQueue<Integer> maxProfitHeap = new PriorityQueue<>(Collections.reverseOrder());
    int idx = 0;

    for (int i = 0; i < k; i++) {
        while (idx < n && projects[idx][0] <= w) {
            maxProfitHeap.offer(projects[idx][1]);
            idx++;
        }
        if (maxProfitHeap.isEmpty()) break;
        w += maxProfitHeap.poll();
    }
    return w;
}
```

### Meeting Rooms II (Min-Heap for end times)
```java
public int minMeetingRooms(int[][] intervals) {
    Arrays.sort(intervals, Comparator.comparingInt(a -> a[0]));
    PriorityQueue<Integer> endTimes = new PriorityQueue<>();

    for (int[] interval : intervals) {
        if (!endTimes.isEmpty() && endTimes.peek() <= interval[0]) {
            endTimes.poll();
        }
        endTimes.offer(interval[1]);
    }
    return endTimes.size();
}
```

## Dry Run Example

**Problem:** Find Median from Data Stream — insert `[5, 15, 1, 3]` and find median after each insertion.

| Step | Insert | maxHeap (lower) | minHeap (upper) | Rebalance? | Median |
|------|--------|-----------------|-----------------|------------|--------|
| 1 | 5 | [5] | [] | No | 5.0 |
| 2 | 15 | [5] | [15] | No | (5+15)/2 = 10.0 |
| 3 | 1 | [5, 1] | [15] | No (sizes 2 vs 1, diff=1) | 5.0 |
| 4 | 3 | [5, 3, 1] | [15] | Yes: maxHeap has 3, minHeap has 1 → move 5 to minHeap | 3+5 = 8/2 = 4.0 |

After step 4: maxHeap = [3, 1], minHeap = [5, 15]

| State | maxHeap (peek = max of lower) | minHeap (peek = min of upper) | Median |
|-------|-------------------------------|-------------------------------|--------|
| Final | 3 | 5 | (3 + 5) / 2.0 = **4.0** ✓ |

**Verification:** Sorted order is `[1, 3, 5, 15]`. Median = (3 + 5) / 2 = 4.0 ✓

**Why it works:** The maxHeap always holds the smaller half and the minHeap holds the larger half. The roots of the two heaps are the two middle elements (or the single middle element if odd count), giving O(1) median access.

## Complexity
- **Time:**
  - `addNum`: O(log n) — one heap insert + at most one rebalance transfer
  - `findMedian`: O(1) — peek both roots
  - Sliding window median: O(n log n) for n elements with lazy deletion
- **Space:** O(n) — all elements stored across both heaps

## Edge Cases
- Single element — median is that element; maxHeap has size 1, minHeap is empty
- Two elements — median is the average of both roots
- All identical elements — both heaps work correctly; median is that repeated value
- Negative numbers — heaps and comparators handle negatives naturally
- Integer overflow in averaging — use `(long)maxHeap.peek() + minHeap.peek()` before dividing
- Sliding window: removing an element not at the root — lazy deletion must track pending removes per heap

## Common Mistakes
- **Putting all elements in one heap:** You need two heaps. The maxHeap holds the lower half, the minHeap holds the upper half. Mixing this up breaks the invariant.
- **Wrong rebalancing direction:** After every insert, ensure `|maxHeap.size() - minHeap.size()| ≤ 1`. Always move from the larger heap to the smaller.
- **Not using `Collections.reverseOrder()` for maxHeap:** Java's `PriorityQueue` is a min-heap by default. The lower-half heap must be a max-heap.
- **Integer overflow when computing median:** `(maxHeap.peek() + minHeap.peek()) / 2` overflows for large ints. Cast to `double` or `long` first.
- **Lazy deletion bookkeeping errors:** In sliding window problems, track a `balance` variable and only physically remove stale elements when they appear at the root.
- **Forgetting the empty-heap edge case in `findMedian`:** If only one heap has elements, don't try to peek the other.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Find Median from Data Stream | Hard | MaxHeap (lower half) + MinHeap (upper half); rebalance after each insert; median from roots. | https://leetcode.com/problems/find-median-from-data-stream/ |
| 2 | Sliding Window Median | Hard | Two-heap with lazy deletion; track balance to defer physical removal until element surfaces at root. | https://leetcode.com/problems/sliding-window-median/ |
| 3 | IPO | Hard | MinHeap by capital to unlock projects; MaxHeap by profit to greedily pick the best affordable project each round. | https://leetcode.com/problems/ipo/ |
| 4 | Meeting Rooms II | Medium | Min-heap of end times; if earliest ending meeting ends before next starts, reuse the room (poll), else add a room. | https://leetcode.com/problems/meeting-rooms-ii/ |
| 5 | Task Scheduler | Medium | MaxHeap by remaining count; greedily schedule the most frequent task; use a cooldown queue for waiting tasks. | https://leetcode.com/problems/task-scheduler/ |
| 6 | Smallest Range Covering Elements from K Lists | Hard | Two-heap / K-pointer approach; maintain one element per list, track max, minimize range by advancing min. | https://leetcode.com/problems/smallest-range-covering-elements-from-k-lists/ |
| 7 | Distant Barcodes | Medium | MaxHeap by frequency; place most frequent barcode first, alternate to avoid adjacent duplicates. Same pattern as Reorganize String. | https://leetcode.com/problems/distant-barcodes/ |
| 8 | Car Pooling | Medium | Min-heap (or difference array) by pickup location; track active passengers and check against capacity. | https://leetcode.com/problems/car-pooling/ |
| 9 | Last Stone Weight | Easy | MaxHeap; repeatedly smash two heaviest stones; if unequal, push the difference back. Simulate until ≤ 1 stone left. | https://leetcode.com/problems/last-stone-weight/ |
| 10 | Seat Reservation Manager | Medium | Min-heap of available seats; reserve() polls the smallest, unreserve() offers it back. O(log n) per operation. | https://leetcode.com/problems/seat-reservation-manager/ |
