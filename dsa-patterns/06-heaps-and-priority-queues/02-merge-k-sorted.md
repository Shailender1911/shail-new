# Pattern: Merge K Sorted Sequences

## How to Identify This Pattern

### Trigger Words
- "merge K sorted lists/arrays"
- "smallest element across K sequences"
- "Kth smallest from multiple sorted sources"
- "smallest range covering elements from K lists"
- "almost sorted" or "K-sorted array"
- "ugly number", "super ugly number", "prime fraction"

### Constraint Clues
- Input is K sorted lists/arrays/streams that need to be unified
- Need to process elements in globally sorted order from multiple sorted sources
- Each source independently is sorted; you need the cross-source order
- K is moderate (hundreds or thousands of lists, but total elements N can be large)
- Problem generates candidates lazily from sorted dimensions (e.g., pairwise sums)

### When to Use This vs Similar Patterns
| Consideration | Merge K Sorted (Heap) | Two-Pointer Merge | Sort Everything | Binary Search |
|---|---|---|---|---|
| # of sources | K ≥ 2 | Exactly 2 | Any | Single sorted matrix |
| Time | O(N log K) | O(N) | O(N log N) | O(N log(max-min)) |
| Preserves order? | Yes | Yes | Yes | Only finds value |
| Lazy expansion? | Yes (expand one at a time) | No | No | N/A |
| When to prefer | K sources, need global order | Only 2 sources | Small N, any order | Finding Kth value only |

## Core Approach

### Variant 1: Merge K Sorted Linked Lists
1. Create a min-heap and add the head node of each non-empty list.
2. Poll the smallest node from the heap — append it to the result list.
3. If the polled node has a `.next`, push `.next` into the heap.
4. Repeat until the heap is empty.

### Variant 2: Merge K Sorted Arrays / Lazy Candidate Expansion
1. Create a min-heap. For each of the K arrays, insert `(value, listIndex, elementIndex)`.
2. Poll the minimum entry — it is the next element in global sorted order.
3. If `elementIndex + 1` is within bounds for that array, push the next element from the same array.
4. Repeat until you've extracted the desired number of elements (or the heap is empty).

### Variant 3: K-Sorted Array (each element is at most K positions from its sorted position)
1. Create a min-heap of size K+1.
2. Add the first K+1 elements to the heap.
3. For each remaining element: poll the min (it goes to the next output position), then add the new element.
4. After input is exhausted, drain the heap to fill the remaining output positions.

## Java Template

### Merge K Sorted Linked Lists
```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> minHeap = new PriorityQueue<>(
        Comparator.comparingInt(n -> n.val));

    for (ListNode head : lists) {
        if (head != null) minHeap.offer(head);
    }

    ListNode dummy = new ListNode(0);
    ListNode curr = dummy;

    while (!minHeap.isEmpty()) {
        ListNode smallest = minHeap.poll();
        curr.next = smallest;
        curr = curr.next;
        if (smallest.next != null) {
            minHeap.offer(smallest.next);
        }
    }
    return dummy.next;
}
```

### Merge K Sorted Arrays (Lazy Expansion)
```java
public List<Integer> mergeKSortedArrays(int[][] arrays) {
    PriorityQueue<int[]> minHeap = new PriorityQueue<>(
        Comparator.comparingInt(a -> a[0]));

    for (int i = 0; i < arrays.length; i++) {
        if (arrays[i].length > 0) {
            minHeap.offer(new int[]{arrays[i][0], i, 0});
        }
    }

    List<Integer> result = new ArrayList<>();
    while (!minHeap.isEmpty()) {
        int[] curr = minHeap.poll();
        result.add(curr[0]);
        int list = curr[1], idx = curr[2];
        if (idx + 1 < arrays[list].length) {
            minHeap.offer(new int[]{arrays[list][idx + 1], list, idx + 1});
        }
    }
    return result;
}
```

### Sort a K-Sorted (Almost Sorted) Array
```java
public int[] sortKSorted(int[] arr, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    int[] result = new int[arr.length];
    int idx = 0;

    for (int i = 0; i < arr.length; i++) {
        minHeap.offer(arr[i]);
        if (minHeap.size() > k) {
            result[idx++] = minHeap.poll();
        }
    }
    while (!minHeap.isEmpty()) {
        result[idx++] = minHeap.poll();
    }
    return result;
}
```

## Dry Run Example

**Problem:** Merge 3 sorted lists: `[1, 4, 5]`, `[1, 3, 4]`, `[2, 6]`

| Step | Heap State (value, list, idx) | Polled | Result So Far | Pushed |
|------|-------------------------------|--------|---------------|--------|
| Init | [(1,0,0), (1,1,0), (2,2,0)] | — | [] | heads of all 3 lists |
| 1 | [(1,1,0), (2,2,0), (4,0,1)] | (1,0,0) | [1] | (4,0,1) from list 0 |
| 2 | [(2,2,0), (4,0,1), (3,1,1)] | (1,1,0) | [1,1] | (3,1,1) from list 1 |
| 3 | [(3,1,1), (4,0,1), (6,2,1)] | (2,2,0) | [1,1,2] | (6,2,1) from list 2 |
| 4 | [(4,0,1), (6,2,1), (4,1,2)] | (3,1,1) | [1,1,2,3] | (4,1,2) from list 1 |
| 5 | [(4,1,2), (6,2,1), (5,0,2)] | (4,0,1) | [1,1,2,3,4] | (5,0,2) from list 0 |
| 6 | [(5,0,2), (6,2,1)] | (4,1,2) | [1,1,2,3,4,4] | list 1 exhausted |
| 7 | [(6,2,1)] | (5,0,2) | [1,1,2,3,4,4,5] | list 0 exhausted |
| 8 | [] | (6,2,1) | [1,1,2,3,4,4,5,6] | list 2 exhausted |

**Result:** `[1, 1, 2, 3, 4, 4, 5, 6]` ✓

**Key insight:** The heap never holds more than K elements (one per list), so each poll/offer is O(log K), and we do N total operations → O(N log K).

## Complexity
- **Time:** O(N log K) — N is the total number of elements across all K lists; each heap operation is O(log K)
- **Space:** O(K) — the heap holds at most K elements (one frontier element per list)

For K-sorted array: Time O(N log K), Space O(K).

## Edge Cases
- Some lists are empty — skip nulls / empty arrays before adding to the heap
- K = 1 — only one list; return it directly
- K = 2 — standard two-way merge with two pointers is simpler and avoids heap overhead
- All lists have a single element — heap degenerates to sorting K elements
- Lists of vastly different lengths — heap handles naturally since exhausted lists stop contributing
- Duplicate values across lists — heap handles them; stable ordering depends on comparator tiebreaking

## Common Mistakes
- **Adding null nodes to the heap:** Always check `head != null` and `node.next != null` before offering to the heap.
- **Not providing a comparator for custom objects:** `PriorityQueue` needs `Comparable` or an explicit `Comparator`; `ListNode` doesn't implement `Comparable` by default.
- **Expanding all elements upfront:** The whole point is lazy expansion — only push the next element from a list when the current one is polled. Pushing everything defeats the O(K) space advantage.
- **Using divide-and-conquer merge without understanding trade-offs:** Pairwise merging K lists is also O(N log K) but avoids a heap; pick based on whether you need streaming or batch.
- **Forgetting to handle the `dummy.next` return:** The result starts at `dummy.next`, not `dummy`.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Merge k Sorted Lists | Hard | Min-heap with K list heads; poll min, push its .next. Classic K-way merge. | https://leetcode.com/problems/merge-k-sorted-lists/ |
| 2 | Smallest Range Covering Elements from K Lists | Hard | Maintain one element per list in the heap; track global max to compute range. Shrink by advancing the min. | https://leetcode.com/problems/smallest-range-covering-elements-from-k-lists/ |
| 3 | Find K Pairs with Smallest Sums | Medium | Treat as K sorted lists (one per nums1 element). Min-heap with lazy expansion of nums2 index. | https://leetcode.com/problems/find-k-pairs-with-smallest-sums/ |
| 4 | Kth Smallest Element in a Sorted Matrix | Medium | Each row is a sorted list; K-way merge extracting K times. Binary search on value is an alternative. | https://leetcode.com/problems/kth-smallest-element-in-a-sorted-matrix/ |
| 5 | Merge Sorted Array | Easy | Two-pointer merge from the end for in-place merge of 2 sorted arrays. Foundation for K-way merge. | https://leetcode.com/problems/merge-sorted-array/ |
| 6 | Ugly Number II | Medium | Three pointers for factors 2, 3, 5 — equivalent to merging 3 sorted streams. Min-heap also works. | https://leetcode.com/problems/ugly-number-ii/ |
| 7 | Super Ugly Number | Medium | Generalization of Ugly Number II to K prime factors. Min-heap tracks the next candidate from each prime's stream. | https://leetcode.com/problems/super-ugly-number/ |
| 8 | K-th Smallest Prime Fraction | Medium | Binary search on fraction value, or min-heap merging K sorted rows of fractions. | https://leetcode.com/problems/k-th-smallest-prime-fraction/ |
| 9 | Find Median from Data Stream | Hard | Two-heap (max + min) maintains sorted halves; each add is O(log n). Related to maintaining sorted merge. | https://leetcode.com/problems/find-median-from-data-stream/ |
| 10 | Sort an Array (K-Sorted variant) | Medium | Min-heap of size K+1; each element is at most K away from its sorted position. Classic interview variant. | https://leetcode.com/problems/sort-an-array/ |
