# Sorting Algorithms Comparison

## Complete Comparison Table

| Algorithm | Best | Average | Worst | Space | Stable | In-Place | Adaptive |
|-----------|------|---------|-------|-------|--------|----------|----------|
| Bubble Sort | O(n) | O(n^2) | O(n^2) | O(1) | Yes | Yes | Yes |
| Insertion Sort | O(n) | O(n^2) | O(n^2) | O(1) | Yes | Yes | Yes |
| Selection Sort | O(n^2) | O(n^2) | O(n^2) | O(1) | No | Yes | No |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | No | No |
| Quick Sort | O(n log n) | O(n log n) | O(n^2) | O(log n) | No | Yes | No |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No | Yes | No |
| Shell Sort | O(n log n) | O(n^1.25) | O(n^1.5) | O(1) | No | Yes | Yes |
| Radix Sort | O(d*n) | O(d*n) | O(d*n) | O(n+k) | Yes | No | No |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes | No | No |

Where d = number of digits, k = range of key values, n = number of elements.

## Algorithm Details

### Bubble Sort
```
for i from 0 to n-1:
    swapped = false
    for j from 0 to n-i-2:
        if arr[j] > arr[j+1]:
            swap(arr[j], arr[j+1])
            swapped = true
    if not swapped: break  // early termination — makes best case O(n)
```
**When to use**: Educational only. Never in production.

### Insertion Sort
```
for i from 1 to n-1:
    key = arr[i]
    j = i - 1
    while j >= 0 and arr[j] > key:
        arr[j+1] = arr[j]
        j = j - 1
    arr[j+1] = key
```
**When to use**: Small arrays (n < 20), nearly sorted data, online sorting (data arrives incrementally). Used as base case in hybrid sorts (Timsort, Introsort).

### Merge Sort
```
mergeSort(arr, lo, hi):
    if lo >= hi: return
    mid = (lo + hi) / 2
    mergeSort(arr, lo, mid)
    mergeSort(arr, mid+1, hi)
    merge(arr, lo, mid, hi)

merge(arr, lo, mid, hi):
    create temp arrays L = arr[lo..mid], R = arr[mid+1..hi]
    i = 0, j = 0, k = lo
    while i < |L| and j < |R|:
        if L[i] <= R[j]: arr[k++] = L[i++]   // <= preserves stability
        else: arr[k++] = R[j++]
    copy remaining from L or R
```
**When to use**: When stability is required, for linked lists (O(1) extra space), external sorting, guaranteed O(n log n).

### Quick Sort
```
quickSort(arr, lo, hi):
    if lo >= hi: return
    p = partition(arr, lo, hi)
    quickSort(arr, lo, p-1)
    quickSort(arr, p+1, hi)

partition(arr, lo, hi):     // Lomuto scheme
    pivot = arr[hi]
    i = lo - 1
    for j from lo to hi-1:
        if arr[j] <= pivot:
            i++
            swap(arr[i], arr[j])
    swap(arr[i+1], arr[hi])
    return i + 1
```
**Pivot strategies**: Last element (simple, O(n^2) on sorted), random (expected O(n log n)), median-of-three (good practical choice).

**When to use**: General-purpose default. Best average-case constant factor. Use Introsort (switch to heap sort after recursion depth exceeds 2*log(n)) to guarantee O(n log n).

### Heap Sort
```
heapSort(arr):
    buildMaxHeap(arr)       // O(n)
    for i from n-1 downto 1:
        swap(arr[0], arr[i])
        heapify(arr, 0, i)  // O(log n)
```
**When to use**: When O(n log n) worst case is required with O(1) space. Poor cache locality makes it slower than quick sort in practice.

### Radix Sort
```
radixSort(arr, d):
    for digit from 0 to d-1:
        stableSort(arr, by digit)  // counting sort on current digit
```
**When to use**: Integers or strings with bounded length. O(d * n) can beat O(n log n) comparison sorts when d is small constant.

## Decision Flowchart

```
Is stability required?
├── Yes → Merge Sort (or Timsort)
└── No
    ├── Is worst-case O(n log n) required?
    │   ├── Yes → Heap Sort
    │   └── No → Quick Sort (with random pivot)
    ├── Is input nearly sorted?
    │   └── Yes → Insertion Sort or Timsort
    ├── Are keys integers with small range?
    │   └── Yes → Counting Sort or Radix Sort
    └── Is n very small (< 20)?
        └── Yes → Insertion Sort
```
