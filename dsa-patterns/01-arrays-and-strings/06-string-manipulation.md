# Pattern: String Manipulation

## How to Identify This Pattern

### Trigger Words
- "reverse", "reverse words", "rotate"
- "convert", "parse", "extract"
- "string to integer", "integer to string"
- "compare", "prefix", "suffix"
- "zigzag", "spiral", "reshape"
- "add", "multiply" (on string-represented numbers)
- "Roman", "binary", "encode/decode"

### Constraint Clues
- Input/output are strings with specific formatting rules
- Need to process characters one at a time with state tracking
- Problem involves mathematical operations on numbers too large for `int`/`long`
- Must handle leading zeros, signs, whitespace, overflow
- Pattern involves character-by-character building or transformation
- String comparison with version numbers, custom rules

### When to Use This vs Similar Patterns
| Consideration | String Manipulation | Two Pointers | HashMap |
|---|---|---|---|
| Core operation | Build/transform strings char by char | In-place reorder | Lookup/frequency |
| State tracking | Often complex (signs, carry, position) | Minimal | Frequency map |
| Typical output | A new string | Modified array/indices | Count or boolean |
| Common pitfall | Edge cases (empty, overflow, special chars) | Off-by-one | Missing initialization |

## Core Approach

### Variant 1: Reverse / Rearrange
1. Convert string to character array (or use StringBuilder).
2. Apply reversal logic (full reverse, word-level reverse, etc.).
3. Handle extra whitespace, leading/trailing spaces.

### Variant 2: Parsing / Conversion (atoi, Roman)
1. Skip leading whitespace.
2. Determine sign (+/-).
3. Process digits one at a time, building the result.
4. Check for overflow at each step **before** adding the next digit.
5. Stop at the first non-digit character.

### Variant 3: Arithmetic on String Numbers
1. Process both strings from the rightmost digit (least significant).
2. Maintain a carry variable.
3. At each position: `sum = digit1 + digit2 + carry`, `result_digit = sum % base`, `carry = sum / base`.
4. After processing all digits, append remaining carry if non-zero.
5. Reverse the result (since we built it backwards).

### Variant 4: Pattern-Based Construction (Zigzag, Count-and-Say)
1. Simulate the pattern by tracking which row/group each character belongs to.
2. Build the result by concatenating groups in order.

## Java Template

### Reverse Words in a String
```java
public String reverseWords(String s) {
    String[] words = s.trim().split("\\s+");
    StringBuilder sb = new StringBuilder();
    for (int i = words.length - 1; i >= 0; i--) {
        sb.append(words[i]);
        if (i > 0) sb.append(" ");
    }
    return sb.toString();
}
```

### String to Integer (atoi)
```java
public int myAtoi(String s) {
    int i = 0, n = s.length();
    while (i < n && s.charAt(i) == ' ') i++;
    if (i == n) return 0;

    int sign = 1;
    if (s.charAt(i) == '-' || s.charAt(i) == '+') {
        sign = (s.charAt(i) == '-') ? -1 : 1;
        i++;
    }

    int result = 0;
    while (i < n && Character.isDigit(s.charAt(i))) {
        int digit = s.charAt(i) - '0';
        if (result > (Integer.MAX_VALUE - digit) / 10) {
            return (sign == 1) ? Integer.MAX_VALUE : Integer.MIN_VALUE;
        }
        result = result * 10 + digit;
        i++;
    }
    return result * sign;
}
```

### Add Binary
```java
public String addBinary(String a, String b) {
    StringBuilder sb = new StringBuilder();
    int i = a.length() - 1, j = b.length() - 1, carry = 0;

    while (i >= 0 || j >= 0 || carry > 0) {
        int sum = carry;
        if (i >= 0) sum += a.charAt(i--) - '0';
        if (j >= 0) sum += b.charAt(j--) - '0';
        sb.append(sum % 2);
        carry = sum / 2;
    }
    return sb.reverse().toString();
}
```

### Longest Common Prefix
```java
public String longestCommonPrefix(String[] strs) {
    if (strs.length == 0) return "";
    String prefix = strs[0];
    for (int i = 1; i < strs.length; i++) {
        while (strs[i].indexOf(prefix) != 0) {
            prefix = prefix.substring(0, prefix.length() - 1);
            if (prefix.isEmpty()) return "";
        }
    }
    return prefix;
}
```

### Zigzag Conversion
```java
public String convert(String s, int numRows) {
    if (numRows == 1 || numRows >= s.length()) return s;

    StringBuilder[] rows = new StringBuilder[numRows];
    for (int i = 0; i < numRows; i++) rows[i] = new StringBuilder();

    int currentRow = 0;
    boolean goingDown = false;

    for (char c : s.toCharArray()) {
        rows[currentRow].append(c);
        if (currentRow == 0 || currentRow == numRows - 1) goingDown = !goingDown;
        currentRow += goingDown ? 1 : -1;
    }

    StringBuilder result = new StringBuilder();
    for (StringBuilder row : rows) result.append(row);
    return result.toString();
}
```

## Dry Run Example

**Problem:** Add Binary — `a = "1011", b = "1010"`

| Step | i | j | a[i] | b[j] | carry | sum | digit (sum%2) | carry (sum/2) | result (reversed) |
|------|---|---|------|------|-------|-----|--------------|---------------|-------------------|
| 1 | 3 | 3 | 1 | 0 | 0 | 1 | 1 | 0 | "1" |
| 2 | 2 | 2 | 1 | 1 | 0 | 2 | 0 | 1 | "10" |
| 3 | 1 | 1 | 0 | 0 | 1 | 1 | 1 | 0 | "101" |
| 4 | 0 | 0 | 1 | 1 | 0 | 2 | 0 | 1 | "1010" |
| 5 | -1 | -1 | — | — | 1 | 1 | 1 | 0 | "10101" |

**Reverse:** `"10101"` → `"10101"` → Result: `"10101"` ✓ (which is 21 in decimal: 11 + 10 = 21)

**Problem:** myAtoi — `s = "   -42abc"`

| Step | i | Character | Action | result | sign |
|------|---|-----------|--------|--------|------|
| 1-3 | 0→2 | ' ' | Skip whitespace | 0 | 1 |
| 4 | 3 | '-' | Set sign = -1 | 0 | -1 |
| 5 | 4 | '4' | result = 0*10 + 4 = 4 | 4 | -1 |
| 6 | 5 | '2' | result = 4*10 + 2 = 42 | 42 | -1 |
| 7 | 6 | 'a' | Not a digit → stop | 42 | -1 |

**Return:** `42 * -1 = -42` ✓

## Complexity
- **Time:** O(n) for most string manipulation (single pass); O(n·m) for multiply strings where n, m are lengths
- **Space:** O(n) for building the result string (strings are immutable in Java)

## Edge Cases
- Empty string — return 0, "", or appropriate default
- All whitespace — same as empty after trimming
- Overflow in atoi — clamp to `Integer.MAX_VALUE` or `Integer.MIN_VALUE`
- Leading zeros — `"00042"` should be treated as `42`; `"000"` + `"000"` = `"0"` not `"000000"`
- Single character — handle as a valid single-digit number or single-char string
- Strings of very different lengths in addition/multiplication — shorter string runs out of digits first
- Special characters — Roman numeral subtraction rules (IV = 4, IX = 9)

## Common Mistakes
- **Not handling overflow before it happens in atoi:** Check `result > (MAX_VALUE - digit) / 10` before `result = result * 10 + digit`.
- **Forgetting to reverse the result in addition:** When building from least significant digit, the result string is backwards.
- **Using `==` instead of `.equals()` for string comparison in Java.**
- **Not trimming whitespace or handling multiple spaces between words.**
- **Off-by-one in Roman numeral conversion:** Process from right to left; subtract when a smaller value precedes a larger one.
- **Forgetting the carry after the main loop in addition:** If carry is 1 after processing all digits, append it.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Reverse Words in a String | Medium | Split by whitespace, reverse the array of words, rejoin | https://leetcode.com/problems/reverse-words-in-a-string/ |
| 2 | Zigzag Conversion | Medium | Simulate row assignment with a bouncing index (0→n-1→0) | https://leetcode.com/problems/zigzag-conversion/ |
| 3 | String to Integer (atoi) | Medium | Skip spaces, read sign, build number digit by digit, clamp overflow | https://leetcode.com/problems/string-to-integer-atoi/ |
| 4 | Reverse String | Easy | Two pointers from both ends swapping in-place | https://leetcode.com/problems/reverse-string/ |
| 5 | Longest Common Prefix | Easy | Shrink prefix against each string; stop when it no longer matches | https://leetcode.com/problems/longest-common-prefix/ |
| 6 | Count and Say | Medium | Iterate n-1 times; each iteration reads consecutive groups of same digits | https://leetcode.com/problems/count-and-say/ |
| 7 | Multiply Strings | Medium | Grade-school multiplication: multiply digit-by-digit, accumulate in position array | https://leetcode.com/problems/multiply-strings/ |
| 8 | Compare Version Numbers | Medium | Split by '.', compare each numeric level left to right; treat missing levels as 0 | https://leetcode.com/problems/compare-version-numbers/ |
| 9 | Add Binary | Easy | Process from rightmost bit with carry; handle unequal lengths | https://leetcode.com/problems/add-binary/ |
| 10 | Roman to Integer | Easy | Scan right to left; subtract when current < next, otherwise add | https://leetcode.com/problems/roman-to-integer/ |
