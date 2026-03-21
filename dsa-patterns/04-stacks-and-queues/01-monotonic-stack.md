# Pattern: Monotonic Stack

## How to Identify This Pattern

### Trigger Words
- "next greater element", "next smaller element"
- "previous greater/smaller"
- "span", "days until warmer"
- "largest rectangle", "histogram"
- "remove digits to minimize/maximize"
- "visible", "can see"

### Constraint Clues
- Need to find the nearest larger or smaller element to the left or right
- Brute force would involve nested loops comparing every pair — O(n²)
- Each element is processed at most twice (pushed and popped) suggesting O(n)
- Problem involves a 1D array and asks about relationships between non-adjacent elements
- Answer for each element depends on the first element that breaks a monotonic trend

### When to Use This vs Similar Patterns
| Consideration | Monotonic Stack | Sliding Window | Two Pointers | Priority Queue |
|---|---|---|---|---|
| Goal | Find next/prev greater/smaller | Maintain window property | Pair/partition | Global min/max access |
| Data structure | Stack (LIFO) | Deque or pointers | Two indices | Heap |
| Key operation | Pop when monotonicity breaks | Expand/shrink window | Move pointers | Insert/extract |
| Typical time | O(n) | O(n) | O(n) | O(n log n) |

## Core Approach

### Variant 1: Next Greater Element (right-to-left scan, decreasing stack)
1. Initialize a stack and a result array filled with `-1`.
2. Iterate from right to left through the array.
3. While the stack is non-empty and `stack.peek() <= nums[i]`, pop from the stack — these elements are shadowed by `nums[i]`.
4. If the stack is non-empty after popping, `stack.peek()` is the next greater element for `nums[i]`.
5. Push `nums[i]` onto the stack.

### Variant 2: Next Greater Element (left-to-right scan, decreasing stack of indices)
1. Initialize a stack and a result array filled with `-1`.
2. Iterate left to right through the array.
3. While the stack is non-empty and `nums[stack.peek()] < nums[i]`, the answer for index `stack.pop()` is `nums[i]`.
4. Push index `i` onto the stack.

### Variant 3: Largest Rectangle in Histogram (increasing stack)
1. Maintain a stack of indices where heights are in increasing order.
2. When a bar shorter than `stack.peek()` is encountered, pop and calculate area using the popped bar's height and the width between the current index and the new stack top.
3. After processing all bars, pop remaining elements and calculate their areas.

## Java Template

### Next Greater Element (left-to-right, index-based)
```java
public int[] nextGreaterElement(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>(); // stores indices

    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            result[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return result;
}
```

### Largest Rectangle in Histogram
```java
public int largestRectangleArea(int[] heights) {
    Deque<Integer> stack = new ArrayDeque<>();
    int maxArea = 0;
    int n = heights.length;

    for (int i = 0; i <= n; i++) {
        int currHeight = (i == n) ? 0 : heights[i];
        while (!stack.isEmpty() && heights[stack.peek()] > currHeight) {
            int h = heights[stack.pop()];
            int w = stack.isEmpty() ? i : i - stack.peek() - 1;
            maxArea = Math.max(maxArea, h * w);
        }
        stack.push(i);
    }
    return maxArea;
}
```

## Dry Run Example

**Problem:** Next Greater Element — find NGE for each element in `[2, 1, 2, 4, 3]`.

Using left-to-right scan with decreasing stack of indices:

| Step | i | nums[i] | Stack (indices) | Action | Result Array |
|------|---|---------|-----------------|--------|--------------|
| Init | - | - | [] | - | [-1,-1,-1,-1,-1] |
| 1 | 0 | 2 | [] | Push 0 | [-1,-1,-1,-1,-1] |
| 2 | 1 | 1 | [0] | 1 < 2 → just push 1 | [-1,-1,-1,-1,-1] |
| 3 | 2 | 2 | [0,1] | 2 > nums[1]=1 → pop 1, result[1]=2; 2 == nums[0] → stop; push 2 | [-1,2,-1,-1,-1] |
| 4 | 3 | 4 | [0,2] | 4 > nums[2]=2 → pop 2, result[2]=4; 4 > nums[0]=2 → pop 0, result[0]=4; push 3 | [4,2,4,-1,-1] |
| 5 | 4 | 3 | [3] | 3 < nums[3]=4 → just push 4 | [4,2,4,-1,-1] |
| End | - | - | [3,4] | Remaining indices have no NGE | **[4,2,4,-1,-1]** ✓ |

**Problem:** Largest Rectangle in Histogram — `heights = [2, 1, 5, 6, 2, 3]`

| Step | i | currH | Stack | Pop & Calc | maxArea |
|------|---|-------|-------|------------|---------|
| 0 | 0 | 2 | [0] | — | 0 |
| 1 | 1 | 1 | [0] | Pop 0: h=2, w=1, area=2 | 2 |
| 1 | 1 | 1 | [1] | Push 1 | 2 |
| 2 | 2 | 5 | [1,2] | — | 2 |
| 3 | 3 | 6 | [1,2,3] | — | 2 |
| 4 | 4 | 2 | [1,2,3] | Pop 3: h=6, w=1, area=6; Pop 2: h=5, w=2, area=10 | 10 |
| 4 | 4 | 2 | [1,4] | Push 4 | 10 |
| 5 | 5 | 3 | [1,4,5] | — | 10 |
| 6 | 6 | 0 | [1,4,5] | Pop 5: h=3, w=1, area=3; Pop 4: h=2, w=3, area=6; Pop 1: h=1, w=6, area=6 | **10** ✓ |

## Complexity
- **Time:** O(n) — each element is pushed and popped at most once
- **Space:** O(n) — stack can hold up to n elements in the worst case

## Edge Cases
- Array with a single element — the answer is always -1 (no next greater exists)
- Strictly increasing array — stack empties completely at each step; every element except the last has an NGE
- Strictly decreasing array — nothing is popped until the end; all answers are -1 (for next greater)
- All elements identical — no element is strictly greater; all answers are -1
- Circular array variant (Next Greater Element II) — iterate `2n` times using `i % n`

## Common Mistakes
- **Using a max-stack instead of a min-stack (or vice versa):** For *next greater*, the stack must be monotonically *decreasing* from bottom to top. For *next smaller*, it must be *increasing*.
- **Storing values instead of indices:** Most problems need indices for width calculations or result placement. Always store indices unless the problem explicitly only needs values.
- **Off-by-one in width calculation for histogram:** Width is `i - stack.peek() - 1` when the stack is non-empty, or `i` when it's empty (the popped bar extends all the way to the left).
- **Forgetting the sentinel at the end:** In histogram problems, append a bar of height 0 (or iterate to `n`) to flush remaining elements from the stack.
- **Confusing strict vs non-strict inequality:** `<` vs `<=` in the while condition changes whether equal elements are treated as "greater" — read the problem carefully.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Next Greater Element I | Easy | Use a map to store NGE from nums2, then look up each element of nums1 | https://leetcode.com/problems/next-greater-element-i/ |
| 2 | Next Greater Element II | Medium | Circular array — iterate 2n times using modulo; decreasing stack of indices | https://leetcode.com/problems/next-greater-element-ii/ |
| 3 | Daily Temperatures | Medium | Classic NGE on indices; stack stores indices of days waiting for a warmer day | https://leetcode.com/problems/daily-temperatures/ |
| 4 | Largest Rectangle in Histogram | Hard | Increasing stack; pop when a shorter bar arrives and compute area using width span | https://leetcode.com/problems/largest-rectangle-in-histogram/ |
| 5 | Online Stock Span | Medium | Monotonic decreasing stack tracking cumulative span; pop and accumulate spans of smaller prices | https://leetcode.com/problems/online-stock-span/ |
| 6 | Asteroid Collision | Medium | Stack simulation; push positive, pop when negative asteroid meets smaller positive on top | https://leetcode.com/problems/asteroid-collision/ |
| 7 | 132 Pattern | Medium | Reverse iterate with decreasing stack; track the "2" (third) as the max popped value | https://leetcode.com/problems/132-pattern/ |
| 8 | Remove K Digits | Medium | Increasing stack; remove a digit when it's greater than the next one to minimize the number | https://leetcode.com/problems/remove-k-digits/ |
| 9 | Sum of Subarray Minimums | Medium | Find previous-less and next-less-or-equal boundaries using two monotonic stacks; contribution technique | https://leetcode.com/problems/sum-of-subarray-minimums/ |
| 10 | Trapping Rain Water | Hard | Two approaches: monotonic stack (process horizontal layers) or two-pointer; stack tracks bars forming containers | https://leetcode.com/problems/trapping-rain-water/ |
