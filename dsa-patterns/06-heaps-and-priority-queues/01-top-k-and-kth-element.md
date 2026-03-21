# Pattern: Top-K and Kth Element

## How to Identify This Pattern

### Trigger Words
- "kth largest", "kth smallest", "kth most frequent"
- "top K", "K most", "K least", "K closest"
- "sort by frequency", "most common", "least common"
- "rank", "order statistics"
- "reorganize", "rearrange by frequency"

### Constraint Clues
- Need only the K largest/smallest elements, not a full sort
- K is significantly smaller than N (partial sort is cheaper than full sort)
- Stream of data where you need running kth element
- Need to repeatedly extract the next most/least frequent element
- Constraint says O(n log k) is acceptable

### When to Use This vs Similar Patterns
| Consideration | Top-K Heap | Full Sort | QuickSelect | Bucket Sort |
|---|---|---|---|---|
| Need all K elements? | Yes, ordered | Yes, all N | Only kth element | Yes, if bounded range |
| Time complexity | O(n log k) | O(n log n) | O(n) average | O(n) |
| Space complexity | O(k) | O(1) or O(n) | O(1) | O(n) |
| Online / streaming? | Yes | No | No | No |
| When K ≈ N? | Degrades to sort | Prefer this | Prefer this | If values bounded |

## Core Approach

### Variant 1: Kth Largest / Top-K Largest Elements
1. Create a **min-heap** of capacity K.
2. Iterate through every element in the input.
3. Add the element to the heap.
4. If the heap size exceeds K, remove the minimum (poll).
5. After processing all elements, the heap contains the K largest elements.
6. The heap's root (peek) is the Kth largest element.

### Variant 2: Kth Smallest / Top-K Smallest Elements
1. Create a **max-heap** of capacity K.
2. Iterate through every element in the input.
3. Add the element to the heap.
4. If the heap size exceeds K, remove the maximum (poll).
5. After processing all elements, the heap contains the K smallest elements.
6. The heap's root (peek) is the Kth smallest element.

### Variant 3: Top-K by Frequency
1. Count frequency of each element using a HashMap.
2. Create a min-heap ordered by frequency, with capacity K.
3. Iterate through the frequency map entries.
4. Add each entry to the heap; if size exceeds K, poll the least frequent.
5. The heap now holds the K most frequent elements.

## Java Template

### Top-K Largest (Min-Heap of size K)
```java
public int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll();
        }
    }
    return minHeap.peek();
}
```

### Top-K by Frequency
```java
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int num : nums) {
        freq.merge(num, 1, Integer::sum);
    }

    PriorityQueue<Map.Entry<Integer, Integer>> minHeap =
        new PriorityQueue<>(Comparator.comparingInt(Map.Entry::getValue));

    for (Map.Entry<Integer, Integer> entry : freq.entrySet()) {
        minHeap.offer(entry);
        if (minHeap.size() > k) {
            minHeap.poll();
        }
    }

    int[] result = new int[k];
    for (int i = k - 1; i >= 0; i--) {
        result[i] = minHeap.poll().getKey();
    }
    return result;
}
```

### Kth Smallest in a Sorted Matrix (Min-Heap multi-way merge)
```java
public int kthSmallest(int[][] matrix, int k) {
    int n = matrix.length;
    PriorityQueue<int[]> minHeap = new PriorityQueue<>(
        Comparator.comparingInt(a -> a[0]));

    for (int i = 0; i < Math.min(n, k); i++) {
        minHeap.offer(new int[]{matrix[i][0], i, 0});
    }

    int result = 0;
    for (int i = 0; i < k; i++) {
        int[] curr = minHeap.poll();
        result = curr[0];
        int row = curr[1], col = curr[2];
        if (col + 1 < n) {
            minHeap.offer(new int[]{matrix[row][col + 1], row, col + 1});
        }
    }
    return result;
}
```

## Dry Run Example

**Problem:** Find the 2nd largest element in `[3, 1, 5, 12, 2, 11]` using a min-heap of size K=2.

| Step | Element | Heap (min at top) | Action |
|------|---------|-------------------|--------|
| 1 | 3 | [3] | Add 3, size=1 ≤ 2 |
| 2 | 1 | [1, 3] | Add 1, size=2 ≤ 2 |
| 3 | 5 | [3, 5] | Add 5 → size=3 > 2 → poll 1 (min) |
| 4 | 12 | [5, 12] | Add 12 → size=3 > 2 → poll 3 (min) |
| 5 | 2 | [5, 12] | Add 2 → size=3 > 2 → poll 2 (min) |
| 6 | 11 | [11, 12] | Add 11 → size=3 > 2 → poll 5 (min) |

**Result:** `peek() = 11` → the 2nd largest element is **11** ✓

**Why it works:** The min-heap of size K acts as a filter. Any element smaller than the current kth largest gets evicted immediately. After processing all elements, only the K largest survive, and the smallest among them (the root) is exactly the Kth largest.

## Complexity
- **Time:** O(n log k) — each of the n elements does at most one insert and one poll, each O(log k)
- **Space:** O(k) — the heap never exceeds K elements

For QuickSelect alternative: O(n) average time, O(n²) worst case, O(1) space.

## Edge Cases
- K equals 1 — finding the maximum; a simple scan is O(n) and simpler
- K equals N — finding the minimum; again a scan suffices
- Array contains duplicates — heap handles naturally; duplicates are separate entries
- Array has exactly K elements — no polling needed, peek the root
- Negative numbers — comparator works identically
- Frequency ties — define a tiebreaker (lexicographic, insertion order) if needed

## Common Mistakes
- **Using a max-heap for top-K largest:** A max-heap of size N works but is O(n log n). The trick is a **min-heap of size K** so you discard the smallest of the candidates.
- **Confusing Kth largest vs Kth smallest:** Kth largest uses min-heap; Kth smallest uses max-heap. Think: "I keep K elements, and I evict the one that doesn't belong."
- **Off-by-one on K:** Ensure you poll when `size > k`, not `size >= k`.
- **Not handling K > N:** Some problems guarantee K ≤ N, but validate if they don't.
- **Using `Collections.reverseOrder()` unnecessarily:** For top-K largest, you want the default min-heap, not a max-heap.
- **Forgetting to count frequencies first:** In top-K frequent problems, you must build a frequency map before heapifying.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Kth Largest Element in an Array | Medium | Min-heap of size K; after processing all elements, peek gives the answer. QuickSelect is O(n) alternative. | https://leetcode.com/problems/kth-largest-element-in-an-array/ |
| 2 | Top K Frequent Elements | Medium | Build frequency map, then use min-heap of size K ordered by frequency. Bucket sort is O(n) alternative. | https://leetcode.com/problems/top-k-frequent-elements/ |
| 3 | K Closest Points to Origin | Medium | Min-heap of size K using max-heap ordered by distance; poll when size > K to keep K closest. | https://leetcode.com/problems/k-closest-points-to-origin/ |
| 4 | Sort Characters By Frequency | Medium | Count character frequencies, use max-heap to extract in descending frequency order. | https://leetcode.com/problems/sort-characters-by-frequency/ |
| 5 | Kth Smallest Element in a Sorted Matrix | Medium | Treat each row as a sorted list; use min-heap to merge and extract K times. Binary search on value range also works. | https://leetcode.com/problems/kth-smallest-element-in-a-sorted-matrix/ |
| 6 | Third Maximum Number | Easy | Track the top 3 distinct values; a min-heap of size 3 with duplicate check works cleanly. | https://leetcode.com/problems/third-maximum-number/ |
| 7 | Reorganize String | Medium | Max-heap by character frequency; greedily place the most frequent character, ensuring no two adjacent are the same. | https://leetcode.com/problems/reorganize-string/ |
| 8 | Find K Pairs with Smallest Sums | Medium | Min-heap with pairs (nums1[i] + nums2[j]); start with all (i, 0) pairs, expand j for polled entries. | https://leetcode.com/problems/find-k-pairs-with-smallest-sums/ |
| 9 | Kth Largest Element in a Stream | Easy | Maintain a min-heap of size K across add() calls; peek gives the running Kth largest. | https://leetcode.com/problems/kth-largest-element-in-a-stream/ |
| 10 | Reduce Array Size to The Half | Medium | Count frequencies, sort or heap them descending, greedily remove most frequent until half the array is gone. | https://leetcode.com/problems/reduce-array-size-to-the-half/ |
