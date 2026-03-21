# Pattern: Hashmaps and Counting

## How to Identify This Pattern

### Trigger Words
- "find", "lookup", "check if exists"
- "frequency", "count occurrences", "most common"
- "group", "categorize", "anagram"
- "duplicate", "unique", "first non-repeating"
- "two numbers that sum to", "pair with property"
- "mapping", "bijection", "isomorphic"

### Constraint Clues
- Need O(1) lookup by value (not index)
- Counting frequency of elements, characters, or words
- Grouping elements by some computed key (sorted form, pattern, etc.)
- Finding pairs/groups based on a relationship
- Problem asks "does X exist" repeatedly → hashset
- Problem asks "how many times does X appear" → hashmap

### When to Use This vs Similar Patterns
| Consideration | HashMap | Sorting | Two Pointers |
|---|---|---|---|
| Time for lookup | O(1) average | O(log n) with binary search | O(n) scan |
| Space | O(n) | O(1) or O(n) | O(1) |
| Preserves order? | No (unless LinkedHashMap) | No | Yes |
| Best for | Existence check, frequency, grouping | Finding order, duplicates | Sorted array pair-finding |
| Typical trade-off | Speed for space | In-place but slower | Requires sorted input |

## Core Approach

### Variant 1: Value Lookup (Two Sum pattern)
1. Iterate through the array.
2. For each element, compute the complement (target − current).
3. Check if the complement exists in the hashmap.
4. If yes, return the pair. If no, store the current element and its index.

### Variant 2: Frequency Counting
1. Build a frequency map: `element → count`.
2. Use the frequency map to answer questions (most frequent, unique, duplicates).
3. Optionally sort by frequency using a bucket sort or priority queue.

### Variant 3: Grouping by Key
1. Compute a canonical key for each element (e.g., sorted characters for anagrams).
2. Group elements into a `Map<Key, List<Element>>`.
3. Return the grouped values.

### Variant 4: Pattern Matching (Isomorphic / Word Pattern)
1. Build two bidirectional mappings: `a → b` and `b → a`.
2. For each pair, verify consistency in both directions.
3. If any conflict, return false.

## Java Template

### Two Sum (Value Lookup)
```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (seen.containsKey(complement)) {
            return new int[]{seen.get(complement), i};
        }
        seen.put(nums[i], i);
    }
    return new int[]{-1, -1};
}
```

### Frequency Counting (Top K Frequent)
```java
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int num : nums) freq.merge(num, 1, Integer::sum);

    // Bucket sort: index = frequency, value = list of elements
    List<Integer>[] buckets = new List[nums.length + 1];
    for (var entry : freq.entrySet()) {
        int f = entry.getValue();
        if (buckets[f] == null) buckets[f] = new ArrayList<>();
        buckets[f].add(entry.getKey());
    }

    int[] result = new int[k];
    int idx = 0;
    for (int i = buckets.length - 1; i >= 0 && idx < k; i--) {
        if (buckets[i] != null) {
            for (int num : buckets[i]) {
                if (idx >= k) break;
                result[idx++] = num;
            }
        }
    }
    return result;
}
```

### Grouping (Group Anagrams)
```java
public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> groups = new HashMap<>();
    for (String s : strs) {
        char[] chars = s.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars);
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(groups.values());
}
```

### Pattern Matching (Isomorphic Strings)
```java
public boolean isIsomorphic(String s, String t) {
    if (s.length() != t.length()) return false;
    Map<Character, Character> sToT = new HashMap<>();
    Map<Character, Character> tToS = new HashMap<>();

    for (int i = 0; i < s.length(); i++) {
        char sc = s.charAt(i), tc = t.charAt(i);
        if (sToT.containsKey(sc) && sToT.get(sc) != tc) return false;
        if (tToS.containsKey(tc) && tToS.get(tc) != sc) return false;
        sToT.put(sc, tc);
        tToS.put(tc, sc);
    }
    return true;
}
```

## Dry Run Example

**Problem:** Two Sum — `nums = [2, 7, 11, 15], target = 9`

| Step | Index | num | complement (9 - num) | seen map | Found? | Action |
|------|-------|-----|---------------------|----------|--------|--------|
| 1 | 0 | 2 | 7 | {} | No | Add {2: 0} |
| 2 | 1 | 7 | 2 | {2: 0} | Yes! 2 at index 0 | Return [0, 1] ✓ |

**Problem:** Group Anagrams — `["eat", "tea", "tan", "ate", "nat", "bat"]`

| Step | Word | Sorted Key | Groups Map |
|------|------|-----------|------------|
| 1 | "eat" | "aet" | {"aet": ["eat"]} |
| 2 | "tea" | "aet" | {"aet": ["eat","tea"]} |
| 3 | "tan" | "ant" | {"aet": ["eat","tea"], "ant": ["tan"]} |
| 4 | "ate" | "aet" | {"aet": ["eat","tea","ate"], "ant": ["tan"]} |
| 5 | "nat" | "ant" | {"aet": ["eat","tea","ate"], "ant": ["tan","nat"]} |
| 6 | "bat" | "abt" | {"aet": ["eat","tea","ate"], "ant": ["tan","nat"], "abt": ["bat"]} |

**Result:** `[["eat","tea","ate"], ["tan","nat"], ["bat"]]` ✓

## Complexity
- **Time:** O(n) for single-pass hashmap problems; O(n·k·log k) for group anagrams (k = max string length due to sorting)
- **Space:** O(n) for storing elements in the map

## Edge Cases
- Empty array/string — return empty result
- Single element — may be the answer itself (Two Sum with target = 2 * element is invalid since you can't reuse)
- All elements identical — duplicates collapse in a HashSet but not in a frequency HashMap
- Very large input — hashmap collisions degrade to O(n) per operation in worst case
- Negative numbers / zero — hashmaps handle these naturally
- Case sensitivity in strings — clarify if 'A' and 'a' are the same character

## Common Mistakes
- **Reusing the same element in Two Sum:** Store the element in the map *after* checking the complement, not before.
- **Using sorted string as key when strings are very long:** Consider using a character-count key (`"a1b2c3"`) instead of sorting for O(k) vs O(k log k).
- **Forgetting bidirectional mapping in isomorphic problems:** Checking only `s → t` misses cases where two characters in `t` map to the same character.
- **Not handling `null` keys or values:** Java HashMaps allow `null` keys, but this can cause unexpected behavior.
- **Confusing HashSet (existence) with HashMap (key-value):** Use HashSet when you only care about "is it there?", HashMap when you need associated data.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Two Sum | Easy | Store complement in map; one-pass lookup | https://leetcode.com/problems/two-sum/ |
| 2 | Group Anagrams | Medium | Sort each word as key; group into map of lists | https://leetcode.com/problems/group-anagrams/ |
| 3 | Valid Anagram | Easy | Compare character frequency maps of both strings | https://leetcode.com/problems/valid-anagram/ |
| 4 | Contains Duplicate | Easy | HashSet — if add returns false, duplicate found | https://leetcode.com/problems/contains-duplicate/ |
| 5 | Longest Consecutive Sequence | Medium | HashSet for O(1) lookup; only start counting from sequence heads (num-1 not in set) | https://leetcode.com/problems/longest-consecutive-sequence/ |
| 6 | Top K Frequent Elements | Medium | Frequency map + bucket sort by frequency for O(n) | https://leetcode.com/problems/top-k-frequent-elements/ |
| 7 | Encode and Decode TinyURL | Medium | Two hashmaps: long→short and short→long; generate unique short codes | https://leetcode.com/problems/encode-and-decode-tinyurl/ |
| 8 | Isomorphic Strings | Easy | Bidirectional char mapping; both directions must be consistent | https://leetcode.com/problems/isomorphic-strings/ |
| 9 | Word Pattern | Easy | Same as isomorphic but maps char↔word; split string into words first | https://leetcode.com/problems/word-pattern/ |
| 10 | First Unique Character in a String | Easy | Frequency map pass, then scan string for first char with count 1 | https://leetcode.com/problems/first-unique-character-in-a-string/ |
