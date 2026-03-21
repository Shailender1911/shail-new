# Pattern: Prefix Sums

## How to Identify This Pattern

### Trigger Words
- "subarray sum", "range sum", "sum between indices"
- "sum equals K", "divisible by K"
- "cumulative", "running total"
- "number of subarrays with property X"
- "product of all elements except self"

### Constraint Clues
- Multiple range-sum queries on the same array → precompute prefix sums
- "Find the number of subarrays where sum equals K" → prefix sum + hashmap
- Need O(1) per query after O(n) preprocessing
- Subarray problems where sliding window doesn't work (negative numbers present)
- Problem involves counting subarrays, not finding the longest/shortest

### When to Use This vs Similar Patterns
| Consideration | Prefix Sum | Sliding Window | Kadane's |
|---|---|---|---|
| Negative numbers? | Handles fine | Breaks (can't decide when to shrink) | Handles fine |
| Goal | Count subarrays / range queries | Longest/shortest subarray | Maximum sum subarray |
| Precomputation? | Yes (prefix array) | No | No |
| Works with "sum = K"? | Yes (with hashmap) | Only if all positive | No (finds max, not exact) |

## Core Approach

### Variant 1: Basic Prefix Sum Array
1. Build an array `prefix` where `prefix[i] = nums[0] + nums[1] + ... + nums[i-1]` (with `prefix[0] = 0`).
2. Sum of subarray `[i, j]` = `prefix[j+1] - prefix[i]`.
3. Answer any range query in O(1).

### Variant 2: Prefix Sum + HashMap (Count subarrays with sum = K)
1. Maintain a running `prefixSum` as you iterate.
2. At each index, check if `prefixSum - K` exists in the hashmap.
3. If it does, add the count of `prefixSum - K` to the result (each occurrence represents a valid subarray).
4. Store `prefixSum` in the hashmap with its frequency.
5. **Key insight:** `sum(i, j) = prefix[j] - prefix[i] = K` means `prefix[i] = prefix[j] - K`.

### Variant 3: Prefix Sum for "Except Self"
1. Build a `leftProduct[i]` = product of all elements to the left of `i`.
2. Build a `rightProduct[i]` = product of all elements to the right of `i`.
3. `result[i] = leftProduct[i] * rightProduct[i]`.
4. Optimize to O(1) space by building left pass into result, then a right running product.

## Java Template

### Basic Prefix Sum
```java
public int[] buildPrefixSum(int[] nums) {
    int[] prefix = new int[nums.length + 1];
    for (int i = 0; i < nums.length; i++) {
        prefix[i + 1] = prefix[i] + nums[i];
    }
    return prefix;
}

public int rangeSum(int[] prefix, int left, int right) {
    return prefix[right + 1] - prefix[left];
}
```

### Prefix Sum + HashMap (Subarray Sum Equals K)
```java
public int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1); // empty prefix has sum 0
    int sum = 0, count = 0;

    for (int num : nums) {
        sum += num;
        count += prefixCount.getOrDefault(sum - k, 0);
        prefixCount.merge(sum, 1, Integer::sum);
    }
    return count;
}
```

### Product of Array Except Self (O(1) space)
```java
public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];

    result[0] = 1;
    for (int i = 1; i < n; i++) {
        result[i] = result[i - 1] * nums[i - 1];
    }

    int rightProduct = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= rightProduct;
        rightProduct *= nums[i];
    }
    return result;
}
```

## Dry Run Example

**Problem:** Subarray Sum Equals K — `nums = [1, 2, 3], k = 3`

| Step | Index | num | sum | sum - k | prefixCount lookup | count | prefixCount state |
|------|-------|-----|-----|---------|-------------------|-------|-------------------|
| Init | — | — | 0 | — | — | 0 | {0: 1} |
| 1 | 0 | 1 | 1 | 1-3 = -2 | -2 not found → +0 | 0 | {0:1, 1:1} |
| 2 | 1 | 2 | 3 | 3-3 = 0 | 0 found (1 time) → +1 | 1 | {0:1, 1:1, 3:1} |
| 3 | 2 | 3 | 6 | 6-3 = 3 | 3 found (1 time) → +1 | 2 | {0:1, 1:1, 3:1, 6:1} |

**Result:** `count = 2` → subarrays `[1,2]` and `[3]` both sum to 3. ✓

## Complexity
- **Time:** O(n) for building prefix sums; O(1) per range query; O(n) for hashmap variant
- **Space:** O(n) for prefix array or hashmap; O(1) for the "except self" variant with output array not counted

## Edge Cases
- Array with all zeros — every subarray sums to 0
- Negative numbers — prefix sums can decrease; hashmap approach handles this naturally
- `k = 0` — look for subarrays where prefix sum repeats (same value appears twice)
- Single element equal to `k` — the `{0: 1}` initialization handles this
- Integer overflow — for large arrays with large values, use `long` instead of `int`
- Empty array — return 0

## Common Mistakes
- **Forgetting to initialize the hashmap with `{0: 1}`:** Without this, subarrays starting from index 0 that sum to K are missed.
- **Using prefix sum when sliding window works:** If all elements are positive and you want "longest subarray with sum ≤ K", sliding window is simpler.
- **Off-by-one in range sum:** With a prefix array of size `n+1`, the sum of `[i, j]` is `prefix[j+1] - prefix[i]`, not `prefix[j] - prefix[i]`.
- **Integer overflow in product problems:** Use `long` or handle the zero case separately.
- **Storing prefix sum before checking:** You must check `sum - k` in the map **before** adding the current `sum`, otherwise you'll find the current element itself for `k = 0`.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Running Sum of 1d Array | Easy | Direct prefix sum computation; result[i] = sum of nums[0..i] | https://leetcode.com/problems/running-sum-of-1d-array/ |
| 2 | Range Sum Query - Immutable | Easy | Precompute prefix sums; answer queries in O(1) | https://leetcode.com/problems/range-sum-query-immutable/ |
| 3 | Subarray Sum Equals K | Medium | Prefix sum + hashmap; count prefix[j] - prefix[i] == k | https://leetcode.com/problems/subarray-sum-equals-k/ |
| 4 | Product of Array Except Self | Medium | Left-product and right-product passes; no division needed | https://leetcode.com/problems/product-of-array-except-self/ |
| 5 | Find Pivot Index | Easy | Pivot where leftSum == rightSum; use total sum − leftSum − nums[i] for right | https://leetcode.com/problems/find-pivot-index/ |
| 6 | Continuous Subarray Sum | Medium | Prefix sum mod k + hashmap; if same remainder seen before, subarray is divisible by k | https://leetcode.com/problems/continuous-subarray-sum/ |
| 7 | Subarray Sums Divisible by K | Medium | Count prefix sums with same remainder mod k; use modular arithmetic | https://leetcode.com/problems/subarray-sums-divisible-by-k/ |
| 8 | Random Pick with Weight | Medium | Build prefix sum of weights; binary search on prefix sum for weighted random index | https://leetcode.com/problems/random-pick-with-weight/ |
| 9 | Count Number of Nice Subarrays | Medium | Rephrase as prefix sum of odd-number counts; same technique as Subarray Sum Equals K | https://leetcode.com/problems/count-number-of-nice-subarrays/ |
| 10 | Maximum Size Subarray Sum Equals K | Medium | Prefix sum + hashmap; store first occurrence of each prefix sum to maximize length | https://leetcode.com/problems/maximum-size-subarray-sum-equals-k/ |
