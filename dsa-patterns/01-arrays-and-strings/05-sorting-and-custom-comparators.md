# Pattern: Sorting and Custom Comparators

## How to Identify This Pattern

### Trigger Words
- "merge", "overlapping intervals", "non-overlapping"
- "largest number", "custom order", "sort by"
- "Kth largest/smallest"
- "rearrange", "wiggle", "specific order"
- "meeting rooms", "schedule", "conflict"
- "gap", "spacing", "consecutive difference"

### Constraint Clues
- Problem becomes trivial or much simpler once the data is sorted
- Interval-based problems (start/end times)
- Need to assemble elements in a custom order (not natural sorting)
- "Find the Kth element" → quickselect or partial sort
- Relative ordering matters more than absolute position
- O(n log n) time is acceptable

### When to Use This vs Similar Patterns
| Consideration | Sorting | HashMap | Heap/PQ |
|---|---|---|---|
| Need all elements ordered? | Yes | No | No (only top K) |
| Stability matters? | Use stable sort or comparator | N/A | N/A |
| Space | O(1) to O(n) | O(n) | O(k) |
| Best for | Intervals, custom ordering, preprocessing | Frequency, lookup | Top-K, streaming |
| Time | O(n log n) | O(n) | O(n log k) |

## Core Approach

### Variant 1: Sort + Sweep (Merge Intervals)
1. Sort intervals by start time.
2. Initialize result with the first interval.
3. For each subsequent interval, check if it overlaps with the last interval in result.
4. If overlap, merge by extending the end time. Otherwise, add as a new interval.

### Variant 2: Custom Comparator (Largest Number)
1. Define a comparator that compares two elements based on which concatenation order produces a larger result.
2. Sort using this comparator.
3. Build the result from the sorted order.

### Variant 3: Quickselect (Kth Largest)
1. Pick a pivot element.
2. Partition the array so elements larger than pivot are on the left.
3. If pivot lands at position k-1, return it.
4. Otherwise, recurse on the appropriate partition.

### Variant 4: Bucket/Counting Sort
1. When the range of values is bounded, create buckets.
2. Place each element in its bucket.
3. Read buckets in order to produce sorted output in O(n + range).

## Java Template

### Merge Intervals
```java
public int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
    List<int[]> merged = new ArrayList<>();
    merged.add(intervals[0]);

    for (int i = 1; i < intervals.length; i++) {
        int[] last = merged.get(merged.size() - 1);
        if (intervals[i][0] <= last[1]) {
            last[1] = Math.max(last[1], intervals[i][1]);
        } else {
            merged.add(intervals[i]);
        }
    }
    return merged.toArray(new int[0][]);
}
```

### Custom Comparator (Largest Number)
```java
public String largestNumber(int[] nums) {
    String[] strs = new String[nums.length];
    for (int i = 0; i < nums.length; i++) strs[i] = String.valueOf(nums[i]);

    Arrays.sort(strs, (a, b) -> (b + a).compareTo(a + b));

    if (strs[0].equals("0")) return "0";
    StringBuilder sb = new StringBuilder();
    for (String s : strs) sb.append(s);
    return sb.toString();
}
```

### Quickselect (Kth Largest Element)
```java
public int findKthLargest(int[] nums, int k) {
    int target = nums.length - k;
    return quickselect(nums, 0, nums.length - 1, target);
}

private int quickselect(int[] nums, int lo, int hi, int target) {
    int pivot = nums[hi];
    int storeIndex = lo;

    for (int i = lo; i < hi; i++) {
        if (nums[i] <= pivot) {
            swap(nums, i, storeIndex);
            storeIndex++;
        }
    }
    swap(nums, storeIndex, hi);

    if (storeIndex == target) return nums[storeIndex];
    if (storeIndex < target) return quickselect(nums, storeIndex + 1, hi, target);
    return quickselect(nums, lo, storeIndex - 1, target);
}

private void swap(int[] nums, int i, int j) {
    int temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}
```

### Sort Characters by Frequency
```java
public String frequencySort(String s) {
    Map<Character, Integer> freq = new HashMap<>();
    for (char c : s.toCharArray()) freq.merge(c, 1, Integer::sum);

    List<Character> chars = new ArrayList<>(freq.keySet());
    chars.sort((a, b) -> freq.get(b) - freq.get(a));

    StringBuilder sb = new StringBuilder();
    for (char c : chars) {
        for (int i = 0; i < freq.get(c); i++) sb.append(c);
    }
    return sb.toString();
}
```

## Dry Run Example

**Problem:** Merge Intervals — `[[1,3], [2,6], [8,10], [15,18]]`

**Step 1: Sort by start time** → already sorted: `[[1,3], [2,6], [8,10], [15,18]]`

| Step | Current Interval | Last in Merged | Overlaps? | Action | Merged List |
|------|-----------------|---------------|-----------|--------|-------------|
| Init | — | — | — | Add [1,3] | [[1,3]] |
| 1 | [2,6] | [1,3] | 2 ≤ 3 → Yes | Extend end to max(3,6)=6 | [[1,6]] |
| 2 | [8,10] | [1,6] | 8 ≤ 6 → No | Add new interval | [[1,6],[8,10]] |
| 3 | [15,18] | [8,10] | 15 ≤ 10 → No | Add new interval | [[1,6],[8,10],[15,18]] |

**Result:** `[[1,6],[8,10],[15,18]]` ✓

**Problem:** Largest Number — `[3, 30, 34, 5, 9]`

| Comparison | a+b vs b+a | Winner |
|-----------|-----------|--------|
| "3" vs "30" | "330" vs "303" | "3" > "30" |
| "3" vs "34" | "334" vs "343" | "34" > "3" |
| "9" vs "5" | "95" vs "59" | "9" > "5" |

**Sorted order:** `["9", "5", "34", "3", "30"]` → Result: `"9534330"` ✓

## Complexity
- **Time:** O(n log n) for comparison-based sorting; O(n) for counting/bucket sort; O(n) average for quickselect
- **Space:** O(n) for merge sort (Java's Arrays.sort for objects); O(1) for in-place quicksort; O(range) for counting sort

## Edge Cases
- Empty or single-element input — already sorted; return as-is
- All intervals identical — merge into one
- Comparator overflow: `(a, b) -> a - b` can overflow for Integer.MIN_VALUE; use `Integer.compare(a, b)` instead
- All zeros in Largest Number — result should be `"0"`, not `"000"`
- Kth largest where k = 1 → maximum element; k = n → minimum element
- Negative numbers in custom comparators — string comparisons handle signs poorly; be careful

## Common Mistakes
- **Integer overflow in comparators:** `(a, b) -> a[0] - b[0]` overflows when values are near Integer.MIN_VALUE/MAX_VALUE. Use `Integer.compare(a[0], b[0])`.
- **Forgetting the `"0"` edge case in Largest Number:** After sorting, if the first element is `"0"`, all elements are 0.
- **Modifying the array during quickselect without understanding the partition:** The pivot index after partition is the element's final sorted position.
- **Not sorting a copy when the original order matters:** Some problems need the original array intact.
- **Using the wrong sort for stability:** Java's `Arrays.sort` is stable for objects but not for primitives.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Merge Intervals | Medium | Sort by start, then sweep and merge overlapping intervals | https://leetcode.com/problems/merge-intervals/ |
| 2 | Largest Number | Medium | Custom comparator: compare (a+b) vs (b+a) as strings | https://leetcode.com/problems/largest-number/ |
| 3 | Sort Characters By Frequency | Medium | Frequency map → sort characters by frequency descending | https://leetcode.com/problems/sort-characters-by-frequency/ |
| 4 | Meeting Rooms | Easy | Sort by start time; if any interval overlaps the previous, return false | https://leetcode.com/problems/meeting-rooms/ |
| 5 | Kth Largest Element in an Array | Medium | Quickselect for O(n) average; or use min-heap of size k | https://leetcode.com/problems/kth-largest-element-in-an-array/ |
| 6 | Valid Anagram | Easy | Sort both strings and compare, or use frequency counting | https://leetcode.com/problems/valid-anagram/ |
| 7 | H-Index | Medium | Sort citations descending; find largest h where citations[h-1] ≥ h | https://leetcode.com/problems/h-index/ |
| 8 | Maximum Gap | Medium | Bucket sort / radix sort for O(n); pigeonhole principle for gap lower bound | https://leetcode.com/problems/maximum-gap/ |
| 9 | Wiggle Sort II | Medium | Find median via quickselect, then interleave small and large halves | https://leetcode.com/problems/wiggle-sort-ii/ |
| 10 | Custom Sort String | Medium | Define order priority from the order string; sort with custom comparator | https://leetcode.com/problems/custom-sort-string/ |
