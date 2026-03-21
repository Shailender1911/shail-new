# Pattern: Stack Design and Simulation

## How to Identify This Pattern

### Trigger Words
- "design a stack", "implement a queue", "custom data structure"
- "min stack", "max stack", "frequency stack"
- "simulate", "undo/redo", "browser history"
- "flatten", "iterator over nested structure"
- "circular queue", "bounded buffer"
- "LRU", "least recently used"

### Constraint Clues
- Operations must be O(1) for push, pop, and some auxiliary query (min, max, frequency)
- Must implement one data structure using another (queue using stacks, stack using queues)
- Need to support non-standard stack operations like increment bottom k elements
- Problem asks you to design a class with specific method signatures
- Nested or hierarchical data needs to be traversed in a flat, lazy manner

### When to Use This vs Similar Patterns
| Consideration | Stack Design | Monotonic Stack | Standard Stack | Hash Map |
|---|---|---|---|---|
| Goal | Build custom DS with extra capabilities | Find next greater/smaller | LIFO push/pop | Key-value storage |
| Auxiliary tracking | Extra stack/map for min/freq/etc. | Monotonic ordering | None | None |
| Typical operations | push, pop, getMin, getMax, increment | Push/pop based on ordering | Push/pop | Get/put |
| Design emphasis | API correctness + O(1) guarantees | Algorithmic pattern | Basic usage | Access pattern |

## Core Approach

### Min Stack (two-stack approach)
1. Maintain a main stack for regular push/pop operations.
2. Maintain a parallel min-stack: push the current minimum onto it whenever you push to the main stack.
3. On `pop()`, pop from both stacks.
4. `getMin()` returns the top of the min-stack — always O(1).

### Queue Using Two Stacks (amortized O(1))
1. Use an `input` stack for enqueue operations.
2. Use an `output` stack for dequeue operations.
3. When `output` is empty and a dequeue is needed, transfer all elements from `input` to `output` (reversing the order).
4. Each element is moved at most once, so amortized cost per operation is O(1).

### Maximum Frequency Stack
1. Use a map `freq` to track the frequency of each element.
2. Use a map `groupByFreq` where key = frequency and value = a stack of elements with that frequency.
3. Track `maxFreq` globally.
4. On `push(x)`: increment `freq[x]`, update `maxFreq`, push x onto `groupByFreq[freq[x]]`.
5. On `pop()`: pop from `groupByFreq[maxFreq]`, decrement `freq[x]`, if group is now empty decrement `maxFreq`.

## Java Template

### Min Stack
```java
class MinStack {
    private Deque<Integer> stack;
    private Deque<Integer> minStack;

    public MinStack() {
        stack = new ArrayDeque<>();
        minStack = new ArrayDeque<>();
    }

    public void push(int val) {
        stack.push(val);
        int min = minStack.isEmpty() ? val : Math.min(val, minStack.peek());
        minStack.push(min);
    }

    public void pop() {
        stack.pop();
        minStack.pop();
    }

    public int top() {
        return stack.peek();
    }

    public int getMin() {
        return minStack.peek();
    }
}
```

### Queue Using Two Stacks
```java
class MyQueue {
    private Deque<Integer> input;
    private Deque<Integer> output;

    public MyQueue() {
        input = new ArrayDeque<>();
        output = new ArrayDeque<>();
    }

    public void push(int x) {
        input.push(x);
    }

    public int pop() {
        peek();
        return output.pop();
    }

    public int peek() {
        if (output.isEmpty()) {
            while (!input.isEmpty()) {
                output.push(input.pop());
            }
        }
        return output.peek();
    }

    public boolean empty() {
        return input.isEmpty() && output.isEmpty();
    }
}
```

### Maximum Frequency Stack
```java
class FreqStack {
    private Map<Integer, Integer> freq;
    private Map<Integer, Deque<Integer>> groupByFreq;
    private int maxFreq;

    public FreqStack() {
        freq = new HashMap<>();
        groupByFreq = new HashMap<>();
        maxFreq = 0;
    }

    public void push(int val) {
        int f = freq.merge(val, 1, Integer::sum);
        maxFreq = Math.max(maxFreq, f);
        groupByFreq.computeIfAbsent(f, k -> new ArrayDeque<>()).push(val);
    }

    public int pop() {
        Deque<Integer> group = groupByFreq.get(maxFreq);
        int val = group.pop();
        freq.merge(val, -1, Integer::sum);
        if (group.isEmpty()) maxFreq--;
        return val;
    }
}
```

## Dry Run Example

**Problem:** Min Stack — operations: `push(5), push(3), push(7), getMin(), pop(), getMin(), push(1), getMin()`

| Operation | Main Stack | Min Stack | Return |
|-----------|-----------|-----------|--------|
| push(5) | [5] | [5] | — |
| push(3) | [5, 3] | [5, 3] | — |
| push(7) | [5, 3, 7] | [5, 3, 3] | — |
| getMin() | [5, 3, 7] | [5, 3, 3] | **3** |
| pop() | [5, 3] | [5, 3] | 7 |
| getMin() | [5, 3] | [5, 3] | **3** |
| push(1) | [5, 3, 1] | [5, 3, 1] | — |
| getMin() | [5, 3, 1] | [5, 3, 1] | **1** ✓ |

**Problem:** Queue Using Two Stacks — operations: `push(1), push(2), peek(), pop(), push(3), pop()`

| Operation | Input Stack | Output Stack | Return | Notes |
|-----------|------------|-------------|--------|-------|
| push(1) | [1] | [] | — | — |
| push(2) | [1, 2] | [] | — | — |
| peek() | [] | [2, 1] | **1** | Transfer: input → output |
| pop() | [] | [2] | **1** | Pop from output |
| push(3) | [3] | [2] | — | — |
| pop() | [3] | [] | **2** | Output still has 2 |
| pop() | [] | [3] | **3** | Transfer again ✓ |

## Complexity
- **Min Stack:** O(1) time for all operations; O(n) space for the auxiliary min-stack
- **Queue using Two Stacks:** O(1) amortized time for enqueue and dequeue; O(n) space
- **Frequency Stack:** O(1) time for push and pop; O(n) space for maps and groups

## Edge Cases
- Pop or peek on an empty structure — must handle gracefully or state precondition
- Single element — min is that element; queue and stack behave identically
- All duplicate elements — min-stack holds the same value repeated; freq-stack freq keeps incrementing
- Large number of operations — verify amortized guarantees hold (queue with two stacks)
- Interleaved push/pop — test that lazy transfer in queue-from-stacks is triggered correctly

## Common Mistakes
- **Min Stack: not syncing pop on both stacks:** Every `pop()` on the main stack must also pop from the min-stack, or they go out of sync.
- **Queue from Stacks: transferring when output is non-empty:** Only transfer from input to output when output is *empty*. Transferring prematurely scrambles the order.
- **Frequency Stack: not decrementing maxFreq:** After popping the last element at `maxFreq`, you must decrement `maxFreq` — otherwise the next `pop()` will look at an empty group.
- **Circular Queue: confusing full vs empty:** Both conditions involve `front == rear`. Use a size counter or waste one slot to distinguish them.
- **Not handling the `empty()` check properly in Queue from Stacks:** Must check *both* stacks are empty.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Min Stack | Medium | Parallel min-stack tracks running minimum; push min(val, currentMin) | https://leetcode.com/problems/min-stack/ |
| 2 | Implement Queue using Stacks | Easy | Two stacks — lazy transfer from input to output only when output is empty | https://leetcode.com/problems/implement-queue-using-stacks/ |
| 3 | Implement Stack using Queues | Easy | Push costly: enqueue then rotate all previous elements behind it | https://leetcode.com/problems/implement-stack-using-queues/ |
| 4 | Design a Stack With Increment Operation | Medium | Lazy increment: store increment values and apply only on pop | https://leetcode.com/problems/design-a-stack-with-increment-operation/ |
| 5 | Maximum Frequency Stack | Hard | Group by frequency + track maxFreq; most frequent and most recent pops first | https://leetcode.com/problems/maximum-frequency-stack/ |
| 6 | LRU Cache | Medium | HashMap + doubly-linked list; move accessed node to head, evict from tail | https://leetcode.com/problems/lru-cache/ |
| 7 | Flatten Nested List Iterator | Medium | Use a stack of list iterators; on hasNext, flatten until a plain integer is on top | https://leetcode.com/problems/flatten-nested-list-iterator/ |
| 8 | Design Circular Queue | Medium | Fixed-size array with front/rear pointers; use modular arithmetic for wrap-around | https://leetcode.com/problems/design-circular-queue/ |
| 9 | Dinner Plate Stacks | Hard | Array of stacks + TreeSet/min-heap to track leftmost non-full stack for push | https://leetcode.com/problems/dinner-plate-stacks/ |
| 10 | Design Browser History | Medium | Two stacks (back and forward) or a list with pointer; clear forward history on visit | https://leetcode.com/problems/design-browser-history/ |
