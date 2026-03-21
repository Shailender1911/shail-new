# Pattern: Rotated and Bitonic Arrays

## How to Identify This Pattern

### Trigger Words
- "rotated sorted array"
- "shifted", "pivoted", "cyclically sorted"
- "mountain array", "bitonic sequence"
- "peak element", "local maximum"
- "find minimum in rotated"
- "search in rotated"
- "single element in a sorted array"

### Constraint Clues
- Array was originally sorted but has been rotated at an unknown pivot
- Array first increases then decreases (or vice versa) — bitonic/mountain shape
- Expected time complexity is **O(log n)**
- Array has a structural invariant: at least one half around `mid` is always sorted
- Duplicates may or may not be present (affects worst-case complexity)

### When to Use This vs Similar Patterns
| Consideration | Rotated / Bitonic Search | Classic Binary Search | Linear Scan |
|---|---|---|---|
| Array structure | Sorted but rotated / mountain-shaped | Fully sorted | No structure |
| Key invariant | One half around mid is always sorted | Entire array sorted | None |
| Duplicates impact | May degrade to O(n) worst case | No impact | N/A |
| Complexity | O(log n) average, O(n) worst with dupes | O(log n) always | O(n) |

## Core Approach

### Variant 1: Search in Rotated Sorted Array (no duplicates)
1. Set `lo = 0`, `hi = nums.length - 1`.
2. While `lo <= hi`, compute `mid`.
3. If `nums[mid] == target`, return `mid`.
4. Determine which half is sorted:
   - If `nums[lo] <= nums[mid]` → left half `[lo..mid]` is sorted.
     - If `nums[lo] <= target < nums[mid]` → target is in left half → `hi = mid - 1`.
     - Otherwise → `lo = mid + 1`.
   - Else → right half `[mid..hi]` is sorted.
     - If `nums[mid] < target <= nums[hi]` → target is in right half → `lo = mid + 1`.
     - Otherwise → `hi = mid - 1`.

### Variant 2: Find Minimum in Rotated Sorted Array
1. Set `lo = 0`, `hi = nums.length - 1`.
2. While `lo < hi`:
   - If `nums[mid] > nums[hi]` → min is in `[mid+1..hi]` → `lo = mid + 1`.
   - Else → min is in `[lo..mid]` → `hi = mid`.
3. Return `nums[lo]`.

### Variant 3: Find Peak Element (Bitonic / Mountain)
1. Set `lo = 0`, `hi = nums.length - 1`.
2. While `lo < hi`:
   - If `nums[mid] < nums[mid + 1]` → we are on the ascending slope → peak is to the right → `lo = mid + 1`.
   - Else → we are on the descending slope or at the peak → `hi = mid`.
3. Return `lo` (index of the peak).

## Java Template

### Search in Rotated Sorted Array
```java
public int search(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;

        if (nums[lo] <= nums[mid]) {
            if (nums[lo] <= target && target < nums[mid]) {
                hi = mid - 1;
            } else {
                lo = mid + 1;
            }
        } else {
            if (nums[mid] < target && target <= nums[hi]) {
                lo = mid + 1;
            } else {
                hi = mid - 1;
            }
        }
    }
    return -1;
}
```

### Search in Rotated Sorted Array II (with duplicates)
```java
public boolean search(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return true;

        if (nums[lo] == nums[mid] && nums[mid] == nums[hi]) {
            lo++;
            hi--;
        } else if (nums[lo] <= nums[mid]) {
            if (nums[lo] <= target && target < nums[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else {
            if (nums[mid] < target && target <= nums[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    return false;
}
```

### Find Minimum in Rotated Sorted Array
```java
public int findMin(int[] nums) {
    int lo = 0, hi = nums.length - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] > nums[hi]) {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    return nums[lo];
}
```

### Find Peak Element
```java
public int findPeakElement(int[] nums) {
    int lo = 0, hi = nums.length - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] < nums[mid + 1]) {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    return lo;
}
```

## Dry Run Example

**Problem:** Search in Rotated Sorted Array — `nums = [4, 5, 6, 7, 0, 1, 2]`, `target = 0`.

| Step | lo | hi | mid | nums[mid] | Sorted Half | Target in Sorted Half? | Action |
|------|----|----|-----|-----------|------------|----------------------|--------|
| 1 | 0 | 6 | 3 | 7 | Left [4,5,6,7] sorted | 4 ≤ 0 < 7? No | lo = 4 |
| 2 | 4 | 6 | 5 | 1 | Right [1,2] sorted | 1 < 0 ≤ 2? No | hi = 4 |
| 3 | 4 | 4 | 4 | 0 | nums[mid] == target | — | return 4 ✓ |

**Problem:** Find Minimum — `nums = [3, 4, 5, 1, 2]`.

| Step | lo | hi | mid | nums[mid] | nums[hi] | Action |
|------|----|----|-----|-----------|----------|--------|
| 1 | 0 | 4 | 2 | 5 | 2 | 5 > 2 → lo = 3 |
| 2 | 3 | 4 | 3 | 1 | 2 | 1 ≤ 2 → hi = 3 |
| 3 | 3 | 3 | — | — | — | lo == hi → min = nums[3] = 1 ✓ |

**Problem:** Find Peak — `nums = [1, 2, 3, 1]`.

| Step | lo | hi | mid | nums[mid] | nums[mid+1] | Action |
|------|----|----|-----|-----------|------------|--------|
| 1 | 0 | 3 | 1 | 2 | 3 | 2 < 3 → ascending → lo = 2 |
| 2 | 2 | 3 | 2 | 3 | 1 | 3 > 1 → at/past peak → hi = 2 |
| 3 | 2 | 2 | — | — | — | lo == hi → peak index = 2 ✓ |

## Complexity
- **Time:**
  - Without duplicates: O(log n) — search space halves each step
  - With duplicates: O(log n) average, **O(n) worst case** — when all elements are identical except one, `lo++; hi--` degrades to linear
- **Space:** O(1) — iterative binary search with constant extra variables

## Edge Cases
- Array not rotated at all (pivot = 0) — works correctly since the entire array is one sorted half
- Array of length 1 — single element is both the min and the peak
- Array of length 2 — ensure `mid + 1` doesn't go out of bounds in peak finding
- All elements identical (with duplicates variant) — worst case O(n); cannot determine sorted half
- Target not in array — must return `-1` or `false`
- Rotation at the very end — `[2, 1]` — minimum is at index 1
- Mountain array with peak at the first or last index (boundary peaks)

## Common Mistakes
- **Using `nums[lo] < nums[mid]` instead of `nums[lo] <= nums[mid]`:** The `=` handles the case where `lo == mid` (two-element subarray). Without it, the wrong half gets chosen.
- **Not handling duplicates:** When `nums[lo] == nums[mid] == nums[hi]`, you cannot determine the sorted half. Shrink both ends with `lo++; hi--`.
- **Comparing mid with lo instead of hi for findMin:** Comparing `nums[mid]` with `nums[hi]` is safer because `hi` is always in the right portion. Using `nums[lo]` requires extra care when the array is not rotated.
- **Out-of-bounds with `mid + 1`:** In peak finding, `nums[mid + 1]` is safe only when `lo < hi` (guarantees `mid < hi`). Using `lo <= hi` risks accessing `nums[n]`.
- **Returning `mid` instead of `lo` after the loop:** In boundary-style searches (`lo < hi`), the answer is `lo` (or `hi`) when the loop exits, not `mid`.
- **Forgetting that "rotated 0 times" is a valid input:** The array may be fully sorted — your algorithm must handle this without special-casing.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Search in Rotated Sorted Array | Medium | Identify the sorted half, check if target falls in it, search accordingly | https://leetcode.com/problems/search-in-rotated-sorted-array/ |
| 2 | Search in Rotated Sorted Array II | Medium | Same as above but duplicates force lo++/hi-- when both ends match mid | https://leetcode.com/problems/search-in-rotated-sorted-array-ii/ |
| 3 | Find Minimum in Rotated Sorted Array | Medium | Compare nums[mid] with nums[hi]; if mid > hi, min is to the right | https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/ |
| 4 | Find Minimum in Rotated Sorted Array II | Hard | Same as above but lo++/hi-- on duplicates; O(n) worst case | https://leetcode.com/problems/find-minimum-in-rotated-sorted-array-ii/ |
| 5 | Find Peak Element | Medium | Compare nums[mid] with nums[mid+1]; climb toward the higher neighbor | https://leetcode.com/problems/find-peak-element/ |
| 6 | Search a 2D Matrix | Medium | Treat m×n matrix as a virtual sorted array; index = mid/n, mid%n | https://leetcode.com/problems/search-a-2d-matrix/ |
| 7 | Search a 2D Matrix II | Medium | Start from top-right corner; go left if too big, down if too small — O(m+n) | https://leetcode.com/problems/search-a-2d-matrix-ii/ |
| 8 | Single Element in a Sorted Array | Medium | Binary search on even indices; compare pairs to determine which half has the single | https://leetcode.com/problems/single-element-in-a-sorted-array/ |
| 9 | Median of Two Sorted Arrays | Hard | Binary search on shorter array partition; ensure left-max ≤ right-min for both arrays | https://leetcode.com/problems/median-of-two-sorted-arrays/ |
| 10 | Find in Mountain Array | Hard | Find peak via binary search, then binary search both halves (ascending + descending) | https://leetcode.com/problems/find-in-mountain-array/ |
