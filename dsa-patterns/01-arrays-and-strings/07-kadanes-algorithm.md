# Pattern: Kadane's Algorithm

## How to Identify This Pattern

### Trigger Words
- "maximum subarray", "largest sum"
- "maximum product"
- "best time to buy and sell"
- "contiguous subarray" combined with "maximum" or "minimum"
- "circular array" with "maximum sum"
- "sum with one deletion/modification"

### Constraint Clues
- Find the **maximum (or minimum) value** over all contiguous subarrays
- Array contains both positive and negative numbers
- O(n) time expected (divide-and-conquer O(n log n) also possible but Kadane's is preferred)
- Problem is a variant of "extend current subarray or start fresh"
- Subarray must be contiguous (not subsequence)
- May involve products instead of sums (need to track both max and min)

### When to Use This vs Similar Patterns
| Consideration | Kadane's | Prefix Sum | Sliding Window | DP Table |
|---|---|---|---|---|
| Goal | Max/min sum of contiguous subarray | Count subarrays with sum = K | Longest subarray with property | Optimal substructure (general) |
| Negative numbers? | Core use case | Handled by hashmap | Breaks window logic | Handled |
| Space | O(1) | O(n) | O(1) to O(k) | O(n) or O(n²) |
| Key decision | "Extend or restart?" | "Does complement exist?" | "Shrink or expand?" | State transition |

## Core Approach

### Classic Kadane's (Maximum Subarray Sum)
1. Initialize `currentMax = nums[0]` and `globalMax = nums[0]`.
2. For each element starting from index 1:
   - `currentMax = max(nums[i], currentMax + nums[i])` — either extend the existing subarray or start a new one at `nums[i]`.
   - `globalMax = max(globalMax, currentMax)` — track the best seen so far.
3. Return `globalMax`.

### Maximum Product Variant
1. Track both `currentMax` and `currentMin` because a negative × negative = positive.
2. At each element, compute candidates: `nums[i]`, `currentMax * nums[i]`, `currentMin * nums[i]`.
3. `currentMax = max(all three)`, `currentMin = min(all three)`.
4. Update `globalMax`.

### Circular Array Variant
1. Compute `maxKadane` = standard Kadane's on the array.
2. Compute `totalSum` = sum of all elements.
3. Compute `minKadane` = minimum subarray sum (Kadane's with min instead of max).
4. The maximum circular sum = `totalSum - minKadane` (wrapping around means excluding the minimum middle).
5. Answer = `max(maxKadane, totalSum - minKadane)`.
6. **Edge case:** If all elements are negative, `totalSum - minKadane = 0`. Return `maxKadane` instead.

### With One Deletion Variant
1. Use two DP arrays:
   - `noDelete[i]` = max subarray sum ending at `i` with no deletions.
   - `oneDelete[i]` = max subarray sum ending at `i` with exactly one deletion.
2. Transitions:
   - `noDelete[i] = max(nums[i], noDelete[i-1] + nums[i])`
   - `oneDelete[i] = max(noDelete[i-1], oneDelete[i-1] + nums[i])` — either delete `nums[i]` (use `noDelete[i-1]`) or keep `nums[i]` and carry forward a previous deletion.
3. Answer = `max(all noDelete[i], all oneDelete[i])`.

## Java Template

### Classic Kadane's
```java
public int maxSubArray(int[] nums) {
    int currentMax = nums[0];
    int globalMax = nums[0];

    for (int i = 1; i < nums.length; i++) {
        currentMax = Math.max(nums[i], currentMax + nums[i]);
        globalMax = Math.max(globalMax, currentMax);
    }
    return globalMax;
}
```

### Maximum Product Subarray
```java
public int maxProduct(int[] nums) {
    int currentMax = nums[0], currentMin = nums[0], globalMax = nums[0];

    for (int i = 1; i < nums.length; i++) {
        if (nums[i] < 0) {
            int temp = currentMax;
            currentMax = currentMin;
            currentMin = temp;
        }
        currentMax = Math.max(nums[i], currentMax * nums[i]);
        currentMin = Math.min(nums[i], currentMin * nums[i]);
        globalMax = Math.max(globalMax, currentMax);
    }
    return globalMax;
}
```

### Maximum Sum Circular Subarray
```java
public int maxSubarraySumCircular(int[] nums) {
    int totalSum = 0;
    int maxKadane = nums[0], currentMax = nums[0];
    int minKadane = nums[0], currentMin = nums[0];

    totalSum = nums[0];
    for (int i = 1; i < nums.length; i++) {
        totalSum += nums[i];

        currentMax = Math.max(nums[i], currentMax + nums[i]);
        maxKadane = Math.max(maxKadane, currentMax);

        currentMin = Math.min(nums[i], currentMin + nums[i]);
        minKadane = Math.min(minKadane, currentMin);
    }

    if (maxKadane < 0) return maxKadane;
    return Math.max(maxKadane, totalSum - minKadane);
}
```

### Best Time to Buy and Sell Stock (Kadane's Perspective)
```java
public int maxProfit(int[] prices) {
    int minPrice = prices[0];
    int maxProfit = 0;

    for (int i = 1; i < prices.length; i++) {
        maxProfit = Math.max(maxProfit, prices[i] - minPrice);
        minPrice = Math.min(minPrice, prices[i]);
    }
    return maxProfit;
}
```

### Maximum Subarray Sum with One Deletion
```java
public int maximumSum(int[] arr) {
    int n = arr.length;
    int noDelete = arr[0], oneDelete = 0, globalMax = arr[0];

    for (int i = 1; i < n; i++) {
        oneDelete = Math.max(noDelete, oneDelete + arr[i]);
        noDelete = Math.max(arr[i], noDelete + arr[i]);
        globalMax = Math.max(globalMax, Math.max(noDelete, oneDelete));
    }
    return globalMax;
}
```

## Dry Run Example

**Problem:** Maximum Subarray — `nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]`

| Step | i | nums[i] | currentMax + nums[i] | nums[i] alone | currentMax (best) | globalMax |
|------|---|---------|---------------------|---------------|-------------------|-----------|
| Init | 0 | -2 | — | — | -2 | -2 |
| 1 | 1 | 1 | -2 + 1 = -1 | 1 | **1** (restart) | 1 |
| 2 | 2 | -3 | 1 + (-3) = -2 | -3 | **-2** (extend) | 1 |
| 3 | 3 | 4 | -2 + 4 = 2 | 4 | **4** (restart) | 4 |
| 4 | 4 | -1 | 4 + (-1) = 3 | -1 | **3** (extend) | 4 |
| 5 | 5 | 2 | 3 + 2 = 5 | 2 | **5** (extend) | 5 |
| 6 | 6 | 1 | 5 + 1 = 6 | 1 | **6** (extend) | **6** |
| 7 | 7 | -5 | 6 + (-5) = 1 | -5 | **1** (extend) | 6 |
| 8 | 8 | 4 | 1 + 4 = 5 | 4 | **5** (extend) | 6 |

**Result:** `globalMax = 6` from subarray `[4, -1, 2, 1]` ✓

**Problem:** Maximum Product — `nums = [2, 3, -2, 4]`

| Step | i | nums[i] | currentMax | currentMin | globalMax |
|------|---|---------|-----------|-----------|-----------|
| Init | 0 | 2 | 2 | 2 | 2 |
| 1 | 1 | 3 | max(3, 6, 6) = 6 | min(3, 6, 6) = 3 | 6 |
| 2 | 2 | -2 | max(-2, -12, -6) = -2 | min(-2, -12, -6) = -12 | 6 |
| 3 | 3 | 4 | max(4, -8, -48) = 4 | min(4, -8, -48) = -48 | 6 |

**Result:** `globalMax = 6` from subarray `[2, 3]` ✓

## Complexity
- **Time:** O(n) — single pass through the array
- **Space:** O(1) — only a constant number of variables (no extra array needed)

## Edge Cases
- All negative numbers — Kadane's correctly returns the least negative (single element)
- Single element — return that element
- All positive numbers — entire array is the answer
- Contains zero — for product variant, zero resets both max and min to 0 (handled by `max(nums[i], ...)`)
- Circular array where all elements are negative — `totalSum - minKadane = 0`, so return `maxKadane` (the least negative)
- Array of length 1 with deletion allowed — cannot delete the only element; return it
- Overflow in product variant — use `long` if products can exceed `int` range

## Common Mistakes
- **Initializing `currentMax` and `globalMax` to 0 instead of `nums[0]`:** If all elements are negative, you'd incorrectly return 0. Initialize with the first element.
- **Forgetting to track `currentMin` in product variant:** A negative × negative = positive, so a large negative min can become the new max.
- **Not handling the all-negative case in circular variant:** When all elements are negative, `totalSum - minKadane = 0`, but the answer should be the max single element, not 0.
- **Confusing subarray (contiguous) with subsequence (non-contiguous):** Kadane's only works for contiguous subarrays.
- **Applying Kadane's when prefix sum + hashmap is needed:** Kadane's finds the maximum sum, not subarrays with a specific sum.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Maximum Subarray | Medium | Classic Kadane's: extend or restart at each element | https://leetcode.com/problems/maximum-subarray/ |
| 2 | Maximum Product Subarray | Medium | Track both max and min products; negative flips them | https://leetcode.com/problems/maximum-product-subarray/ |
| 3 | Maximum Sum Circular Subarray | Medium | max(standard Kadane, totalSum − minKadane); handle all-negative edge case | https://leetcode.com/problems/maximum-sum-circular-subarray/ |
| 4 | Best Time to Buy and Sell Stock | Easy | Track min price so far; max profit = max(price − minPrice) — Kadane's on price differences | https://leetcode.com/problems/best-time-to-buy-and-sell-stock/ |
| 5 | Maximum Sum of Two Non-Overlapping Subarrays | Medium | Prefix sums + track best L-length subarray before current M-length window and vice versa | https://leetcode.com/problems/maximum-sum-of-two-non-overlapping-subarrays/ |
| 6 | Longest Turbulent Subarray | Medium | Two Kadane-style counters: one for increasing, one for decreasing; swap on alternation | https://leetcode.com/problems/longest-turbulent-subarray/ |
| 7 | Maximum Absolute Sum of Any Subarray | Medium | Answer = max(maxKadane, abs(minKadane)); compute both in one pass | https://leetcode.com/problems/maximum-absolute-sum-of-any-subarray/ |
| 8 | Maximum Subarray Sum with One Deletion | Medium | Two-state DP: noDelete and oneDelete at each position | https://leetcode.com/problems/maximum-subarray-sum-with-one-deletion/ |
| 9 | K-Concatenation Maximum Sum | Medium | For k≥2: consider max in one copy, and max wrapping with (k−2)×totalSum if totalSum > 0 | https://leetcode.com/problems/k-concatenation-maximum-sum/ |
| 10 | Shortest Subarray with Sum at Least K | Hard | Monotonic deque on prefix sums (not pure Kadane's); handles negative numbers unlike sliding window | https://leetcode.com/problems/shortest-subarray-with-sum-at-least-k/ |
