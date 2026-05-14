# Tree Operations Reference

## Binary Search Tree (BST)

### BST Property
For every node N: all values in left subtree < N.value < all values in right subtree.

### Insert
```
insert(root, value):
    if root is null: return new Node(value)
    if value < root.value:
        root.left = insert(root.left, value)
    else if value > root.value:
        root.right = insert(root.right, value)
    return root
```
Complexity: O(h) where h = tree height. Balanced: O(log n). Degenerate: O(n).

### Search
```
search(root, value):
    if root is null: return null
    if value == root.value: return root
    if value < root.value: return search(root.left, value)
    return search(root.right, value)
```

### Delete (Three Cases)
```
delete(root, value):
    if root is null: return null
    if value < root.value:
        root.left = delete(root.left, value)
    else if value > root.value:
        root.right = delete(root.right, value)
    else:  // found node
        Case 1 - Leaf: return null
        Case 2 - One child: return the non-null child
        Case 3 - Two children:
            successor = findMin(root.right)  // in-order successor
            root.value = successor.value
            root.right = delete(root.right, successor.value)
    return root
```

### Traversals

| Traversal | Order | Use Case | Output for [4,2,6,1,3,5,7] |
|-----------|-------|----------|----------------------------|
| In-order | Left, Root, Right | Sorted output | 1,2,3,4,5,6,7 |
| Pre-order | Root, Left, Right | Copy/serialize tree | 4,2,1,3,6,5,7 |
| Post-order | Left, Right, Root | Delete tree, evaluate expression | 1,3,2,5,7,6,4 |
| Level-order | Level by level | BFS, level-based processing | 4,2,6,1,3,5,7 |

```
inOrder(node):
    if node is null: return
    inOrder(node.left)
    visit(node)
    inOrder(node.right)

levelOrder(root):
    queue = [root]
    while queue is not empty:
        node = queue.dequeue()
        visit(node)
        if node.left: queue.enqueue(node.left)
        if node.right: queue.enqueue(node.right)
```

## AVL Tree

### Balance Factor
```
balanceFactor(node) = height(node.left) - height(node.right)
```
Must be in {-1, 0, 1} for every node. Violation triggers rotation.

### Four Rotation Cases

**Case 1: Left-Left (LL) — Right Rotation**
```
        z                y
       / \             /   \
      y   T4          x     z
     / \       →     / \   / \
    x   T3          T1 T2 T3 T4
   / \
  T1  T2

rightRotate(z):
    y = z.left
    T3 = y.right
    y.right = z
    z.left = T3
    update heights of z, then y
    return y
```

**Case 2: Right-Right (RR) — Left Rotation**
```
    z                  y
   / \               /   \
  T1   y            z     x
      / \    →     / \   / \
     T2  x       T1  T2 T3 T4
        / \
       T3  T4

leftRotate(z):
    y = z.right
    T2 = y.left
    y.left = z
    z.right = T2
    update heights of z, then y
    return y
```

**Case 3: Left-Right (LR) — Left then Right**
```
      z               z              x
     / \             / \           /   \
    y   T4          x   T4        y     z
   / \       →     / \      →   / \   / \
  T1  x           y   T3      T1  T2 T3 T4
     / \         / \
    T2  T3      T1  T2

Step 1: leftRotate(y)
Step 2: rightRotate(z)
```

**Case 4: Right-Left (RL) — Right then Left**
```
    z                z                x
   / \              / \             /   \
  T1   y           T1   x         z     y
      / \    →         / \   →   / \   / \
     x   T4          T2   y    T1 T2  T3 T4
    / \                   / \
   T2  T3               T3  T4

Step 1: rightRotate(y)
Step 2: leftRotate(z)
```

### AVL Insert
```
insert(node, value):
    1. Standard BST insert
    2. Update height: node.height = 1 + max(height(left), height(right))
    3. Get balance factor
    4. Four rebalancing cases:
       - balance > 1 and value < node.left.value  → rightRotate(node)        [LL]
       - balance < -1 and value > node.right.value → leftRotate(node)         [RR]
       - balance > 1 and value > node.left.value   → leftRotate(left), then rightRotate(node)  [LR]
       - balance < -1 and value < node.right.value  → rightRotate(right), then leftRotate(node) [RL]
    return node
```

### Complexity Guarantee
AVL height h satisfies: h <= 1.44 * log2(n+2) - 0.328
All operations: O(log n) guaranteed.

## Heap Operations Detail

### Insert (Bubble Up)
```
insert(heap, value):
    heap.append(value)
    i = heap.size - 1
    while i > 0:
        parent = (i - 1) / 2
        if heap[i] > heap[parent]:  // max-heap
            swap(heap[i], heap[parent])
            i = parent
        else: break
```

### Extract Max (Bubble Down)
```
extractMax(heap):
    max = heap[0]
    heap[0] = heap[last]
    heap.removeLast()
    heapifyDown(heap, 0)
    return max

heapifyDown(heap, i):
    largest = i
    left = 2*i + 1
    right = 2*i + 2
    if left < size and heap[left] > heap[largest]: largest = left
    if right < size and heap[right] > heap[largest]: largest = right
    if largest != i:
        swap(heap[i], heap[largest])
        heapifyDown(heap, largest)
```

### Build Heap — O(n)
```
buildMaxHeap(arr):
    for i from (n/2 - 1) downto 0:  // start from last non-leaf
        heapifyDown(arr, i)
```
Why O(n) not O(n log n): Most nodes are near the bottom and heapify very little. Sum of work = O(n) by geometric series argument.
