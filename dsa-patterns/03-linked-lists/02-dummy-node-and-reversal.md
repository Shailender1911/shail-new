# Pattern: Dummy Node and In-Place Reversal

## How to Identify This Pattern

### Trigger Words
- "reverse", "reverse between positions", "reverse in groups of k"
- "remove", "delete nodes matching a condition"
- "swap", "rearrange", "rotate"
- "partition", "split around a value"
- "odd-even", "separate by position/property"
- "remove duplicates"

### Constraint Clues
- The head of the list might change (node removal, reversal, reordering)
- Must solve in **O(1) extra space** — no auxiliary list or array allowed
- Need to rewire `next` pointers without creating new nodes
- Problem says "in-place" or "modify the list itself"
- Operations happen on a subrange of the list (positions l to r)

### When to Use This vs Similar Patterns
| Consideration | Dummy Node + Reversal | Fast & Slow | Stack-Based Reversal |
|---|---|---|---|
| Head may change? | Yes — dummy node handles it cleanly | Rarely | Yes, but uses O(n) space |
| Subrange operation? | Yes — reverse between positions l..r | No | Yes, but heavier |
| Space | O(1) | O(1) | O(n) for the stack |
| Complexity of pointer rewiring | Higher — must track prev, curr, next carefully | Low | Low (push/pop) |
| When to prefer | In-place reversal, head-changing deletions | Cycle / middle finding | When simplicity > space |

## Core Approach

### Dummy Node Technique
1. Create a `dummy` node whose `.next` points to `head`.
2. Perform all operations using `dummy` as the anchor — this avoids special-casing when the head itself is removed or changed.
3. Return `dummy.next` as the new head.

### Full Reversal
1. Initialize `prev = null`, `curr = head`.
2. At each step, save `next = curr.next`.
3. Point `curr.next = prev` (reverse the link).
4. Advance: `prev = curr`, `curr = next`.
5. When `curr` is null, `prev` is the new head.

### Partial Reversal (between positions l and r)
1. Walk to position `l - 1` to get the node just before the reversal window (`pre`).
2. Reverse the next `r - l + 1` nodes using the standard reversal technique.
3. Reconnect: `pre.next` to the new sub-head, and the old sub-head's `.next` to the node after the window.

### Reverse in Groups of k
1. Count whether at least k nodes remain.
2. If yes, reverse the next k nodes and connect the tail of the reversed group to the result of recursively processing the rest.
3. If fewer than k nodes remain, leave them as-is (or reverse, depending on the variant).

## Java Template

### Full Reversal
```java
public ListNode reverseList(ListNode head) {
    ListNode prev = null, curr = head;
    while (curr != null) {
        ListNode next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}
```

### Partial Reversal (positions left to right)
```java
public ListNode reverseBetween(ListNode head, int left, int right) {
    ListNode dummy = new ListNode(0, head);
    ListNode pre = dummy;
    for (int i = 1; i < left; i++) {
        pre = pre.next;
    }
    ListNode curr = pre.next;
    for (int i = 0; i < right - left; i++) {
        ListNode next = curr.next;
        curr.next = next.next;
        next.next = pre.next;
        pre.next = next;
    }
    return dummy.next;
}
```

### Dummy Node for Removal
```java
public ListNode removeElements(ListNode head, int val) {
    ListNode dummy = new ListNode(0, head);
    ListNode prev = dummy;
    while (prev.next != null) {
        if (prev.next.val == val) {
            prev.next = prev.next.next;
        } else {
            prev = prev.next;
        }
    }
    return dummy.next;
}
```

## Dry Run Example

**Problem:** Reverse `1 → 2 → 3 → 4 → 5`.

| Step | prev | curr | next | Action | List State |
|------|------|------|------|--------|------------|
| Init | null | 1 | — | — | 1→2→3→4→5 |
| 1 | null | 1 | 2 | 1.next = null | null←1  2→3→4→5 |
| 2 | 1 | 2 | 3 | 2.next = 1 | null←1←2  3→4→5 |
| 3 | 2 | 3 | 4 | 3.next = 2 | null←1←2←3  4→5 |
| 4 | 3 | 4 | 5 | 4.next = 3 | null←1←2←3←4  5 |
| 5 | 4 | 5 | null | 5.next = 4 | null←1←2←3←4←5 |
| End | 5 | null | — | return prev | 5→4→3→2→1 ✓ |

**Problem:** Reverse Between positions 2 and 4 in `1 → 2 → 3 → 4 → 5`.

| Step | pre | curr | next | Action | List State |
|------|-----|------|------|--------|------------|
| Init | 1 | 2 | — | Walk pre to pos 1, curr = pre.next | d→1→2→3→4→5 |
| i=0 | 1 | 2 | 3 | curr.next=3.next(4), 3.next=pre.next(2), pre.next=3 | d→1→3→2→4→5 |
| i=1 | 1 | 2 | 4 | curr.next=4.next(5), 4.next=pre.next(3), pre.next=4 | d→1→4→3→2→5 |
| Done | — | — | — | return dummy.next | 1→4→3→2→5 ✓ |

## Complexity
- **Time:** O(n) — single or double pass through the list
- **Space:** O(1) — all pointer manipulation is in-place (iterative reversal); O(n/k) stack space if using recursion for k-group reversal

## Edge Cases
- Empty list or single node — nothing to reverse or remove; return as-is
- `left == right` in partial reversal — no reversal needed, return unchanged
- Removing the head node — dummy node prevents null-pointer issues
- All nodes match the removal condition — list becomes empty, `dummy.next` is null
- `k == 1` in group reversal — no reversal needed
- `k > list length` — leave the list unchanged (or reverse all, depending on variant)
- Consecutive duplicates when removing duplicates — must handle chains of identical values

## Common Mistakes
- **Not using a dummy node when the head can change:** Leads to special-case code and subtle bugs. Always use a dummy when the first node might be removed or moved.
- **Losing the `next` pointer before reassigning `curr.next`:** Save `next = curr.next` before any pointer modification.
- **Off-by-one in partial reversal:** The loop runs `right - left` times, not `right - left + 1`. Drawing the pointers on paper helps.
- **Forgetting to reconnect the tail of a reversed segment:** After reversing a sublist, the old first node of that segment is now the tail — its `.next` must point to the rest of the list.
- **Mixing up `pre.next` and `curr` in the insertion-style reversal:** `pre.next` always points to the current front of the reversed segment; `curr` stays fixed as the rotating pivot.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Reverse Linked List | Easy | Iterative: prev/curr/next trio; recursive: reverse rest then rewire | https://leetcode.com/problems/reverse-linked-list/ |
| 2 | Reverse Linked List II | Medium | Walk to position left-1, then insertion-style reversal for (right-left) steps | https://leetcode.com/problems/reverse-linked-list-ii/ |
| 3 | Swap Nodes in Pairs | Medium | Dummy node + swap adjacent pairs by rewiring three pointers per pair | https://leetcode.com/problems/swap-nodes-in-pairs/ |
| 4 | Reverse Nodes in k-Group | Hard | Count k nodes, reverse the group, recurse/iterate on the remainder | https://leetcode.com/problems/reverse-nodes-in-k-group/ |
| 5 | Remove Linked List Elements | Easy | Dummy node + skip nodes where val matches | https://leetcode.com/problems/remove-linked-list-elements/ |
| 6 | Remove Duplicates from Sorted List | Easy | Single pointer — skip consecutive nodes with same value | https://leetcode.com/problems/remove-duplicates-from-sorted-list/ |
| 7 | Remove Duplicates from Sorted List II | Medium | Dummy node — delete all nodes that have duplicates, keep only distinct values | https://leetcode.com/problems/remove-duplicates-from-sorted-list-ii/ |
| 8 | Partition List | Medium | Dummy nodes for two sublists (< x and >= x), concatenate at the end | https://leetcode.com/problems/partition-list/ |
| 9 | Odd Even Linked List | Medium | Separate odd-indexed and even-indexed nodes into two chains, then connect | https://leetcode.com/problems/odd-even-linked-list/ |
| 10 | Rotate List | Medium | Form a cycle, compute effective rotation with modulo, break at the new tail | https://leetcode.com/problems/rotate-list/ |
