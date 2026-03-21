# Pattern: Merge and Recursion on Linked Lists

## How to Identify This Pattern

### Trigger Words
- "merge", "combine two/k sorted lists"
- "add two numbers represented as linked lists"
- "flatten", "multilevel", "nested structure"
- "copy", "deep copy", "clone with random pointer"
- "sort the linked list"
- "split", "divide into parts"
- "insertion sort on a linked list"
- "recursive structure"

### Constraint Clues
- Multiple sorted linked lists need to be combined into one
- Digit-by-digit arithmetic where each node represents a digit
- List has extra pointers (random, child) requiring careful cloning
- Need to split a list into k roughly equal parts
- Problem naturally decomposes into subproblems on sublists (recursion)
- Must maintain relative order or sorted order during the operation

### When to Use This vs Similar Patterns
| Consideration | Merge / Recursion | Dummy + Reversal | Fast & Slow |
|---|---|---|---|
| Multiple lists? | Yes — merge 2 or k lists | No — operates on one list | No — operates on one list |
| Arithmetic? | Yes — carry-based node-by-node | No | No |
| Recursive decomposition? | Natural fit — sublists are smaller instances | Usually iterative | Usually iterative |
| Extra pointer handling? | Yes — random/child pointers | No | No |
| Sorting? | Merge sort on lists | No | Provides the split point |

## Core Approach

### Merge Two Sorted Lists
1. Create a `dummy` node and a `tail` pointer at `dummy`.
2. Compare the heads of both lists; attach the smaller node to `tail.next`.
3. Advance the pointer of the list that donated the node and advance `tail`.
4. When one list is exhausted, attach the remainder of the other.
5. Return `dummy.next`.

### Merge K Sorted Lists (Divide and Conquer)
1. Pair up the k lists and merge each pair using the two-list merge.
2. After one round, you have k/2 merged lists.
3. Repeat until a single merged list remains.
4. This is O(N log k) where N is total nodes and k is the number of lists.

### Add Two Numbers (digit-by-digit with carry)
1. Traverse both lists simultaneously, summing corresponding digits plus `carry`.
2. Create a new node with `sum % 10`, update `carry = sum / 10`.
3. Continue until both lists are exhausted and carry is 0.

### Recursive Pattern
1. Define the base case: null or single-node list.
2. Recurse on the subproblem (rest of the list or two sublists).
3. Combine the recursive result with the current node by rewiring pointers.

## Java Template

### Merge Two Sorted Lists
```java
public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0);
    ListNode tail = dummy;
    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) {
            tail.next = l1;
            l1 = l1.next;
        } else {
            tail.next = l2;
            l2 = l2.next;
        }
        tail = tail.next;
    }
    tail.next = (l1 != null) ? l1 : l2;
    return dummy.next;
}
```

### Merge K Sorted Lists (Divide and Conquer)
```java
public ListNode mergeKLists(ListNode[] lists) {
    if (lists == null || lists.length == 0) return null;
    return mergeRange(lists, 0, lists.length - 1);
}

private ListNode mergeRange(ListNode[] lists, int lo, int hi) {
    if (lo == hi) return lists[lo];
    int mid = lo + (hi - lo) / 2;
    ListNode left = mergeRange(lists, lo, mid);
    ListNode right = mergeRange(lists, mid + 1, hi);
    return mergeTwoLists(left, right);
}
```

### Add Two Numbers
```java
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0);
    ListNode curr = dummy;
    int carry = 0;
    while (l1 != null || l2 != null || carry != 0) {
        int sum = carry;
        if (l1 != null) { sum += l1.val; l1 = l1.next; }
        if (l2 != null) { sum += l2.val; l2 = l2.next; }
        curr.next = new ListNode(sum % 10);
        carry = sum / 10;
        curr = curr.next;
    }
    return dummy.next;
}
```

## Dry Run Example

**Problem:** Merge `1 → 3 → 5` and `2 → 4 → 6`.

| Step | l1 | l2 | Comparison | tail.next = | Merged So Far |
|------|----|----|------------|-------------|---------------|
| Init | 1 | 2 | — | — | dummy→ |
| 1 | 1 | 2 | 1 ≤ 2 | 1 | dummy→1 |
| 2 | 3 | 2 | 3 > 2 | 2 | dummy→1→2 |
| 3 | 3 | 4 | 3 ≤ 4 | 3 | dummy→1→2→3 |
| 4 | 5 | 4 | 5 > 4 | 4 | dummy→1→2→3→4 |
| 5 | 5 | 6 | 5 ≤ 6 | 5 | dummy→1→2→3→4→5 |
| 6 | null | 6 | l1 exhausted | attach 6 | dummy→1→2→3→4→5→6 ✓ |

**Problem:** Add `2 → 4 → 3` (342) and `5 → 6 → 4` (465). Expected: 807 → `7 → 0 → 8`.

| Step | l1.val | l2.val | carry | sum | node val | carry out |
|------|--------|--------|-------|-----|----------|-----------|
| 1 | 2 | 5 | 0 | 7 | 7 | 0 |
| 2 | 4 | 6 | 0 | 10 | 0 | 1 |
| 3 | 3 | 4 | 1 | 8 | 8 | 0 |
| Done | — | — | 0 | — | — | Result: 7→0→8 ✓ |

## Complexity
- **Time:**
  - Merge two lists: O(n + m) where n, m are lengths of the two lists
  - Merge k lists (divide & conquer): O(N log k) where N = total nodes
  - Add two numbers: O(max(n, m))
  - Copy with random pointer: O(n)
- **Space:**
  - Iterative merge/add: O(1) extra (excluding output nodes)
  - Divide & conquer merge: O(log k) recursion stack
  - Recursive solutions: O(n) stack space

## Edge Cases
- One or both lists are empty — return the non-empty list or null
- Lists of very different lengths — the merge/add loop must handle the remaining longer portion
- Carry propagates past the end of both lists (e.g., 999 + 1 = 1000) — continue loop while carry > 0
- K lists where some are null — skip null entries
- Single-element lists — base case must handle correctly
- Digits stored in reverse order vs forward order — changes whether you need to reverse or use a stack
- Random pointer is null for some nodes — handle null gracefully during cloning
- Split into k parts where k > list length — some parts will be empty (null)

## Common Mistakes
- **Forgetting the final carry in addition:** The while loop must continue when `carry != 0` even if both lists are exhausted. Missing this drops the leading digit (e.g., 999 + 1 = 000 instead of 1000).
- **Not handling null lists in merge k:** If `lists` contains null entries, skip them or handle gracefully; don't assume all entries are non-null.
- **Creating a cycle when cloning random pointers:** The interleaving approach (insert clones between originals) or a HashMap must be used carefully to avoid corrupting the original list.
- **Stack overflow on very long lists with recursion:** Iterative approaches are safer for lists with 10⁴+ nodes. Use recursion only when the problem size is bounded.
- **Modifying the original lists during merge:** If the problem requires preserving the originals, create new nodes instead of rewiring existing ones.
- **Off-by-one when splitting into k parts:** Calculate `quotient = n / k` and `remainder = n % k`. The first `remainder` parts get `quotient + 1` nodes; the rest get `quotient`.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Merge Two Sorted Lists | Easy | Dummy node + compare heads, attach smaller, advance that pointer | https://leetcode.com/problems/merge-two-sorted-lists/ |
| 2 | Merge k Sorted Lists | Hard | Divide-and-conquer pairwise merge or min-heap of k heads | https://leetcode.com/problems/merge-k-sorted-lists/ |
| 3 | Sort List | Medium | Find middle (fast/slow), split, merge sort both halves recursively | https://leetcode.com/problems/sort-list/ |
| 4 | Add Two Numbers | Medium | Traverse both lists with carry; digits are in reverse order | https://leetcode.com/problems/add-two-numbers/ |
| 5 | Add Two Numbers II | Medium | Digits in forward order — use stacks or reverse first, then add with carry | https://leetcode.com/problems/add-two-numbers-ii/ |
| 6 | Flatten a Multilevel Doubly Linked List | Medium | DFS/recursion — when child exists, flatten child chain and splice into main | https://leetcode.com/problems/flatten-a-multilevel-doubly-linked-list/ |
| 7 | Copy List with Random Pointer | Medium | Interleave clones between originals, copy random links, then separate | https://leetcode.com/problems/copy-list-with-random-pointer/ |
| 8 | Split Linked List in Parts | Medium | Compute n/k and n%k; first (n%k) parts get one extra node | https://leetcode.com/problems/split-linked-list-in-parts/ |
| 9 | Insertion Sort List | Medium | Build sorted result list by inserting each node at correct position | https://leetcode.com/problems/insertion-sort-list/ |
| 10 | Linked List in Binary Tree | Medium | For each tree node, try matching the list from head using DFS | https://leetcode.com/problems/linked-list-in-binary-tree/ |
