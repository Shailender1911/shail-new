# Pattern: Parentheses and Expressions

## How to Identify This Pattern

### Trigger Words
- "valid parentheses", "balanced brackets"
- "generate all valid combinations of parentheses"
- "evaluate expression", "calculator"
- "decode string", "nested encoding"
- "reverse Polish notation", "postfix"
- "remove invalid", "minimum removals"
- "score of parentheses"

### Constraint Clues
- Input contains characters like `(`, `)`, `[`, `]`, `{`, `}` and the task involves matching or balancing them
- Expression contains nested operations that must be resolved inside-out
- Need to handle operator precedence (`+`, `-`, `*`, `/`) without built-in eval
- String has nested repeated patterns like `3[a2[c]]` that must be expanded
- Problem asks for all valid arrangements — suggests backtracking with pruning

### When to Use This vs Similar Patterns
| Consideration | Parentheses/Expression Stack | Recursion / DFS | Monotonic Stack | String Manipulation |
|---|---|---|---|---|
| Nesting involved? | Yes — stack mirrors nesting depth | Yes — call stack mirrors nesting | No — flat ordering | No |
| Matching pairs? | Yes — push open, pop on close | Possible but stack is simpler | No | No |
| Operator precedence? | Handle with stack levels | Handle with grammar rules | N/A | N/A |
| Key structure | Explicit stack | Implicit call stack | Stack with ordering invariant | StringBuilder |

## Core Approach

### Valid Parentheses (bracket matching)
1. Push every opening bracket onto the stack.
2. When a closing bracket arrives, check if the stack top is the matching opener.
3. If it matches, pop the opener. If it doesn't match or the stack is empty, return false.
4. After processing all characters, the stack must be empty for a valid string.

### Basic Calculator (with `+`, `-`, `(`, `)`)
1. Use a stack to save the `(sign, accumulated result)` before entering a parenthesized sub-expression.
2. Maintain `result` and `sign` (+1 or -1). When hitting a digit, parse the full number.
3. On `(`: push current `result` and `sign` onto the stack, reset them.
4. On `)`: pop `sign` and `prevResult`, compute `result = prevResult + sign * result`.
5. After the loop, return `result`.

### Decode String like `3[a2[c]]`
1. Use two stacks: one for repeat counts, one for accumulated strings.
2. On digit: parse the full number.
3. On `[`: push current count and current string onto their stacks, reset both.
4. On `]`: pop the previous string and count, append `currentString * count` to the popped string.
5. On letter: append to the current string.

## Java Template

### Valid Parentheses
```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = Map.of(')', '(', ']', '[', '}', '{');

    for (char c : s.toCharArray()) {
        if (pairs.containsValue(c)) {
            stack.push(c);
        } else if (pairs.containsKey(c)) {
            if (stack.isEmpty() || stack.pop() != pairs.get(c)) {
                return false;
            }
        }
    }
    return stack.isEmpty();
}
```

### Basic Calculator
```java
public int calculate(String s) {
    Deque<Integer> stack = new ArrayDeque<>();
    int result = 0, num = 0, sign = 1;

    for (char c : s.toCharArray()) {
        if (Character.isDigit(c)) {
            num = num * 10 + (c - '0');
        } else if (c == '+') {
            result += sign * num;
            num = 0;
            sign = 1;
        } else if (c == '-') {
            result += sign * num;
            num = 0;
            sign = -1;
        } else if (c == '(') {
            stack.push(result);
            stack.push(sign);
            result = 0;
            sign = 1;
        } else if (c == ')') {
            result += sign * num;
            num = 0;
            result *= stack.pop(); // sign before the parenthesis
            result += stack.pop(); // result before the parenthesis
        }
    }
    return result + sign * num;
}
```

### Decode String
```java
public String decodeString(String s) {
    Deque<Integer> countStack = new ArrayDeque<>();
    Deque<StringBuilder> stringStack = new ArrayDeque<>();
    StringBuilder current = new StringBuilder();
    int k = 0;

    for (char c : s.toCharArray()) {
        if (Character.isDigit(c)) {
            k = k * 10 + (c - '0');
        } else if (c == '[') {
            countStack.push(k);
            stringStack.push(current);
            current = new StringBuilder();
            k = 0;
        } else if (c == ']') {
            int repeat = countStack.pop();
            StringBuilder decoded = stringStack.pop();
            decoded.append(String.valueOf(current).repeat(repeat));
            current = decoded;
        } else {
            current.append(c);
        }
    }
    return current.toString();
}
```

## Dry Run Example

**Problem:** Valid Parentheses — input: `"({[]})"`.

| Step | Char | Stack | Action |
|------|------|-------|--------|
| 1 | ( | [(] | Push ( |
| 2 | { | [(, {] | Push { |
| 3 | [ | [(, {, [] | Push [ |
| 4 | ] | [(, {] | Pop [ — matches ] ✓ |
| 5 | } | [(] | Pop { — matches } ✓ |
| 6 | ) | [] | Pop ( — matches ) ✓ |
| End | — | [] | Stack empty → **true** ✓ |

**Problem:** Decode String — input: `"3[a2[c]]"`.

| Step | Char | k | current | countStack | stringStack | Action |
|------|------|---|---------|------------|-------------|--------|
| 1 | 3 | 3 | "" | [] | [] | Parse digit |
| 2 | [ | 0 | "" | [3] | [""] | Push k=3 and current="", reset |
| 3 | a | 0 | "a" | [3] | [""] | Append 'a' |
| 4 | 2 | 2 | "a" | [3] | [""] | Parse digit |
| 5 | [ | 0 | "" | [3,2] | ["","a"] | Push k=2 and current="a", reset |
| 6 | c | 0 | "c" | [3,2] | ["","a"] | Append 'c' |
| 7 | ] | 0 | "acc" | [3] | [""] | Pop 2 and "a" → "a" + "cc" = "acc" |
| 8 | ] | 0 | "accaccacc" | [] | [] | Pop 3 and "" → "" + "accaccacc" |

**Result:** `"accaccacc"` ✓

## Complexity
- **Valid Parentheses:** Time O(n), Space O(n)
- **Basic Calculator:** Time O(n), Space O(n) for nesting depth
- **Decode String:** Time O(n * maxRepeat), Space O(nesting depth)
- **Generate Parentheses:** Time O(4^n / √n) — Catalan number, Space O(n) recursion depth

## Edge Cases
- Empty string — trivially valid / no output
- Single bracket `"("` — invalid, stack won't be empty at the end
- No parentheses in expression — just compute the arithmetic directly
- Deeply nested structures `"1[1[1[1[a]]]]"` — stack depth grows linearly
- Leading/trailing spaces in calculator expressions — skip whitespace
- Multi-digit numbers — always parse the complete number, not just one digit
- Negative numbers in calculator — handle unary minus (e.g., `(-1 + 2)`)

## Common Mistakes
- **Not checking stack emptiness before popping:** A closing bracket with an empty stack means the string is invalid — don't let it throw an exception.
- **Forgetting to process the last number:** In calculator problems, the final number isn't followed by an operator, so you must add it to the result after the loop.
- **Confusing operator precedence in Calculator II:** `*` and `/` bind tighter than `+` and `-`. Process `*` and `/` immediately; push `+` and `-` results for later summation.
- **Not resetting state on `[` in Decode String:** Both the count and current string must be pushed and reset; forgetting either produces wrong output.
- **Generate Parentheses: not pruning properly:** At any point, the count of `)` used must not exceed `(` used, and `(` used must not exceed `n`.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Valid Parentheses | Easy | Stack push for openers, pop and match for closers; stack must be empty at end | https://leetcode.com/problems/valid-parentheses/ |
| 2 | Generate Parentheses | Medium | Backtracking with two counters (open, close); add ')' only if close < open | https://leetcode.com/problems/generate-parentheses/ |
| 3 | Longest Valid Parentheses | Hard | Stack of indices; push -1 as base, pop on ')', length = i - stack.peek() | https://leetcode.com/problems/longest-valid-parentheses/ |
| 4 | Basic Calculator | Hard | Stack saves (result, sign) on '('; handles +, -, and nested parentheses | https://leetcode.com/problems/basic-calculator/ |
| 5 | Basic Calculator II | Medium | Process * and / immediately (higher precedence); push +/- operands onto stack, sum at end | https://leetcode.com/problems/basic-calculator-ii/ |
| 6 | Decode String | Medium | Two stacks (count + string); expand inner brackets first on ']', concatenate on pop | https://leetcode.com/problems/decode-string/ |
| 7 | Evaluate Reverse Polish Notation | Medium | Stack of operands; on operator, pop two operands, compute, push result back | https://leetcode.com/problems/evaluate-reverse-polish-notation/ |
| 8 | Remove Invalid Parentheses | Hard | BFS level by level removing one bracket at a time; or backtracking with left/right counters | https://leetcode.com/problems/remove-invalid-parentheses/ |
| 9 | Score of Parentheses | Medium | Stack-based: push 0 on '(', on ')' pop and compute max(2*top, 1), add to new top | https://leetcode.com/problems/score-of-parentheses/ |
| 10 | Minimum Remove to Make Valid Parentheses | Medium | Two-pass or stack of indices: mark unmatched parens, rebuild string without them | https://leetcode.com/problems/minimum-remove-to-make-valid-parentheses/ |
