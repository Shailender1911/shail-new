# Pattern: Sliding Window

## How to Identify This Pattern

### Trigger Words
- "contiguous subarray" or "substring"
- "maximum/minimum length", "longest", "shortest"
- "at most K distinct", "at most K replacements"
- "window", "consecutive"
- "sum ≥ target", "product < target"
- "permutation", "anagram"

### Constraint Clues
- Looking for a **contiguous** sequence (subarray or substring)
- Need to optimize over all possible subarrays of a given/variable size
- Constraint involves a frequency count, sum, or product within a range
- Problem can be rephrased as: "find the best window where some invariant holds"
- O(n) or O(n log n) time expected (brute force would be O(n²) or O(n³))

### When to Use This vs Similar Patterns
| Consideration | Sliding Window | Two Pointers | Prefix Sum |
|---|---|---|---|
| Subarray contiguous? | Always yes | Not necessarily | Yes |
| Window state tracked? | Yes (freq map, sum, etc.) | Minimal state | Precomputed array |
| Pointer movement | Right expands, left contracts | Both move freely | Random access via indices |
| Typical question | "longest/shortest subarray with property X" | "find pair summing to target" | "sum of range [i, j]" |

## Core Approach

### Variant 1: Fixed-Size Window
1. Compute the result for the first window of size `k`.
2. Slide the window one position right: add the new element, remove the leftmost element.
3. Update the answer at each position.
4. Repeat until the window reaches the end.

### Variant 2: Variable-Size Window (Expand + Shrink)
1. Use two pointers `left` and `right`, both starting at `0`.
2. Expand the window by moving `right` and updating window state.
3. When the window **violates** the constraint, shrink from `left` until valid again.
4. At each valid state, update the answer (max length, min length, etc.).
5. Continue until `right` reaches the end of the array.

### Variant 3: Exact-Match Window (Permutation/Anagram)
1. Build a frequency map of the target pattern.
2. Slide a window of exactly `pattern.length` over the source string.
3. Maintain a running frequency map of the window.
4. Compare the two maps (or use a "matches" counter) at each position.

## Java Template

### Fixed-Size Window
```java
public double findMaxAverage(int[] nums, int k) {
    int windowSum = 0;
    for (int i = 0; i < k; i++) {
        windowSum += nums[i];
    }
    int maxSum = windowSum;
    for (int i = k; i < nums.length; i++) {
        windowSum += nums[i] - nums[i - k];
        maxSum = Math.max(maxSum, windowSum);
    }
    return (double) maxSum / k;
}
```

### Variable-Size Window (Shrinkable)
```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> freq = new HashMap<>();
    int left = 0, maxLen = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        freq.merge(c, 1, Integer::sum);

        while (freq.get(c) > 1) {
            char leftChar = s.charAt(left);
            freq.merge(leftChar, -1, Integer::sum);
            left++;
        }

        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

### Minimum Window Substring (Shrink-to-Optimize)
```java
public String minWindow(String s, String t) {
    Map<Character, Integer> need = new HashMap<>();
    for (char c : t.toCharArray()) need.merge(c, 1, Integer::sum);

    int left = 0, formed = 0, required = need.size();
    int[] result = {-1, 0, 0}; // {length, start, end}
    Map<Character, Integer> window = new HashMap<>();

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        window.merge(c, 1, Integer::sum);

        if (need.containsKey(c) && window.get(c).intValue() == need.get(c).intValue()) {
            formed++;
        }

        while (formed == required) {
            if (result[0] == -1 || right - left + 1 < result[0]) {
                result[0] = right - left + 1;
                result[1] = left;
                result[2] = right;
            }
            char leftChar = s.charAt(left);
            window.merge(leftChar, -1, Integer::sum);
            if (need.containsKey(leftChar) && window.get(leftChar) < need.get(leftChar)) {
                formed--;
            }
            left++;
        }
    }
    return result[0] == -1 ? "" : s.substring(result[1], result[2] + 1);
}
```

## Dry Run Example

**Problem:** Longest Substring Without Repeating Characters — `s = "abcabcbb"`

| Step | right | char | freq map | left | Window | Length | maxLen |
|------|-------|------|----------|------|--------|--------|--------|
| 1 | 0 | a | {a:1} | 0 | "a" | 1 | 1 |
| 2 | 1 | b | {a:1,b:1} | 0 | "ab" | 2 | 2 |
| 3 | 2 | c | {a:1,b:1,c:1} | 0 | "abc" | 3 | 3 |
| 4 | 3 | a | {a:2,b:1,c:1} | 0 | duplicate! shrink → | — | — |
| — | — | — | shrink: remove a → {a:1,b:1,c:1} | 1 | "bca" | 3 | 3 |
| 5 | 4 | b | {a:1,b:2,c:1} | 1 | duplicate! shrink → | — | — |
| — | — | — | shrink: remove b → {a:1,b:1,c:1} | 2 | "cab" | 3 | 3 |
| 6 | 5 | c | {a:1,b:1,c:2} | 2 | shrink → remove c → {b:1,c:1} | — | — |
| — | — | — | | 3 | "abc" wait, "bc" actually | 2→"bc"+c="bcc"→ shrink | 3 |
| 7 | 6 | b | duplicate → shrink | 5 | "b" | 1 | 3 |
| 8 | 7 | b | duplicate → shrink | 7 | "b" | 1 | 3 |

**Result:** `maxLen = 3` (substring `"abc"`)

## Complexity
- **Time:** O(n) — each element is added and removed from the window at most once
- **Space:** O(k) — where k is the size of the character set or window state (e.g., O(26) for lowercase English letters)

## Edge Cases
- Empty string or array — return 0 or empty
- `k` larger than array length — invalid input for fixed-size window
- All characters identical — window never grows past size 1 (for distinct-char problems)
- Single element — window of size 1 is always valid
- Entire string is the answer — ensure you capture windows that span the full input

## Common Mistakes
- **Forgetting to shrink the window:** The `while` loop for shrinking must run until the invariant is restored, not just once.
- **Updating the answer at the wrong time:** For "longest" problems, update after expanding; for "shortest" problems, update during shrinking.
- **Off-by-one in window size:** Window size is `right - left + 1`, not `right - left`.
- **Using `==` instead of `.equals()` for Integer comparisons:** In Java, cached Integer values (-128 to 127) work with `==`, but larger values don't. Use `.intValue()` or `.equals()`.
- **Not handling the "formed" counter correctly in minimum-window problems:** Decrement `formed` only when the count drops **below** the required count, not on every removal.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Maximum Average Subarray I | Easy | Fixed-size window of k; slide and track max sum | https://leetcode.com/problems/maximum-average-subarray-i/ |
| 2 | Longest Substring Without Repeating Characters | Medium | Variable window; shrink when a duplicate char enters | https://leetcode.com/problems/longest-substring-without-repeating-characters/ |
| 3 | Minimum Window Substring | Hard | Shrink-to-optimize; track "formed" count vs required unique chars | https://leetcode.com/problems/minimum-window-substring/ |
| 4 | Permutation in String | Medium | Fixed-size window of len(s1); compare frequency maps | https://leetcode.com/problems/permutation-in-string/ |
| 5 | Longest Repeating Character Replacement | Medium | Variable window; valid when window_size − max_freq ≤ k | https://leetcode.com/problems/longest-repeating-character-replacement/ |
| 6 | Max Consecutive Ones III | Medium | Variable window; treat flipping 0→1 as budget k; shrink when zeros exceed k | https://leetcode.com/problems/max-consecutive-ones-iii/ |
| 7 | Fruit Into Baskets | Medium | Longest subarray with at most 2 distinct values; variable window | https://leetcode.com/problems/fruit-into-baskets/ |
| 8 | Minimum Size Subarray Sum | Medium | Variable window; shrink when sum ≥ target; track min length | https://leetcode.com/problems/minimum-size-subarray-sum/ |
| 9 | Grumpy Bookstore Owner | Medium | Fixed window of size k over a "grumpy" mask; maximize rescued customers | https://leetcode.com/problems/grumpy-bookstore-owner/ |
| 10 | Subarray Product Less Than K | Medium | Variable window; shrink when product ≥ k; count subarrays ending at right | https://leetcode.com/problems/subarray-product-less-than-k/ |
