# Pattern: Two Pointers

## How to Identify This Pattern

### Trigger Words
- "sorted array" or "sorted in non-decreasing order"
- "pair", "triplet", "two numbers that sum to"
- "remove duplicates", "remove element in-place"
- "partition", "rearrange", "move all X to one side"
- "container", "area", "maximize/minimize distance"
- "compare from both ends"

### Constraint Clues
- Array is already sorted (or can be sorted without breaking the solution)
- Must solve in **O(1) extra space** (in-place)
- Need to find pairs/triplets satisfying a condition
- Problem involves partitioning elements by some property
- Need to shrink a search space from two ends

### When to Use This vs Similar Patterns
| Consideration | Two Pointers | Sliding Window | Binary Search |
|---|---|---|---|
| Array sorted? | Often yes | Usually no | Yes |
| Pointer movement | Both move inward or one chases the other | Right expands, left shrinks | Mid jumps to half |
| Goal | Find pair/partition | Find optimal subarray | Find target/boundary |
| Typical constraint | O(1) space | O(k) space for window state | O(log n) time |

## Core Approach

### Variant 1: Opposite-Direction (converging pointers)
1. Place pointer `left` at index `0` and `right` at index `n - 1`.
2. Evaluate the condition using elements at both pointers.
3. If the combined value is too small, move `left` right to increase it.
4. If the combined value is too large, move `right` left to decrease it.
5. If the condition is met, record the answer and move one or both pointers.
6. Stop when `left >= right`.

### Variant 2: Same-Direction (slow/fast pointers)
1. Use a `slow` pointer to track the position of the next valid element.
2. Use a `fast` pointer to scan every element.
3. When `fast` finds a valid element, copy it to `slow` and advance `slow`.
4. After the loop, `slow` indicates the new logical length.

### Variant 3: Three-Way Partition (Dutch National Flag)
1. Maintain three regions using pointers `low`, `mid`, `high`.
2. Elements before `low` are group A, between `low` and `mid` are group B, after `high` are group C.
3. Swap elements to place them in the correct region and advance the relevant pointer.

## Java Template

### Opposite-Direction (Two Sum on sorted array)
```java
public int[] twoSumSorted(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) {
            return new int[]{left, right};
        } else if (sum < target) {
            left++;
        } else {
            right--;
        }
    }
    return new int[]{-1, -1};
}
```

### Same-Direction (Remove Duplicates)
```java
public int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;
    int slow = 1;
    for (int fast = 1; fast < nums.length; fast++) {
        if (nums[fast] != nums[fast - 1]) {
            nums[slow] = nums[fast];
            slow++;
        }
    }
    return slow;
}
```

### Three-Way Partition (Sort Colors / Dutch National Flag)
```java
public void sortColors(int[] nums) {
    int low = 0, mid = 0, high = nums.length - 1;
    while (mid <= high) {
        if (nums[mid] == 0) {
            swap(nums, low, mid);
            low++;
            mid++;
        } else if (nums[mid] == 1) {
            mid++;
        } else {
            swap(nums, mid, high);
            high--;
        }
    }
}

private void swap(int[] nums, int i, int j) {
    int temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}
```

## Dry Run Example

**Problem:** Two Sum II — find two numbers in sorted array `[2, 7, 11, 15]` that sum to `9`.

| Step | left | right | nums[left] | nums[right] | Sum | Action |
|------|------|-------|------------|-------------|-----|--------|
| 1 | 0 | 3 | 2 | 15 | 17 | 17 > 9 → move right left |
| 2 | 0 | 2 | 2 | 11 | 13 | 13 > 9 → move right left |
| 3 | 0 | 1 | 2 | 7 | 9 | 9 == 9 → return [0, 1] ✓ |

**Problem:** Sort Colors — `[2, 0, 2, 1, 1, 0]`

| Step | low | mid | high | Array State | Action |
|------|-----|-----|------|-------------|--------|
| Init | 0 | 0 | 5 | [2,0,2,1,1,0] | nums[mid]=2 → swap(mid,high), high-- |
| 1 | 0 | 0 | 4 | [0,0,2,1,1,2] | nums[mid]=0 → swap(low,mid), low++, mid++ |
| 2 | 1 | 1 | 4 | [0,0,2,1,1,2] | nums[mid]=0 → swap(low,mid), low++, mid++ |
| 3 | 2 | 2 | 4 | [0,0,2,1,1,2] | nums[mid]=2 → swap(mid,high), high-- |
| 4 | 2 | 2 | 3 | [0,0,1,1,2,2] | nums[mid]=1 → mid++ |
| 5 | 2 | 3 | 3 | [0,0,1,1,2,2] | nums[mid]=1 → mid++ |
| 6 | 2 | 4 | 3 | mid > high → done | Result: [0,0,1,1,2,2] ✓ |

## Complexity
- **Time:** O(n) for two-pointer scan; O(n²) for 3Sum (one loop + two-pointer inside)
- **Space:** O(1) — all operations are in-place

## Edge Cases
- Array has 0 or 1 element — nothing to compare/partition
- All elements are identical — duplicates must be skipped properly in 3Sum-style problems
- Target cannot be achieved — ensure you return a proper "not found" indicator
- Negative numbers — sum-based problems must handle negatives correctly
- Array already in desired state — ensure pointers still terminate

## Common Mistakes
- **Off-by-one with `left < right` vs `left <= right`:** Use `<` when pointers should not overlap (pair problems), `<=` when they process the same element (partition problems).
- **Not skipping duplicates in 3Sum:** After finding a valid triplet, you must skip identical values for both `left` and `right` to avoid duplicate results.
- **Modifying the array when you shouldn't:** Some problems want the original array intact; read the problem statement carefully.
- **Forgetting 1-indexed output:** LeetCode's Two Sum II uses 1-indexed answers.
- **In Sort Colors, not leaving `mid` unchanged after swapping with `high`:** The swapped-in element from `high` hasn't been inspected yet, so don't increment `mid`.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Two Sum II - Input Array Is Sorted | Medium | Opposite-direction pointers on sorted array; shrink the wider side | https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/ |
| 2 | 3Sum | Medium | Sort + fix one element + two-pointer inner loop; skip duplicates at all three levels | https://leetcode.com/problems/3sum/ |
| 3 | Container With Most Water | Medium | Opposite-direction; always move the shorter line inward because it's the bottleneck | https://leetcode.com/problems/container-with-most-water/ |
| 4 | Remove Duplicates from Sorted Array | Easy | Slow/fast same-direction; slow marks the write position | https://leetcode.com/problems/remove-duplicates-from-sorted-array/ |
| 5 | Move Zeroes | Easy | Same-direction; slow tracks next non-zero position, swap when fast finds non-zero | https://leetcode.com/problems/move-zeroes/ |
| 6 | Sort Colors | Medium | Dutch National Flag three-way partition with low/mid/high | https://leetcode.com/problems/sort-colors/ |
| 7 | Trapping Rain Water | Hard | Opposite-direction; water at each position = min(leftMax, rightMax) − height; advance the smaller side | https://leetcode.com/problems/trapping-rain-water/ |
| 8 | Squares of a Sorted Array | Easy | Opposite-direction on sorted array with negatives; fill result array from the end | https://leetcode.com/problems/squares-of-a-sorted-array/ |
| 9 | 3Sum Closest | Medium | Sort + fix one + two-pointer; track closest sum by minimizing absolute difference | https://leetcode.com/problems/3sum-closest/ |
| 10 | Remove Element | Easy | Same-direction slow/fast; overwrite unwanted values | https://leetcode.com/problems/remove-element/ |
