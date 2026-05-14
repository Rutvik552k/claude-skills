# Path Testing and Data Flow Analysis

How to construct program graphs, compute cyclomatic complexity, identify basis paths, perform define-use analysis, and apply data flow coverage criteria.

---

## 1. Constructing a Program Graph from Code

A program graph represents the control flow of a program. Each node is a statement or statement fragment, and each edge represents possible flow of control.

### Example Code: Commission Calculator

```python
def commission(sales, price):
    # Node 1: entry
    total = sales * price          # Node 2
    if total > 10000:              # Node 3 (decision)
        rate = 0.15                # Node 4
    elif total > 5000:             # Node 5 (decision)
        rate = 0.10                # Node 6
    else:
        rate = 0.05                # Node 7
    # Node 8: merge
    commission = total * rate      # Node 9
    if commission < 100:           # Node 10 (decision)
        commission = 100           # Node 11
    # Node 12: merge
    return commission              # Node 13 (exit)
```

### Program Graph

```
Nodes: {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13}

Edges:
  1 -> 2
  2 -> 3
  3 -> 4    (total > 10000 is True)
  3 -> 5    (total > 10000 is False)
  4 -> 8
  5 -> 6    (total > 5000 is True)
  5 -> 7    (total > 5000 is False)
  6 -> 8
  7 -> 8
  8 -> 9
  9 -> 10
  10 -> 11  (commission < 100 is True)
  10 -> 12  (commission < 100 is False)
  11 -> 12
  12 -> 13
```

### Rules for Program Graph Construction

1. **Sequential statements**: connect with a single edge
2. **If-then-else**: decision node has two outgoing edges (true, false)
3. **If-then (no else)**: decision node has two edges; false edge goes to the merge point
4. **While loop**: decision node; true edge goes to loop body; false edge exits; loop body edges back to decision
5. **For loop**: treat as while with initialization before and increment at end of body
6. **Switch/case**: decision node with one edge per case plus default
7. **Try-catch**: normal flow edge plus exception edge to catch block
8. **Return/break/continue**: edge to the appropriate target (exit, loop end, loop start)

---

## 2. DD-Path Graph Construction

DD-paths (Decision-to-Decision paths) compress chains of sequential nodes into single nodes, retaining only the branching structure.

### DD-Path Types

| Type | Description | In-degree | Out-degree |
|---|---|---|---|
| Type 1 | Single node, in-degree or out-degree != 1 | varies | varies |
| Type 2 | Single node, in-degree = 1, out-degree = 1 | 1 | 1 |
| Type 3 | Maximal chain of Type 2 nodes | 1 | 1 |

### Compression Process

1. Identify all decision nodes (out-degree > 1) and merge nodes (in-degree > 1)
2. Each decision node, merge node, entry, and exit becomes a DD-path node
3. Chains of sequential nodes between decisions/merges collapse into single DD-path nodes

### DD-Path Graph for Commission Example

```
DD-Path Nodes:
  A: {1, 2}     (entry + sequential)
  B: {3}        (decision: total > 10000)
  C: {4}        (rate = 0.15)
  D: {5}        (decision: total > 5000)
  E: {6}        (rate = 0.10)
  F: {7}        (rate = 0.05)
  G: {8, 9}     (merge + sequential)
  H: {10}       (decision: commission < 100)
  I: {11}       (commission = 100)
  J: {12, 13}   (merge + exit)

DD-Path Edges:
  A -> B
  B -> C    (true)
  B -> D    (false)
  C -> G
  D -> E    (true)
  D -> F    (false)
  E -> G
  F -> G
  G -> H
  H -> I    (true)
  H -> J    (false)
  I -> J
```

Node count: 10, Edge count: 11

---

## 3. Cyclomatic Complexity Calculation

### Formula

**V(G) = e - n + 2p**

Where:
- **e** = number of edges in the graph
- **n** = number of nodes in the graph
- **p** = number of connected components (usually 1 for a single function)

### Alternative Formulas

- **V(G) = d + 1** where d = number of decision nodes (for a single-component graph)
- **V(G) = number of enclosed regions + 1** (for a planar graph)

### Calculation for Commission Example

Using the DD-path graph:
- e = 11 edges
- n = 10 nodes
- p = 1 connected component

**V(G) = 11 - 10 + 2(1) = 3**

Verification using decision count:
- Decision nodes: B (total > 10000), D (total > 5000), H (commission < 100)
- d = 3
- **V(G) = 3 + 1 = 4**

Wait -- let us recount. Looking at the DD-path graph edges more carefully:

```
A->B, B->C, B->D, C->G, D->E, D->F, E->G, F->G, G->H, H->I, H->J, I->J
```

That is 12 edges, 10 nodes:

**V(G) = 12 - 10 + 2 = 4**

And d = 3 decisions, so V(G) = 3 + 1 = 4. Both formulas agree.

### More Examples

| Function | Edges | Nodes | Components | V(G) | Interpretation |
|---|---|---|---|---|---|
| Linear (no branches) | 5 | 6 | 1 | 1 | Single path; trivial to test |
| Single if-else | 4 | 4 | 1 | 2 | Two paths |
| Nested if-else (2 deep) | 8 | 7 | 1 | 3 | Three independent paths |
| While loop | 4 | 3 | 1 | 3 | Loop entry, body, exit |
| Switch with 4 cases | 6 | 6 | 1 | 2 | Depends on structure |

### Interpretation Guidelines

| V(G) Range | Risk | Testability |
|---|---|---|
| 1-10 | Low | Easy to test; well-structured |
| 11-20 | Moderate | More effort needed; consider refactoring |
| 21-50 | High | Difficult to test; should refactor |
| > 50 | Very high | Untestable in practice; must refactor |

---

## 4. Basis Path Identification

A basis set is a set of V(G) linearly independent paths. Every possible path through the code is a linear combination of basis paths.

### Algorithm: Baseline Method

1. Choose a **baseline path**: the most "normal" execution path through the code (typically the most common or simplest path)
2. For each decision node along the baseline path, create a new path by flipping that decision and keeping as much of the baseline path as possible
3. Continue until you have V(G) paths

### Basis Paths for Commission Example (V(G) = 4)

**Baseline path (P1)**: A -> B -> D -> F -> G -> H -> J
- total <= 5000, commission >= 100 (lowest commission path, no adjustments)

**P2** (flip decision at B): A -> B -> C -> G -> H -> J
- total > 10000, commission >= 100

**P3** (flip decision at D): A -> B -> D -> E -> G -> H -> J
- 5000 < total <= 10000, commission >= 100

**P4** (flip decision at H): A -> B -> D -> F -> G -> H -> I -> J
- total <= 5000, commission < 100 (minimum commission applied)

### Verification

- 4 paths for V(G) = 4: correct
- Each path differs from the baseline in exactly one decision
- Every edge appears in at least one path (edge coverage achieved)
- Every decision outcome (true/false) is exercised (branch coverage achieved)

### Converting Paths to Test Cases

| Path | Condition | Test Input | Expected |
|---|---|---|---|
| P1 | total <= 5000, comm >= 100 | sales=100, price=50 (total=5000) | comm = 250 |
| P2 | total > 10000 | sales=100, price=200 (total=20000) | comm = 3000 |
| P3 | 5000 < total <= 10000 | sales=100, price=80 (total=8000) | comm = 800 |
| P4 | total <= 5000, comm < 100 | sales=10, price=10 (total=100) | comm = 100 (min) |

---

## 5. Define-Use Pair Identification

### Example Code: Grade Calculator

```python
def calculate_grade(scores):
    total = 0                     # Line 1: DEF(total)
    count = 0                     # Line 2: DEF(count)
    for s in scores:              # Line 3: DEF(s), USE(scores)
        if s >= 0:                # Line 4: USE(s) [p-use]
            total = total + s     # Line 5: DEF(total), USE(total), USE(s) [c-use]
            count = count + 1     # Line 6: DEF(count), USE(count) [c-use]
    if count > 0:                 # Line 7: USE(count) [p-use]
        avg = total / count       # Line 8: DEF(avg), USE(total), USE(count) [c-use]
    else:
        avg = 0                   # Line 9: DEF(avg)
    return avg                    # Line 10: USE(avg) [c-use]
```

### Step 1: List All Definitions and Uses

| Variable | Line | Type | Kind |
|---|---|---|---|
| total | 1 | DEF | |
| total | 5 | DEF | |
| total | 5 | USE | c-use (computation) |
| total | 8 | USE | c-use |
| count | 2 | DEF | |
| count | 6 | DEF | |
| count | 6 | USE | c-use |
| count | 7 | USE | p-use (predicate) |
| count | 8 | USE | c-use |
| s | 3 | DEF | |
| s | 4 | USE | p-use |
| s | 5 | USE | c-use |
| scores | (param) | DEF | |
| scores | 3 | USE | c-use |
| avg | 8 | DEF | |
| avg | 9 | DEF | |
| avg | 10 | USE | c-use |

### Step 2: Identify DU-Pairs

A DU-pair (d, u) is a definition at node d and a use at node u where there exists a definition-clear path from d to u.

| Variable | DEF Line | USE Line | Use Kind | Definition-Clear? |
|---|---|---|---|---|
| total | 1 | 5 | c-use | Yes (first iteration) |
| total | 1 | 8 | c-use | Yes (if loop never entered) |
| total | 5 | 5 | c-use | Yes (subsequent iterations) |
| total | 5 | 8 | c-use | Yes (after last iteration) |
| count | 2 | 6 | c-use | Yes (first iteration) |
| count | 2 | 7 | p-use | Yes (if loop never entered) |
| count | 2 | 8 | c-use | Yes (if loop never entered, but count=0 so line 8 not reached) |
| count | 6 | 6 | c-use | Yes (subsequent iterations) |
| count | 6 | 7 | p-use | Yes (after last iteration) |
| count | 6 | 8 | c-use | Yes (after loop, count > 0) |
| s | 3 | 4 | p-use | Yes |
| s | 3 | 5 | c-use | Yes (when s >= 0) |
| avg | 8 | 10 | c-use | Yes |
| avg | 9 | 10 | c-use | Yes |
| scores | param | 3 | c-use | Yes |

### Step 3: DU-Paths

A DU-path is the actual path from definition to use. For variable `total`:

- DU-path(total, 1, 5): 1 -> 2 -> 3 -> 4 -> 5 (first iteration, s >= 0)
- DU-path(total, 5, 5): 5 -> 6 -> 3 -> 4 -> 5 (loop back, next s >= 0)
- DU-path(total, 5, 8): 5 -> 6 -> 3 -> 7 -> 8 (exit loop, count > 0)
- DU-path(total, 1, 8): 1 -> 2 -> 3 -> 7 -> 8 (loop never entered, count = 0, but then line 9 taken not 8)

---

## 6. Coverage Criteria Hierarchy

### Hierarchy Diagram (subsumption relationships)

```
All-Paths
    |
All-DU-Paths
    |
All-Uses
   / \
All-C-Uses    All-P-Uses
   \            /
    All-Defs
        |
   Branch Coverage
        |
  Statement Coverage
```

An arrow from A to B means "A subsumes B" -- achieving A guarantees achieving B.

### Definitions

| Criterion | Requirement |
|---|---|
| **Statement Coverage** | Every statement executed at least once |
| **Branch Coverage** | Every branch (true/false outcome of each decision) executed at least once |
| **All-Defs** | For every variable definition, at least one def-clear path to some use is tested |
| **All-C-Uses** | For every definition, at least one def-clear path to every reachable c-use is tested |
| **All-P-Uses** | For every definition, at least one def-clear path to every reachable p-use is tested |
| **All-Uses** | For every definition, at least one def-clear path to every reachable use (c-use or p-use) is tested |
| **All-DU-Paths** | Every def-clear path from every definition to every reachable use is tested |
| **All-Paths** | Every possible execution path is tested (usually infeasible) |

### Practical Implications

| Criterion | Typical Test Count | Feasibility | Fault Detection |
|---|---|---|---|
| Statement | Low | Always feasible | Catches dead code, crashes |
| Branch | Low-Medium | Always feasible | Catches incorrect branching |
| All-Defs | Medium | Usually feasible | Catches uninitialized/stale variable use |
| All-Uses | Medium-High | Usually feasible | Catches incorrect computations and predicates |
| All-DU-Paths | High | Sometimes infeasible | Catches subtle data flow faults |
| All-Paths | Exponential/Infinite | Usually infeasible | Theoretical maximum |

### Minimum Coverage Recommendations by Risk Level

| Risk Level | Minimum Criterion | Rationale |
|---|---|---|
| Low (utilities, helpers) | Statement coverage | Verify code is reachable |
| Medium (business logic) | Branch coverage | Verify all decision outcomes |
| High (financial, safety) | All-Uses | Verify data flows correctly through all paths |
| Critical (life-safety) | All-DU-Paths | Maximum practical data flow coverage |

---

## 7. Loop Testing Considerations

Loops require special attention because they create potentially infinite path counts.

### Loop Testing Heuristic (Beizer)

For a simple loop with maximum iteration count N:

1. Skip the loop entirely (0 iterations)
2. Execute exactly 1 iteration
3. Execute exactly 2 iterations
4. Execute m iterations where m < N (typical case)
5. Execute N-1 iterations
6. Execute N iterations (maximum)
7. Execute N+1 iterations (attempt to exceed maximum)

### Nested Loop Strategy

- Test inner loop with outer loop at minimum iteration count
- Test outer loop with inner loop at typical iteration count
- Test both at boundary iterations simultaneously for interaction faults

### Cyclomatic Complexity and Loops

A while loop contributes 1 to the decision count (and hence to V(G)), but the number of distinct paths through the loop is unbounded. This is why V(G) basis paths provide a practical minimum -- they ensure each decision outcome is tested, even though they cannot cover all possible iteration counts.

---

## 8. Worked Example: Full Path and Data Flow Analysis

### Code Under Test: Binary Search

```python
def binary_search(arr, target):
    low = 0                        # N1: DEF(low)
    high = len(arr) - 1            # N2: DEF(high), USE(arr)
    result = -1                    # N3: DEF(result)
    while low <= high:             # N4: USE(low), USE(high) [p-use]
        mid = (low + high) // 2    # N5: DEF(mid), USE(low), USE(high) [c-use]
        if arr[mid] == target:     # N6: USE(arr), USE(mid), USE(target) [p-use]
            result = mid           # N7: DEF(result), USE(mid) [c-use]
            break                  # N8: goto N10
        elif arr[mid] < target:    # N9: USE(arr), USE(mid), USE(target) [p-use]
            low = mid + 1          # N10: DEF(low), USE(mid) [c-use]
        else:
            high = mid - 1         # N11: DEF(high), USE(mid) [c-use]
    return result                  # N12: USE(result) [c-use]
```

### Program Graph

```
N1 -> N2 -> N3 -> N4
N4 -> N5 (low <= high true)
N4 -> N12 (low <= high false)
N5 -> N6
N6 -> N7 (arr[mid] == target true)
N6 -> N9 (arr[mid] == target false)
N7 -> N8 -> N12
N9 -> N10 (arr[mid] < target true)
N9 -> N11 (arr[mid] < target false)
N10 -> N4 (loop back)
N11 -> N4 (loop back)
```

### Cyclomatic Complexity

Edges: N1->N2, N2->N3, N3->N4, N4->N5, N4->N12, N5->N6, N6->N7, N6->N9, N7->N8, N8->N12, N9->N10, N9->N11, N10->N4, N11->N4

e = 14, n = 12, p = 1

V(G) = 14 - 12 + 2 = 4

Decision nodes: N4 (while), N6 (==), N9 (<): d = 3, V(G) = 3 + 1 = 4.

### Key DU-Pairs for `low`

| DEF | USE | Path |
|---|---|---|
| N1 | N4 (p-use) | N1->N2->N3->N4 |
| N1 | N5 (c-use) | N1->N2->N3->N4->N5 |
| N10 | N4 (p-use) | N10->N4 |
| N10 | N5 (c-use) | N10->N4->N5 |

### Test Cases for All-Uses Coverage of `low`

| Test | arr | target | Exercises |
|---|---|---|---|
| T1 | [] | 5 | DEF(low, N1) -> USE(low, N4): loop not entered |
| T2 | [5] | 5 | DEF(low, N1) -> USE(low, N5): found immediately |
| T3 | [1,3,5,7,9] | 7 | DEF(low, N10) -> USE(low, N4) -> USE(low, N5): search right half |
