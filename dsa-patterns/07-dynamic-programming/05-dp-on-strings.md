# Pattern: DP on Strings (Matching, Palindromes, Subsequences)

## How to Identify This Pattern

### Trigger Words
- "longest palindromic substring/subsequence"
- "word break", "can the string be segmented"
- "regular expression matching", "wildcard matching"
- "shortest common supersequence"
- "delete characters to make strings equal"
- "count palindromic substrings"
- "pattern matching with . and *"

### Constraint Clues
- Input is one or two strings with length ≤ 1000 (O(n²) feasible) or ≤ 5000 (still okay for O(n²))
- Need to compare characters at different positions or check substrings
- Brute force would enumerate all substrings/subsequences (exponential)
- Problem involves matching/aligning one string against another or against a pattern
- Palindrome detection on substrings → 2D boolean table

### When to Use This vs Similar Patterns
| Consideration | DP on Strings | Two Pointers | Knapsack DP | Trie/Hashing |
|---|---|---|---|---|
| Input | One or two strings | Sorted array or string ends | Items + capacity | Dictionary of words |
| State | `dp[i][j]` over string indices | Two index pointers | `dp[weight]` | Prefix tree nodes |
| Goal | Optimal alignment/matching | Palindrome check, partition | Subset selection | Prefix search, autocomplete |
| Typical problems | LCS, Edit Distance, Regex | Valid Palindrome | Subset Sum | Word Search, Autocomplete |

## Core Approach

### Palindrome DP (on a single string)
1. **Define state:** `dp[i][j]` = true if `s[i..j]` is a palindrome.
2. **Write recurrence:** `dp[i][j] = (s[i] == s[j]) && dp[i+1][j-1]`.
3. **Base cases:** `dp[i][i] = true` (single char); `dp[i][i+1] = (s[i] == s[i+1])` (two chars).
4. **Build order:** Increasing length, or bottom-up: `i` from `n-1` down to `0`, `j` from `i` to `n-1`.
5. **Answer location:** Depends on the problem — longest palindromic substring tracks max length; palindrome partitioning uses the table for cut DP.

### String Matching DP (pattern matching)
1. **Define state:** `dp[i][j]` = does `s[0..i-1]` match `p[0..j-1]`?
2. **Write recurrence:** Handle literal match, `.` (any char), `*` (zero or more of preceding), `?` (any single char).
3. **Base cases:** `dp[0][0] = true`; patterns like `a*b*` can match empty string.
4. **Build order:** Row by row, left to right.
5. **Answer location:** `dp[m][n]`.

### Word Break DP
1. **Define state:** `dp[i]` = can `s[0..i-1]` be segmented into dictionary words?
2. **Write recurrence:** `dp[i] = any j < i where dp[j] == true and s[j..i-1] is in dictionary`.
3. **Base cases:** `dp[0] = true` (empty prefix is valid).
4. **Build order:** Left to right, `i` from 1 to n.
5. **Answer location:** `dp[n]`.

## Java Template

### Longest Palindromic Substring (Expand Around Center)
```java
public String longestPalindrome(String s) {
    int start = 0, maxLen = 1;

    for (int center = 0; center < s.length(); center++) {
        // Odd-length palindromes
        int len1 = expandAroundCenter(s, center, center);
        // Even-length palindromes
        int len2 = expandAroundCenter(s, center, center + 1);

        int len = Math.max(len1, len2);
        if (len > maxLen) {
            maxLen = len;
            start = center - (len - 1) / 2;
        }
    }
    return s.substring(start, start + maxLen);
}

private int expandAroundCenter(String s, int left, int right) {
    while (left >= 0 && right < s.length() && s.charAt(left) == s.charAt(right)) {
        left--;
        right++;
    }
    return right - left - 1;
}
```

### Word Break
```java
public boolean wordBreak(String s, List<String> wordDict) {
    Set<String> dict = new HashSet<>(wordDict);
    int n = s.length();
    boolean[] dp = new boolean[n + 1];
    dp[0] = true;

    for (int i = 1; i <= n; i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && dict.contains(s.substring(j, i))) {
                dp[i] = true;
                break;
            }
        }
    }
    return dp[n];
}
```

### Regular Expression Matching
```java
public boolean isMatch(String s, String p) {
    int m = s.length(), n = p.length();
    boolean[][] dp = new boolean[m + 1][n + 1];
    dp[0][0] = true;

    // Handle patterns like a*, a*b*, a*b*c* that can match empty string
    for (int j = 2; j <= n; j++) {
        if (p.charAt(j - 1) == '*') {
            dp[0][j] = dp[0][j - 2];
        }
    }

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            char sc = s.charAt(i - 1), pc = p.charAt(j - 1);

            if (pc == '.' || pc == sc) {
                dp[i][j] = dp[i - 1][j - 1];
            } else if (pc == '*') {
                char prev = p.charAt(j - 2);
                // Zero occurrences of prev
                dp[i][j] = dp[i][j - 2];
                // One or more occurrences if prev matches current char
                if (prev == '.' || prev == sc) {
                    dp[i][j] = dp[i][j] || dp[i - 1][j];
                }
            }
        }
    }
    return dp[m][n];
}
```

### Wildcard Matching
```java
public boolean isMatch(String s, String p) {
    int m = s.length(), n = p.length();
    boolean[][] dp = new boolean[m + 1][n + 1];
    dp[0][0] = true;

    for (int j = 1; j <= n; j++) {
        if (p.charAt(j - 1) == '*') dp[0][j] = dp[0][j - 1];
    }

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            char sc = s.charAt(i - 1), pc = p.charAt(j - 1);
            if (pc == sc || pc == '?') {
                dp[i][j] = dp[i - 1][j - 1];
            } else if (pc == '*') {
                // * matches empty (dp[i][j-1]) or one more char (dp[i-1][j])
                dp[i][j] = dp[i][j - 1] || dp[i - 1][j];
            }
        }
    }
    return dp[m][n];
}
```

## Dry Run Example

**Problem:** Word Break — `s = "leetcode"`, `wordDict = ["leet", "code"]`

State: `dp[i]` = can `s[0..i-1]` be segmented? Base: `dp[0] = true`.

| i | Substring checked | dp[j] used | In dict? | dp[i] |
|---|---|---|---|---|
| 1 | s[0..0]="l" | dp[0]=T | "l"∉dict | F |
| 2 | s[0..1]="le" | dp[0]=T | "le"∉dict | F |
| 3 | s[0..2]="lee" | dp[0]=T | "lee"∉dict | F |
| 4 | s[0..3]="leet" | dp[0]=T | "leet"∈dict ✓ | **T** |
| 5 | s[0..4]="leetc", s[4..4]="c" | dp[0]=T→"leetc"∉; dp[4]=T→"c"∉ | — | F |
| 6 | s[4..5]="co" | dp[4]=T | "co"∉dict | F |
| 7 | s[4..6]="cod" | dp[4]=T | "cod"∉dict | F |
| 8 | s[4..7]="code" | dp[4]=T | "code"∈dict ✓ | **T** |

**Answer:** `dp[8] = true` ✓ ("leet" + "code")

## Complexity
- **Time:** O(n²) for palindrome DP and word break; O(m×n) for regex/wildcard matching
- **Space:** O(n²) for palindrome table; O(m×n) for matching; O(n) for word break

## Edge Cases
- Empty string → check if pattern can match empty (regex: `a*` matches empty; wildcard: `*` matches empty)
- Single character string → direct check
- Pattern is all `*` → matches everything in wildcard; in regex, depends on preceding chars
- No dictionary word matches any prefix → word break returns false
- String is itself a palindrome → entire string is the answer
- All characters identical → every substring is a palindrome
- Very long strings with short patterns → ensure you don't have unnecessary O(n³) work

## Common Mistakes
- **Regex matching: confusing `*` meaning** — In regex, `*` means "zero or more of the **preceding** element" (not wildcard). `a*` can match "", "a", "aa", etc.
- **Wildcard matching: `*` matches any sequence** — Different from regex `*`. Wildcard `*` is like regex `.*`.
- **Word Break: not using a HashSet for dictionary** — Linear search in a list makes it O(n² × L); HashSet gives O(1) lookups.
- **Palindrome substring vs subsequence confusion** — Substring must be contiguous; subsequence can skip characters. Different DP formulations.
- **Off-by-one in string DP:** `dp[i][j]` usually refers to first `i` chars of `s` and first `j` chars of `p`, so `s.charAt(i-1)` and `p.charAt(j-1)`.
- **Word Break II: need backtracking** — Can't just use boolean DP; need to track which words were used at each split point.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Longest Palindromic Substring | Medium | Expand around each center (O(n²)); or DP table `dp[i][j] = isPalindrome` | https://leetcode.com/problems/longest-palindromic-substring/ |
| 2 | Palindromic Substrings | Medium | Count all palindromic substrings; expand around center or DP; count every `dp[i][j] == true` | https://leetcode.com/problems/palindromic-substrings/ |
| 3 | Word Break | Medium | `dp[i]` = can segment `s[0..i-1]`; check all splits `j` where `dp[j]` is true and `s[j..i-1]` is in dict | https://leetcode.com/problems/word-break/ |
| 4 | Word Break II | Hard | DFS/backtracking with memoization; collect all valid segmentations, not just boolean | https://leetcode.com/problems/word-break-ii/ |
| 5 | Longest Palindromic Subsequence | Medium | `dp[i][j]` = LPS of `s[i..j]`; if ends match, `dp[i+1][j-1]+2`; else max of shrinking either end | https://leetcode.com/problems/longest-palindromic-subsequence/ |
| 6 | Delete Operation for Two Strings | Medium | Find LCS length; answer = `m + n - 2 * LCS`; minimum deletions to make strings equal | https://leetcode.com/problems/delete-operation-for-two-strings/ |
| 7 | Minimum ASCII Delete Sum for Two Strings | Medium | Like LCS but maximize ASCII sum kept (or minimize ASCII sum deleted); weight by char values | https://leetcode.com/problems/minimum-ascii-delete-sum-for-two-strings/ |
| 8 | Shortest Common Supersequence | Hard | Find LCS first, then reconstruct by weaving both strings around the LCS characters | https://leetcode.com/problems/shortest-common-supersequence/ |
| 9 | Regular Expression Matching | Hard | 2D DP handling `.` (any char) and `*` (zero or more of preceding); `*` looks back at `j-2` and optionally consumes `s[i]` | https://leetcode.com/problems/regular-expression-matching/ |
| 10 | Wildcard Matching | Hard | 2D DP: `?` matches any single char; `*` matches any sequence — `dp[i-1][j]` (extend) or `dp[i][j-1]` (empty) | https://leetcode.com/problems/wildcard-matching/ |
