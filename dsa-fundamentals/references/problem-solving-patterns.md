# DSA Problem-Solving Patterns

## Pattern 1: Two Pointers

### When to Use
- Linked list cycle detection
- Finding pairs that sum to target in sorted array
- Removing duplicates from sorted array
- Palindrome checking
- Merging two sorted arrays

### Variants

**Fast/Slow Pointers (Floyd's Cycle Detection)**
```
hasCycle(head):
    slow = head, fast = head
    while fast != null and fast.next != null:
        slow = slow.next
        fast = fast.next.next
        if slow == fast: return true
    return false
```
Also finds cycle start: after detection, reset one pointer to head, advance both by 1 until they meet.

**Converging Pointers (Sorted Array)**
```
twoSum(sorted_arr, target):
    left = 0, right = n - 1
    while left < right:
        sum = arr[left] + arr[right]
        if sum == target: return (left, right)
        if sum < target: left++
        else: right--
```

## Pattern 2: Sliding Window

### When to Use
- Maximum/minimum subarray of size k
- Longest substring without repeating characters
- Subarray with sum equal to target

### Template
```
slidingWindow(arr, condition):
    left = 0
    window_state = initial
    best = initial
    for right from 0 to n-1:
        // expand window: add arr[right] to window_state
        while window violates condition:
            // shrink window: remove arr[left] from window_state
            left++
        // update best with current window
    return best
```

**Fixed-size window**: just move both pointers together after window reaches size k.
**Variable-size window**: expand right, shrink left when condition violated.

## Pattern 3: Binary Search

### When to Use
- Searching in sorted array
- Finding boundary (first/last occurrence)
- Search space reduction (minimum value satisfying condition)

### Standard Template
```
binarySearch(arr, target):
    lo = 0, hi = n - 1
    while lo <= hi:
        mid = lo + (hi - lo) / 2    // avoids overflow
        if arr[mid] == target: return mid
        if arr[mid] < target: lo = mid + 1
        else: hi = mid - 1
    return -1
```

### Find First Occurrence (Lower Bound)
```
lowerBound(arr, target):
    lo = 0, hi = n
    while lo < hi:
        mid = lo + (hi - lo) / 2
        if arr[mid] < target: lo = mid + 1
        else: hi = mid
    return lo
```

### Binary Search on Answer
```
// Find minimum x such that condition(x) is true
lo = min_possible, hi = max_possible
while lo < hi:
    mid = lo + (hi - lo) / 2
    if condition(mid): hi = mid
    else: lo = mid + 1
return lo
```

## Pattern 4: Divide and Conquer

### When to Use
- Sorting (merge sort, quick sort)
- Tree problems (height, diameter, LCA)
- Maximum subarray (Kadane's is better, but D&C is the pattern)
- Closest pair of points

### Template
```
divideAndConquer(problem):
    if problem is base case: return base_solution
    left, right = split(problem)
    left_result = divideAndConquer(left)
    right_result = divideAndConquer(right)
    return combine(left_result, right_result)
```

### Recurrence Analysis (Master Theorem)
T(n) = aT(n/b) + O(n^d)
- If d < log_b(a): T(n) = O(n^(log_b(a)))
- If d == log_b(a): T(n) = O(n^d * log n)
- If d > log_b(a): T(n) = O(n^d)

| Algorithm | a | b | d | Result |
|-----------|---|---|---|--------|
| Binary search | 1 | 2 | 0 | O(log n) |
| Merge sort | 2 | 2 | 1 | O(n log n) |
| Karatsuba multiply | 3 | 2 | 1 | O(n^1.585) |

## Pattern 5: Stack-Based Processing

### When to Use
- Balanced parentheses checking
- Expression evaluation (infix to postfix)
- Next greater/smaller element
- Monotonic stack problems

### Monotonic Stack (Next Greater Element)
```
nextGreater(arr):
    result = [-1] * n
    stack = []
    for i from 0 to n-1:
        while stack is not empty and arr[i] > arr[stack.top()]:
            idx = stack.pop()
            result[idx] = arr[i]
        stack.push(i)
    return result
```

## Pattern 6: BFS/DFS on Trees and Graphs

### When to Use
- Tree traversal (DFS: in/pre/post-order; BFS: level-order)
- Shortest path in unweighted graph (BFS)
- Connected components (DFS/BFS)
- Topological sort (DFS)
- Cycle detection (DFS with coloring)

### BFS Template (Graph)
```
bfs(start):
    queue = [start]
    visited = {start}
    while queue:
        node = queue.dequeue()
        process(node)
        for neighbor in node.neighbors:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.enqueue(neighbor)
```

### DFS Template (Graph)
```
dfs(node, visited):
    visited.add(node)
    process(node)
    for neighbor in node.neighbors:
        if neighbor not in visited:
            dfs(neighbor, visited)
```

## Complexity Cheat Sheet

| Operation | Array | Linked List | BST (balanced) | Hash Table | Heap |
|-----------|-------|-------------|----------------|------------|------|
| Access by index | O(1) | O(n) | - | - | - |
| Search | O(n) | O(n) | O(log n) | O(1) avg | O(n) |
| Insert | O(n) | O(1) head | O(log n) | O(1) avg | O(log n) |
| Delete | O(n) | O(n) | O(log n) | O(1) avg | O(log n) |
| Min/Max | O(n) | O(n) | O(log n) | O(n) | O(1) peek |
| Sorted order | O(n log n) | O(n log n) | O(n) in-order | O(n log n) | O(n log n) |
