# Pattern: Monotonic Queue (Deque)

## How to Identify This Pattern

### Trigger Words
- "sliding window maximum" or "sliding window minimum"
- "maximum/minimum in a subarray of size k"
- "shortest subarray with sum at least K"
- "constrained subsequence", "bounded difference"
- "deque", "double-ended queue"
- "window", "range query" with add/remove at both ends

### Constraint Clues
- Need to maintain the max or min within a sliding window efficiently
- Brute force window max/min would be O(nk); an O(n) solution is expected
- Elements enter from one end and leave from both ends based on position or value
- Problem combines sliding window logic with the need for quick access to extreme values
- Constraints suggest O(n) or O(n log n) — ruling out naive O(nk) approaches

### When to Use This vs Similar Patterns
| Consideration | Monotonic Queue | Monotonic Stack | Segment Tree / Sparse Table | Heap (Priority Queue) |
|---|---|---|---|---|
| Window slides? | Yes — elements expire by position | No — one-pass, no expiration | Yes — arbitrary range queries | Yes, but lazy deletion needed |
| Operation | Max/min in window | Next greater/smaller element | Range min/max/sum | Global or k-th extreme |
| Time | O(n) amortized | O(n) | O(n log n) build, O(1)–O(log n) query | O(n log n) |
| Key structure | Deque | Stack | Array-based tree | Heap |

## Core Approach

### Sliding Window Maximum / Minimum (Decreasing Deque for Max)
1. Use a deque that stores **indices**. The front of the deque always holds the index of the current window maximum.
2. Before adding index `i`, remove all indices from the **back** of the deque whose corresponding values are less than or equal to `nums[i]` — they can never be the max while `nums[i]` is in the window.
3. Add index `i` to the back of the deque.
4. Remove indices from the **front** if they fall outside the current window (`front index <= i - k`).
5. Once `i >= k - 1`, the front of the deque is the max for the current window.

### Shortest Subarray with Sum >= K (with negative numbers)
1. Compute prefix sums.
2. Use a deque to maintain a monotonically increasing sequence of prefix sum indices.
3. For each `i`, while `prefix[i] - prefix[deque.front] >= k`, update the answer and pop front (this start can't do better with a later end).
4. While `prefix[i] <= prefix[deque.back]`, pop back (the current prefix is a better starting candidate).
5. Push `i` to the back.

## Java Template

### Sliding Window Maximum
```java
public int[] maxSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    int[] result = new int[n - k + 1];
    Deque<Integer> deque = new ArrayDeque<>(); // stores indices

    for (int i = 0; i < n; i++) {
        // Remove indices outside the window
        while (!deque.isEmpty() && deque.peekFirst() <= i - k) {
            deque.pollFirst();
        }
        // Maintain decreasing order — remove smaller elements from back
        while (!deque.isEmpty() && nums[deque.peekLast()] <= nums[i]) {
            deque.pollLast();
        }
        deque.offerLast(i);

        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()];
        }
    }
    return result;
}
```

### Shortest Subarray with Sum at Least K
```java
public int shortestSubarray(int[] nums, int k) {
    int n = nums.length;
    long[] prefix = new long[n + 1];
    for (int i = 0; i < n; i++) {
        prefix[i + 1] = prefix[i] + nums[i];
    }

    Deque<Integer> deque = new ArrayDeque<>();
    int minLen = n + 1;

    for (int i = 0; i <= n; i++) {
        while (!deque.isEmpty() && prefix[i] - prefix[deque.peekFirst()] >= k) {
            minLen = Math.min(minLen, i - deque.pollFirst());
        }
        while (!deque.isEmpty() && prefix[i] <= prefix[deque.peekLast()]) {
            deque.pollLast();
        }
        deque.offerLast(i);
    }
    return minLen <= n ? minLen : -1;
}
```

## Dry Run Example

**Problem:** Sliding Window Maximum — `nums = [1, 3, -1, -3, 5, 3, 6, 7]`, `k = 3`.

| Step | i | nums[i] | Deque (indices) | Deque (values) | Remove front? | Result |
|------|---|---------|-----------------|----------------|---------------|--------|
| 0 | 0 | 1 | [0] | [1] | — | — |
| 1 | 1 | 3 | [1] | [3] | Pop 0 (3>1) | — |
| 2 | 2 | -1 | [1,2] | [3,-1] | — | [3] |
| 3 | 3 | -3 | [1,2,3] | [3,-1,-3] | — | [3,3] |
| 4 | 4 | 5 | [4] | [5] | Pop 1,2,3 (5>all); also 1 out of window | [3,3,5] |
| 5 | 5 | 3 | [4,5] | [5,3] | — | [3,3,5,5] |
| 6 | 6 | 6 | [6] | [6] | Pop 5,4 (6>all); also 4 out of window | [3,3,5,5,6] |
| 7 | 7 | 7 | [7] | [7] | Pop 6 (7>6) | [3,3,5,5,6,**7**] |

**Result:** `[3, 3, 5, 5, 6, 7]` ✓

## Complexity
- **Time:** O(n) — each element is added to and removed from the deque at most once
- **Space:** O(k) for the deque in sliding window problems; O(n) for prefix sum variants

## Edge Cases
- Window size `k = 1` — every element is its own window max/min
- Window size `k = n` — only one window; answer is the global max/min
- All elements identical — deque holds only the most recent index
- Array with negative numbers — critical for "shortest subarray with sum >= K"; prefix sums can decrease
- Single element array — result is trivially that element

## Common Mistakes
- **Removing from the wrong end:** Front removals are for expired indices (out of window). Back removals are for maintaining monotonicity. Mixing these up breaks correctness.
- **Using `<` instead of `<=` when popping from the back:** For window max with a decreasing deque, use `<=` to pop equal elements — keeping the latest index of a duplicate is better for longevity in the window.
- **Forgetting to check window validity before recording results:** Only record the answer once `i >= k - 1` (window is fully formed).
- **Not using `long` for prefix sums:** In shortest subarray problems, prefix sums can overflow `int` when array values are large.
- **Confusing deque direction:** `peekFirst()` / `pollFirst()` is the front (oldest/largest for max), `peekLast()` / `pollLast()` is the back (newest).

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Sliding Window Maximum | Hard | Decreasing deque of indices; front is always the current window max | https://leetcode.com/problems/sliding-window-maximum/ |
| 2 | Shortest Subarray with Sum at Least K | Hard | Monotonic deque on prefix sums; handles negatives unlike standard sliding window | https://leetcode.com/problems/shortest-subarray-with-sum-at-least-k/ |
| 3 | Longest Continuous Subarray With Absolute Diff <= Limit | Medium | Two deques (one for max, one for min); shrink window when max − min > limit | https://leetcode.com/problems/longest-continuous-subarray-with-absolute-diff-less-than-or-equal-to-limit/ |
| 4 | Constrained Subsequence Sum | Hard | DP with monotonic deque; dp[i] = nums[i] + max(dp[j]) for j in [i-k, i-1] | https://leetcode.com/problems/constrained-subsequence-sum/ |
| 5 | Max Value of Equation | Hard | Rewrite yi + yj + |xi - xj| as (yj - xj) + (yi + xi); deque tracks max of (yj - xj) in range | https://leetcode.com/problems/max-value-of-equation/ |
| 6 | Jump Game VI | Medium | DP + monotonic deque; dp[i] = nums[i] + max(dp[i-k..i-1]) maintained with decreasing deque | https://leetcode.com/problems/jump-game-vi/ |
| 7 | Sliding Window Median | Hard | Two heaps or sorted structure; deque approach for maintaining sorted order in window | https://leetcode.com/problems/sliding-window-median/ |
| 8 | Maximal Rectangle | Hard | Extend histogram approach row by row; use monotonic stack per row of heights | https://leetcode.com/problems/maximal-rectangle/ |
| 9 | Number of Visible People in a Queue | Hard | Monotonic decreasing stack from right; count pops as visible people | https://leetcode.com/problems/number-of-visible-people-in-a-queue/ |
| 10 | Longest Subarray of 1s After Deleting One Element | Medium | Sliding window tracking zero count; max window with at most one 0 then subtract 1 | https://leetcode.com/problems/longest-subarray-of-1s-after-deleting-one-element/ |
