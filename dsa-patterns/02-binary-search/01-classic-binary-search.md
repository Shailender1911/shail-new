# Pattern: Classic Binary Search

## How to Identify This Pattern

### Trigger Words
- "sorted array" or "sorted in non-decreasing order"
- "find", "search", "locate" a target value
- "first occurrence", "last occurrence", "leftmost", "rightmost"
- "insertion point", "insert position"
- "lower bound", "upper bound"
- "square root", "perfect square"
- "guess", "minimize search space"

### Constraint Clues
- Array is sorted (or matrix rows/columns are sorted)
- Expected time complexity is **O(log n)**
- Search space is monotonic — once a condition flips from false to true, it stays true
- Constraints show n up to 10⁵–10⁹, ruling out linear scan
- Problem asks for a boundary (first/last position where something holds)

### When to Use This vs Similar Patterns
| Consideration | Classic Binary Search | Two Pointers | Search on Answer Space |
|---|---|---|---|
| Input structure | Sorted array/matrix | Sorted or unsorted array | Implicit sorted range of answers |
| Goal | Find target or boundary in data | Find pair/partition | Find optimal value that satisfies a condition |
| Search space | Indices of the array | Indices of the array | Range of possible answers (e.g., 1 to 10⁹) |
| Typical constraint | O(log n) time | O(n) time, O(1) space | O(n log M) time |

## Core Approach

### Variant 1: Exact Match
1. Set `lo = 0`, `hi = nums.length - 1`.
2. While `lo <= hi`, compute `mid = lo + (hi - lo) / 2`.
3. If `nums[mid] == target`, return `mid`.
4. If `nums[mid] < target`, search right half: `lo = mid + 1`.
5. If `nums[mid] > target`, search left half: `hi = mid - 1`.
6. If loop ends, target not found — return `-1`.

### Variant 2: Lower Bound (First Occurrence / Insertion Point)
1. Set `lo = 0`, `hi = nums.length` (note: `hi` is past the last index).
2. While `lo < hi`, compute `mid = lo + (hi - lo) / 2`.
3. If `nums[mid] < target`, move right: `lo = mid + 1`.
4. Otherwise, shrink from right: `hi = mid`.
5. When loop ends, `lo` is the index of the first element ≥ target.

### Variant 3: Upper Bound (Last Occurrence)
1. Set `lo = 0`, `hi = nums.length`.
2. While `lo < hi`, compute `mid = lo + (hi - lo) / 2`.
3. If `nums[mid] <= target`, move right: `lo = mid + 1`.
4. Otherwise, shrink from right: `hi = mid`.
5. When loop ends, `lo - 1` is the index of the last element ≤ target.

## Java Template

### Exact Match
```java
public int binarySearch(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;
        else if (nums[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```

### Lower Bound (first index where nums[i] >= target)
```java
public int lowerBound(int[] nums, int target) {
    int lo = 0, hi = nums.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] < target) lo = mid + 1;
        else hi = mid;
    }
    return lo;
}
```

### Upper Bound (first index where nums[i] > target)
```java
public int upperBound(int[] nums, int target) {
    int lo = 0, hi = nums.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] <= target) lo = mid + 1;
        else hi = mid;
    }
    return lo;
}
```

### Integer Square Root
```java
public int mySqrt(int x) {
    if (x < 2) return x;
    long lo = 1, hi = x / 2;
    while (lo <= hi) {
        long mid = lo + (hi - lo) / 2;
        if (mid * mid == x) return (int) mid;
        else if (mid * mid < x) lo = mid + 1;
        else hi = mid - 1;
    }
    return (int) hi;
}
```

## Dry Run Example

**Problem:** Find First and Last Position of `target = 8` in `[5, 7, 7, 8, 8, 10]`.

**Step 1 — Lower Bound (find first 8):**

| Step | lo | hi | mid | nums[mid] | Action |
|------|----|----|-----|-----------|--------|
| 1 | 0 | 6 | 3 | 8 | 8 ≥ 8 → hi = 3 |
| 2 | 0 | 3 | 1 | 7 | 7 < 8 → lo = 2 |
| 3 | 2 | 3 | 2 | 7 | 7 < 8 → lo = 3 |
| 4 | 3 | 3 | — | — | lo == hi → first = 3 ✓ |

**Step 2 — Upper Bound (find last 8):**

| Step | lo | hi | mid | nums[mid] | Action |
|------|----|----|-----|-----------|--------|
| 1 | 0 | 6 | 3 | 8 | 8 ≤ 8 → lo = 4 |
| 2 | 4 | 6 | 5 | 10 | 10 > 8 → hi = 5 |
| 3 | 4 | 5 | 4 | 8 | 8 ≤ 8 → lo = 5 |
| 4 | 5 | 5 | — | — | lo == hi → last = 5 - 1 = 4 ✓ |

**Result:** `[3, 4]`

## Complexity
- **Time:** O(log n) — search space halves each iteration
- **Space:** O(1) — only a few pointer variables

## Edge Cases
- Empty array — return `-1` or insertion index `0` immediately
- Single-element array — check if it matches the target
- Target smaller than all elements — `lo` stays at `0` (lower bound returns `0`)
- Target larger than all elements — `lo` reaches `n` (lower bound returns `n`)
- All elements identical and equal to target — lower bound gives `0`, upper bound gives `n`
- Integer overflow in `mid` calculation — always use `lo + (hi - lo) / 2` instead of `(lo + hi) / 2`
- Large values in sqrt — use `long` to avoid overflow when computing `mid * mid`

## Common Mistakes
- **`lo <= hi` vs `lo < hi`:** Use `lo <= hi` for exact match (where you return inside the loop). Use `lo < hi` for boundary searches (where the answer is `lo` after the loop).
- **`hi = mid` vs `hi = mid - 1`:** When using `lo < hi`, set `hi = mid` (not `mid - 1`) to avoid skipping the answer. When using `lo <= hi`, set `hi = mid - 1` to avoid infinite loops.
- **Integer overflow in mid:** `(lo + hi) / 2` overflows for large values. Always use `lo + (hi - lo) / 2`.
- **Off-by-one in upper bound:** Upper bound finds the first element *strictly greater* than target, so the last occurrence is at `upperBound - 1`.
- **Forgetting to handle "not found":** After lower bound, verify that `nums[lo] == target` before returning; `lo` might point to a different value.
- **Initializing `hi` incorrectly:** For lower/upper bound, `hi = nums.length` (one past end). For exact match, `hi = nums.length - 1`.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Binary Search | Easy | Textbook exact-match binary search on a sorted array | https://leetcode.com/problems/binary-search/ |
| 2 | Search Insert Position | Easy | Lower bound — find the first index where nums[i] ≥ target | https://leetcode.com/problems/search-insert-position/ |
| 3 | Find First and Last Position of Element in Sorted Array | Medium | Run lower bound + upper bound; last = upperBound − 1 | https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/ |
| 4 | Sqrt(x) | Easy | Binary search on answer 1..x/2; compare mid*mid with x using long | https://leetcode.com/problems/sqrtx/ |
| 5 | Valid Perfect Square | Easy | Binary search 1..num; check if mid*mid == num exactly | https://leetcode.com/problems/valid-perfect-square/ |
| 6 | Find Smallest Letter Greater Than Target | Easy | Upper bound on a circularly sorted letter array; wrap around if needed | https://leetcode.com/problems/find-smallest-letter-greater-than-target/ |
| 7 | Count Negative Numbers in a Sorted Matrix | Easy | Binary search each row for the first negative; count = cols − index | https://leetcode.com/problems/count-negative-numbers-in-a-sorted-matrix/ |
| 8 | Guess Number Higher or Lower | Easy | Classic binary search with an API call replacing array access | https://leetcode.com/problems/guess-number-higher-or-lower/ |
| 9 | Peak Index in a Mountain Array | Medium | Binary search comparing mid with mid+1; move toward the ascending side | https://leetcode.com/problems/peak-index-in-a-mountain-array/ |
| 10 | First Bad Version | Easy | Lower bound on boolean predicate; minimize API calls | https://leetcode.com/problems/first-bad-version/ |
