## Floyd's Cycle Detection Algorithm

### Core Concept

Floyd's algorithm detects cycles in sequences by using two pointers moving at different speeds - a "slow" pointer (tortoise) that moves one step at a time, and a "fast" pointer (hare) that moves two steps at a time.

### Key Principles

**1. Mathematical Foundation**

- If there's a cycle, the fast pointer will eventually catch up to the slow pointer
- The meeting point is guaranteed to be inside the cycle
- Time complexity: O(n) where n is the number of nodes
- Space complexity: O(1) - only uses two pointers

**2. Two-Phase Approach**

**Phase 1: Cycle Detection**

- Initialize both pointers at the start
- Move slow pointer by 1 step, fast pointer by 2 steps
- If they meet, a cycle exists
- If fast pointer reaches null/end, no cycle exists

**Phase 2: Finding Cycle Start (if needed)**

- Reset one pointer to the start
- Move both pointers one step at a time
- They'll meet at the cycle's starting point

### Mathematical Proof

Let's denote:

- μ (mu) = distance from start to cycle beginning
- λ (lambda) = cycle length
- k = distance from cycle start to where pointers first meet

When pointers meet:

- Slow pointer traveled: μ + a*λ + k (for some integer a)
- Fast pointer traveled: μ + b*λ + k (for some integer b > a)
- Since fast moves twice as fast: 2(μ + a_λ + k) = μ + b_λ + k
- Solving: μ + k = (b-2a)λ

This proves that the distance from start to cycle beginning (μ) equals the distance from meeting point to cycle beginning when traveling backwards.

### Common Applications

**1. Linked List Cycle Detection**

```python
def hasCycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False
```

**2. Finding Duplicate Number**

- Treat array as implicit linked list where arr[i] points to index arr[i]
- Used in problems like "Find the Duplicate Number" (LeetCode 287)

**3. Cycle Length Calculation** Once cycle is detected:

```python
def getCycleLength(meetingPoint):
    current = meetingPoint
    length = 0
    while True:
        current = current.next
        length += 1
        if current == meetingPoint:
            return length
```
