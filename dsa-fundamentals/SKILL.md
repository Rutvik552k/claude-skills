---
name: "dsa-fundamentals"
description: "Data structures and algorithms fundamentals: linked lists, BST, AVL trees, heaps, queues, sets, sorting algorithms, and numeric methods. Use when solving algorithmic problems, choosing data structures, analyzing complexity, or implementing efficient solutions."
origin: ECC
---

# DSA Fundamentals

Comprehensive knowledge base for data structures and algorithms covering core structures (linked lists, trees, heaps, queues, sets), sorting algorithms (comparison and non-comparison based), and numeric methods. Every topic includes Big-O complexity analysis for all operations.

## When to Activate

- User asks about choosing between data structures for a problem
- Implementing or optimizing a sorting algorithm
- Analyzing time/space complexity of code
- Solving coding interview or competitive programming problems
- Working with trees, heaps, queues, linked lists, or sets
- Implementing numeric algorithms (GCD, primality, base conversion)
- Reviewing code for algorithmic efficiency

## Topics Index

### Linked Lists
- **Singly Linked List**: node structure (data + next pointer), insertion (head O(1), tail O(n), position O(n)), deletion (by value, by position), traversal, search O(n)
- **Doubly Linked List**: node structure (data + next + prev), insertion/deletion at both ends O(1), reverse traversal, memory overhead vs singly
- **Common Patterns**: two-pointer technique (fast/slow for cycle detection — Floyd's algorithm), list reversal (iterative and recursive), merge two sorted lists, find middle element, detect and remove cycles
- **Complexity Summary**: access O(n), search O(n), insert at head O(1), insert at tail O(n) singly / O(1) doubly, delete O(n)

### Binary Search Trees
- **BST Property**: for every node, all left descendants < node < all right descendants
- **Operations**: insert O(h), search O(h), delete — three cases: leaf node, one child, two children (in-order successor/predecessor replacement)
- **Tree Traversals**: in-order (left-root-right, produces sorted output), pre-order (root-left-right, useful for serialization/copying), post-order (left-right-root, useful for deletion), breadth-first/level-order (uses queue)
- **Finding Min/Max**: follow left/right pointers to leaf — O(h)
- **Degeneracy**: unbalanced BST degrades to O(n) — motivates balanced trees

### AVL Trees
- **Balance Factor**: height(left subtree) - height(right subtree), must be in {-1, 0, 1}
- **Rotations**: left rotation (right-heavy fix), right rotation (left-heavy fix), left-right double rotation, right-left double rotation
- **Rebalancing**: after insert or delete, walk up ancestors updating heights, apply rotations where |balance factor| > 1
- **Guaranteed Performance**: height is O(log n), so all operations are O(log n) worst case

### Heaps
- **Binary Heap Property**: max-heap (parent >= children), min-heap (parent <= children)
- **Array Representation**: parent at index i, left child at 2i+1, right child at 2i+2, parent at floor((i-1)/2)
- **Operations**: insert — add at end, bubble up O(log n); extract-max/min — swap root with last, bubble down O(log n); peek O(1); build-heap from array O(n) via bottom-up heapify
- **Heap Sort**: build max-heap O(n), repeatedly extract max O(n log n), in-place, not stable
- **Priority Queue**: heap is the standard implementation — insert O(log n), extract-min/max O(log n), peek O(1)

### Sets
- **Set Operations**: union, intersection, difference, symmetric difference, subset/superset testing, membership testing
- **Implementations**: hash set — O(1) average for insert/delete/lookup, O(n) worst; balanced BST set — O(log n) guaranteed; bit vector — O(1) operations for small integer domains
- **Ordered vs Unordered**: hash set is unordered O(1), tree set is ordered O(log n) with range queries

### Queues
- **Standard Queue (FIFO)**: enqueue at rear, dequeue from front, circular array implementation avoids shifting — all operations O(1) amortized
- **Priority Queue**: elements have priorities, extract-min/max first — heap-backed O(log n) insert/extract
- **Double-Ended Queue (Deque)**: push/pop from both ends O(1), useful for sliding window problems, implemented with circular buffer or doubly linked list

### Sorting Algorithms
- **Bubble Sort**: compare adjacent pairs, swap if out of order, repeat until sorted. O(n^2) average/worst, O(n) best (with early termination flag). Stable. In-place.
- **Insertion Sort**: build sorted prefix one element at a time. O(n^2) average/worst, O(n) best (nearly sorted input). Stable. In-place. Good for small n or nearly sorted data.
- **Merge Sort**: divide in half, sort recursively, merge sorted halves. O(n log n) all cases. Stable. O(n) extra space. Preferred when stability required.
- **Quick Sort**: choose pivot, partition around it, recurse on halves. O(n log n) average, O(n^2) worst (bad pivot). Not stable. In-place (O(log n) stack). Median-of-three or random pivot mitigates worst case.
- **Shell Sort**: generalized insertion sort with diminishing gap sequences. O(n^1.25) to O(n^1.5) depending on gap sequence (Knuth: 3k+1, Sedgewick). In-place. Not stable.
- **Radix Sort**: sort digit by digit using stable sub-sort (counting sort). O(d * (n + k)) where d = digits, k = radix. Not comparison-based — can beat O(n log n) for integers.

### Numeric Algorithms
- **Primality Testing**: trial division O(sqrt(n)), sieve of Eratosthenes O(n log log n) for range, Miller-Rabin probabilistic test
- **Base Conversions**: repeated division for decimal to base-b, Horner's method for base-b to decimal, hex/octal/binary shortcuts
- **Greatest Common Divisor**: Euclidean algorithm — gcd(a, b) = gcd(b, a mod b), O(log(min(a,b))). Extended GCD gives Bezout coefficients: ax + by = gcd(a,b). Used for modular inverse.
- **Factorial**: iterative O(n) multiplication, recursive with O(n) stack, overflow at ~20! for 64-bit integers

## Key Concepts

### Big-O Complexity Classes
| Class | Name | Example |
|-------|------|---------|
| O(1) | Constant | Hash table lookup, array index access |
| O(log n) | Logarithmic | Binary search, balanced BST operations |
| O(n) | Linear | Linear search, single traversal |
| O(n log n) | Linearithmic | Merge sort, heap sort, optimal comparison sort |
| O(n^2) | Quadratic | Bubble sort, insertion sort (worst), nested loops |
| O(2^n) | Exponential | Recursive Fibonacci (naive), power set |

### Choosing the Right Data Structure
| Need | Best Structure | Why |
|------|---------------|-----|
| Fast lookup by key | Hash map/set | O(1) average |
| Sorted order + fast ops | Balanced BST (AVL/Red-Black) | O(log n) with ordering |
| Fast min/max extraction | Heap | O(1) peek, O(log n) extract |
| FIFO processing | Queue | O(1) enqueue/dequeue |
| LIFO processing | Stack | O(1) push/pop |
| Frequent insert/delete at ends | Deque or doubly linked list | O(1) at both ends |
| Sequential access, no random | Linked list | O(1) insert at head |

### Sorting Algorithm Selection
| Situation | Best Choice | Reason |
|-----------|------------|--------|
| General purpose | Quick sort | O(n log n) average, low constant, in-place |
| Stability required | Merge sort | O(n log n), stable |
| Nearly sorted data | Insertion sort | O(n) best case |
| Small n (< 20) | Insertion sort | Low overhead |
| Integer keys, bounded range | Radix/counting sort | O(n) possible |
| Memory constrained | Heap sort | O(1) extra space, O(n log n) guaranteed |

## Problem-Solving Patterns

| Pattern | When to Use | Key Insight |
|---------|-------------|-------------|
| Two Pointers | Linked list cycle detection, finding pairs | Fast/slow pointers meet in cycle; converging pointers find pairs in O(n) |
| Divide and Conquer | Sorting, searching, tree problems | Split problem in half, solve recursively, combine — gives O(n log n) |
| Sliding Window | Subarray/substring problems with size constraint | Maintain window with deque, expand right, shrink left — O(n) |
| Greedy | Optimal substructure without overlapping subproblems | Local optimal choice leads to global optimal (prove with exchange argument) |
| Binary Search | Sorted array queries, search space reduction | Halve search space each step — O(log n) |

## Common Pitfalls

| Pitfall | Correct Approach |
|---------|-----------------|
| Using BST without considering worst case | Use balanced BST (AVL/Red-Black) or hash table |
| Choosing bubble sort for large data | Use merge sort or quick sort for n > ~1000 |
| Forgetting O(n) space in merge sort | Use heap sort if memory is constrained |
| Not handling empty list / null pointers | Always check head == null before traversal |
| Off-by-one in binary search | Use `lo <= hi` with `mid = lo + (hi - lo) / 2` to avoid overflow |
| Ignoring stability requirement | Merge sort is stable; quick sort and heap sort are not |
| Building heap with repeated insert O(n log n) | Use bottom-up heapify for O(n) build |

## Source Material

- *Data Structures and Algorithms: Annotated Reference with Examples* by Granville Barnett and Luca Del Tongo (2008)
