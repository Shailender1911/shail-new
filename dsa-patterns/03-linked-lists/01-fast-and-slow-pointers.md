# Pattern: Fast and Slow Pointers (Floyd's Tortoise and Hare)

## How to Identify This Pattern

### Trigger Words
- "cycle", "loop", "circular"
- "middle of the linked list"
- "detect", "find the start of the cycle"
- "palindrome" (on a linked list)
- "happy number", "repeating sequence"
- "remove nth node from the end"
- "intersection point of two linked lists"
- "reorder", "rearrange"

### Constraint Clues
- Must solve in **O(1) extra space** — cannot use a HashSet to track visited nodes
- Need to find a structural property (middle, cycle, intersection) without knowing the length
- Problem involves a linked list where traversal speed matters
- Input may form an implicit linked list (e.g., array index → value mapping)
- Need to split the list into two halves

### When to Use This vs Similar Patterns
| Consideration | Fast & Slow Pointers | HashSet Approach | Two Pointers on Array |
|---|---|---|---|
| Space | O(1) | O(n) | O(1) |
| Data structure | Linked list / implicit sequence | Any | Sorted array |
| Cycle detection? | Yes — guaranteed by speed difference | Yes — store visited | Not applicable |
| Finding middle? | Yes — fast reaches end when slow is at mid | No | Use `n/2` index directly |
| Typical goal | Cycle / middle / meeting point | Membership check | Pair finding / partitioning |

## Core Approach

### Variant 1: Cycle Detection
1. Initialize `slow` and `fast` both at `head`.
2. Move `slow` one step and `fast` two steps per iteration.
3. If `fast` or `fast.next` becomes `null`, there is no cycle.
4. If `slow == fast`, a cycle exists.

### Variant 2: Find Cycle Start (Floyd's Algorithm Phase 2)
1. After detecting a meeting point inside the cycle, reset one pointer to `head`.
2. Move both pointers one step at a time.
3. The node where they meet again is the cycle entrance.

### Variant 3: Find the Middle Node
1. Initialize `slow` and `fast` both at `head`.
2. Move `slow` one step and `fast` two steps.
3. When `fast` reaches the end (or `fast.next` is null), `slow` is at the middle.
4. For even-length lists, this gives the second middle node; adjust if the first is needed.

## Java Template

### Cycle Detection
```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

### Find Cycle Start
```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            ListNode entry = head;
            while (entry != slow) {
                entry = entry.next;
                slow = slow.next;
            }
            return entry;
        }
    }
    return null;
}
```

### Find Middle
```java
public ListNode middleNode(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow;
}
```

## Dry Run Example

**Problem:** Detect cycle start in `1 → 2 → 3 → 4 → 5 → 3` (node 5 points back to node 3).

**Phase 1 — Find meeting point:**

| Step | slow | fast | Action |
|------|------|------|--------|
| Init | 1 | 1 | Start both at head |
| 1 | 2 | 3 | slow +1, fast +2 |
| 2 | 3 | 5 | slow +1, fast +2 |
| 3 | 4 | 4 | slow +1, fast +2 (fast went 5→3→4) → slow == fast, meeting point = node 4 |

**Phase 2 — Find cycle entrance:**

| Step | entry (from head) | slow (from meeting) | Action |
|------|-------------------|---------------------|--------|
| Init | 1 | 4 | Reset entry to head |
| 1 | 2 | 5 | Both move +1 |
| 2 | 3 | 3 | Both move +1 → entry == slow → cycle starts at node 3 ✓ |

**Problem:** Find middle of `1 → 2 → 3 → 4 → 5`.

| Step | slow | fast | Action |
|------|------|------|--------|
| Init | 1 | 1 | Start both at head |
| 1 | 2 | 3 | slow +1, fast +2 |
| 2 | 3 | 5 | slow +1, fast +2 → fast.next == null → stop → middle = node 3 ✓ |

## Complexity
- **Time:** O(n) — slow pointer traverses at most n nodes; in cycle problems, phase 2 is also O(n)
- **Space:** O(1) — only two pointers regardless of list size

## Edge Cases
- Empty list (`head == null`) — return immediately, no cycle / no middle
- Single node pointing to itself — cycle of length 1; fast and slow meet after one step
- Single node with no cycle — `fast.next` is null immediately
- Even-length list — middle returns second of the two middle nodes (clarify which one is expected)
- Very long tail before cycle — phase 2 still converges in O(n)
- No cycle at all — fast reaches null, return false/null

## Common Mistakes
- **Checking `fast.next` before `fast` for null:** Always check `fast != null` first, then `fast.next != null`, to avoid NullPointerException.
- **Initializing slow and fast at different nodes:** Both must start at `head` for the math behind Floyd's algorithm to work.
- **Forgetting phase 2 of Floyd's:** Detecting that a cycle exists is not the same as finding where it starts — you need the second phase with `entry` pointer.
- **Off-by-one in middle finding:** For even-length lists, `slow` lands on the second middle node. If you need the first, use `fast.next != null && fast.next.next != null`.
- **Using `==` on values instead of node references:** Cycle detection compares reference identity (`slow == fast`), not `.val`.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Linked List Cycle | Easy | Basic fast/slow — if they meet, cycle exists | https://leetcode.com/problems/linked-list-cycle/ |
| 2 | Linked List Cycle II | Medium | Floyd's phase 2 — reset one pointer to head, walk both at speed 1 | https://leetcode.com/problems/linked-list-cycle-ii/ |
| 3 | Middle of the Linked List | Easy | When fast reaches end, slow is at mid | https://leetcode.com/problems/middle-of-the-linked-list/ |
| 4 | Happy Number | Easy | Implicit linked list — number sequence either cycles or reaches 1 | https://leetcode.com/problems/happy-number/ |
| 5 | Palindrome Linked List | Easy | Find middle with slow/fast, reverse second half, compare both halves | https://leetcode.com/problems/palindrome-linked-list/ |
| 6 | Remove Nth Node From End of List | Medium | Two pointers with n-gap — when fast hits null, slow is at the target | https://leetcode.com/problems/remove-nth-node-from-end-of-list/ |
| 7 | Reorder List | Medium | Find middle, reverse second half, merge alternately | https://leetcode.com/problems/reorder-list/ |
| 8 | Find the Duplicate Number | Medium | Array as implicit linked list — Floyd's on index→value mapping | https://leetcode.com/problems/find-the-duplicate-number/ |
| 9 | Intersection of Two Linked Lists | Easy | Align tails by switching heads — both pointers traverse equal total length | https://leetcode.com/problems/intersection-of-two-linked-lists/ |
| 10 | Sort List | Medium | Find middle to split, merge sort recursively on both halves | https://leetcode.com/problems/sort-list/ |
