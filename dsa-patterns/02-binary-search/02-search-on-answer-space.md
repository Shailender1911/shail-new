# Pattern: Search on Answer Space

## How to Identify This Pattern

### Trigger Words
- "minimize the maximum" or "maximize the minimum"
- "at most K groups/workers/days"
- "minimum capability/capacity/speed to finish"
- "allocate", "distribute", "split" fairly
- "is it possible to achieve X within a constraint?"
- "least", "most", "optimal" combined with a feasibility condition

### Constraint Clues
- The answer is an integer in a known range (e.g., 1 to 10⁹)
- There is a **monotonic feasibility function**: if answer `x` works, then all values more relaxed than `x` also work
- A brute-force check for a single candidate answer is O(n) or O(n log n)
- Constraints show n up to 10⁵ and values up to 10⁹, hinting at O(n log M) where M is the answer range
- The problem asks for an optimal threshold, rate, distance, or capacity

### When to Use This vs Similar Patterns
| Consideration | Search on Answer Space | Classic Binary Search | Greedy | Dynamic Programming |
|---|---|---|---|---|
| Search space | Range of possible answers | Indices of a sorted array | N/A | Subproblem states |
| Key insight | Monotonic feasibility check | Sorted data | Local optimal = global optimal | Overlapping subproblems |
| Typical pattern | Binary search + O(n) feasibility | Direct index comparison | Sort + scan | Table filling |
| When it fails | No monotonicity in feasibility | Data not sorted | Greedy choice not provably optimal | State space too large |

## Core Approach

1. **Define the search space:** Identify the minimum possible answer (`lo`) and the maximum possible answer (`hi`). Typically `lo = max(array)` or `1`, and `hi = sum(array)` or some upper bound.
2. **Write a feasibility function:** `canAchieve(mid)` returns `true` if the task can be completed with the candidate answer `mid` within the given constraints (e.g., within K days, using at most K workers).
3. **Binary search on the answer:**
   - While `lo < hi`:
     - Compute `mid = lo + (hi - lo) / 2`.
     - If `canAchieve(mid)` is true, the answer might be smaller → `hi = mid`.
     - Otherwise, the answer must be larger → `lo = mid + 1`.
4. **Return `lo`** — it converges to the smallest feasible answer.

> For "maximize the minimum" problems, flip the logic: if `canAchieve(mid)` is true, we want a bigger answer → `lo = mid + 1`, and return `lo - 1` (or adjust bounds to `hi = mid - 1`, `lo = mid`).

## Java Template

### Minimize the Maximum (e.g., Split Array Largest Sum)
```java
public int minimizeMax(int[] nums, int k) {
    int lo = max(nums);
    int hi = sum(nums);

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (canSplit(nums, mid, k)) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    return lo;
}

private boolean canSplit(int[] nums, int maxSum, int k) {
    int groups = 1, currentSum = 0;
    for (int num : nums) {
        if (currentSum + num > maxSum) {
            groups++;
            currentSum = num;
            if (groups > k) return false;
        } else {
            currentSum += num;
        }
    }
    return true;
}
```

### Maximize the Minimum (e.g., Aggressive Cows / Magnetic Force)
```java
public int maximizeMin(int[] positions, int k) {
    Arrays.sort(positions);
    int lo = 1;
    int hi = positions[positions.length - 1] - positions[0];

    while (lo < hi) {
        int mid = lo + (hi - lo + 1) / 2;  // upper-mid to avoid infinite loop
        if (canPlace(positions, mid, k)) {
            lo = mid;
        } else {
            hi = mid - 1;
        }
    }
    return lo;
}

private boolean canPlace(int[] positions, int minDist, int k) {
    int count = 1, lastPos = positions[0];
    for (int i = 1; i < positions.length; i++) {
        if (positions[i] - lastPos >= minDist) {
            count++;
            lastPos = positions[i];
            if (count >= k) return true;
        }
    }
    return false;
}
```

### Rate-Based (e.g., Koko Eating Bananas)
```java
public int minEatingSpeed(int[] piles, int h) {
    int lo = 1, hi = max(piles);

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (canFinish(piles, mid, h)) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    return lo;
}

private boolean canFinish(int[] piles, int speed, int h) {
    long hours = 0;
    for (int pile : piles) {
        hours += (pile + speed - 1) / speed;  // ceiling division
    }
    return hours <= h;
}
```

## Dry Run Example

**Problem:** Koko Eating Bananas — `piles = [3, 6, 7, 11]`, `h = 8`.

Search space: `lo = 1`, `hi = 11` (max pile).

Feasibility: at speed `k`, hours needed = ⌈3/k⌉ + ⌈6/k⌉ + ⌈7/k⌉ + ⌈11/k⌉. Must be ≤ 8.

| Step | lo | hi | mid | Hours at mid | ≤ 8? | Action |
|------|----|----|-----|-------------|------|--------|
| 1 | 1 | 11 | 6 | 1+1+2+2 = 6 | Yes | hi = 6 |
| 2 | 1 | 6 | 3 | 1+2+3+4 = 10 | No | lo = 4 |
| 3 | 4 | 6 | 5 | 1+2+2+3 = 8 | Yes | hi = 5 |
| 4 | 4 | 5 | 4 | 1+2+2+3 = 8 | Yes | hi = 4 |
| 5 | 4 | 4 | — | — | — | lo == hi → answer = 4 ✓ |

**Result:** Minimum speed = `4`.

## Complexity
- **Time:** O(n × log M) — where n is array length and M is the answer range (hi − lo). Each binary search step runs an O(n) feasibility check.
- **Space:** O(1) — only a few variables for binary search and the greedy scan.

## Edge Cases
- Single element in array — answer is that element itself (or trivially feasible)
- `k = 1` — one group/worker takes everything; answer = sum or max
- `k = n` — each element is its own group; answer = max element
- Very large values — use `long` for sums and multiplications to avoid overflow
- All elements equal — any split is equally good; answer is predictable
- Minimum possible answer equals maximum — single valid answer, loop terminates immediately

## Common Mistakes
- **Wrong search bounds:** Setting `lo` too high or `hi` too low excludes the answer. Always reason about the absolute min/max the answer can be.
- **Ceiling division errors:** Use `(a + b - 1) / b` for ceiling division, not `Math.ceil((double)a / b)` which has floating-point issues.
- **Infinite loop with `lo = mid`:** When using `lo = mid` (maximize problems), compute `mid = lo + (hi - lo + 1) / 2` (round up) to prevent `lo` from stalling when `hi = lo + 1`.
- **Off-by-one in feasibility:** Starting the greedy count at 0 vs 1 — for splitting, you always have at least 1 group before any split.
- **Integer overflow:** When `hi = sum(array)` and values are up to 10⁹ with n up to 10⁵, the sum exceeds `int` range. Use `long`.
- **Confusing minimize vs maximize:** Minimize-the-max uses `hi = mid` on success. Maximize-the-min uses `lo = mid` on success. Mixing them up inverts the answer.

## Practice Problems

| # | Problem | Difficulty | Key Insight | LeetCode Link |
|---|---------|-----------|-------------|---------------|
| 1 | Koko Eating Bananas | Medium | Binary search on speed 1..max(piles); feasibility = total hours ≤ h | https://leetcode.com/problems/koko-eating-bananas/ |
| 2 | Split Array Largest Sum | Hard | Minimize the max subarray sum; greedily split when running sum exceeds mid | https://leetcode.com/problems/split-array-largest-sum/ |
| 3 | Capacity To Ship Packages Within D Days | Medium | Minimize ship capacity; greedily fill each day until weight exceeds mid | https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/ |
| 4 | Minimum Number of Days to Make m Bouquets | Medium | Binary search on days; feasibility checks consecutive bloomed flowers | https://leetcode.com/problems/minimum-number-of-days-to-make-m-bouquets/ |
| 5 | Magnetic Force Between Two Balls | Medium | Maximize minimum distance; sort positions, greedily place balls | https://leetcode.com/problems/magnetic-force-between-two-balls/ |
| 6 | Allocate Minimum Pages | Medium | Minimize the max pages per student; same structure as Split Array Largest Sum | https://leetcode.com/problems/allocate-minimum-number-of-pages/ |
| 7 | Find the Smallest Divisor Given a Threshold | Medium | Binary search on divisor; feasibility = sum of ⌈nums[i]/mid⌉ ≤ threshold | https://leetcode.com/problems/find-the-smallest-divisor-given-a-threshold/ |
| 8 | Minimize Max Distance to Gas Station | Medium | Binary search on distance (floating point); count segments that need extra stations | https://leetcode.com/problems/minimize-max-distance-to-gas-station/ |
| 9 | Aggressive Cows (SPOJ / GFG) | Medium | Maximize minimum distance between cows; sort + greedy placement in feasibility | https://leetcode.com/problems/magnetic-force-between-two-balls/ |
| 10 | Maximum Candies Allocated to K Children | Medium | Maximize candies per child; feasibility = total children served ≥ k at that size | https://leetcode.com/problems/maximum-candies-allocated-to-k-children/ |
