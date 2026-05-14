# Graph Theory Essentials Reference

Definitions, key theorems, algorithms, and applications for discrete graph theory in computer science.

## 1. Basic Definitions

**Graph.** A graph G = (V, E) consists of a set V of vertices (nodes) and a set E of edges. In an undirected graph, each edge is an unordered pair {u, v}. In a directed graph (digraph), each edge is an ordered pair (u, v).

**Simple graph.** No self-loops (edges from a vertex to itself) and no parallel edges (multiple edges between the same pair of vertices).

**Subgraph.** H = (V', E') is a subgraph of G = (V, E) if V' is a subset of V and E' is a subset of E with endpoints in V'.

**Complement.** The complement of G = (V, E), written G_bar, has the same vertex set but edge {u,v} is in G_bar iff {u,v} is not in E.

**Weighted graph.** Each edge has an associated numerical weight w(e).

**Terminology table:**

| Term | Definition |
|---|---|
| Adjacent | Vertices u, v are adjacent if {u,v} is an edge |
| Incident | Edge e is incident to vertex v if v is an endpoint of e |
| Degree deg(v) | Number of edges incident to v (for digraphs: in-degree + out-degree) |
| Walk | Sequence of vertices v0, v1, ..., vk where each consecutive pair is connected by an edge |
| Path | Walk with no repeated vertices |
| Cycle | Walk where v0 = vk and no other vertices repeat (length >= 3) |
| Connected | Every pair of vertices has a path between them |
| Component | Maximal connected subgraph |
| Tree | Connected graph with no cycles |
| Forest | Acyclic graph (each component is a tree) |

---

## 2. Fundamental Theorems

### Handshaking Lemma

**Theorem.** In any graph G = (V, E):

```
sum_{v in V} deg(v) = 2|E|
```

**Proof.** Each edge {u, v} contributes exactly 1 to deg(u) and 1 to deg(v), thus contributes 2 to the total sum of degrees. QED.

**Corollary.** The number of vertices with odd degree is even.

**Proof.** Let O = {v : deg(v) is odd} and E_set = {v : deg(v) is even}. Then:
```
sum_{v in O} deg(v) = 2|E| - sum_{v in E_set} deg(v)
```
The right side is even (difference of even numbers). A sum of odd numbers is even iff there are an even number of them. So |O| is even. QED.

### Properties of Trees

**Theorem.** For a graph G with n vertices, any two of the following imply the third:
1. G is connected.
2. G is acyclic.
3. G has n - 1 edges.

**Corollary.** Every tree on n vertices has exactly n - 1 edges.

**Corollary.** Every connected graph on n vertices has at least n - 1 edges.

**Corollary.** Adding any edge to a tree creates exactly one cycle. Removing any edge from a tree disconnects it.

### Euler's Formula for Planar Graphs

**Theorem.** For any connected planar graph with V vertices, E edges, and F faces (including the outer unbounded face):

```
V - E + F = 2
```

**Proof (by induction on E).**

**Base case:** E = 0. Then V = 1 (connected graph with no edges is a single vertex) and F = 1 (just the outer face). V - E + F = 1 - 0 + 1 = 2. Verified.

**Inductive step:** Assume the formula holds for all connected planar graphs with fewer than E edges. Consider a connected planar graph G with E edges.

**Case 1: G has a cycle.** Remove an edge e from a cycle. The graph G - e is still connected (the cycle provides an alternate path). Removing e merges two faces into one, so F decreases by 1, E decreases by 1, and V stays the same. By IH: V - (E-1) + (F-1) = 2, so V - E + F = 2.

**Case 2: G has no cycle (G is a tree).** Then E = V - 1 and F = 1. V - (V-1) + 1 = 2. QED.

**Corollaries of Euler's Formula:**

1. **Edge bound:** For a simple connected planar graph with V >= 3: E <= 3V - 6.

   *Proof:* Every face is bounded by at least 3 edges, and each edge borders at most 2 faces: 3F <= 2E, so F <= 2E/3. Substituting into Euler's formula: 2 = V - E + F <= V - E + 2E/3 = V - E/3. So E <= 3V - 6. QED.

2. **Triangle-free bound:** For a simple connected planar graph with no triangles and V >= 3: E <= 2V - 4.

   *Proof:* Each face is bounded by at least 4 edges: 4F <= 2E, so F <= E/2. Then 2 = V - E + F <= V - E/2, giving E <= 2V - 4. QED.

3. **Non-planarity of K5:** K5 has V = 5, E = 10. But 3(5) - 6 = 9 < 10, so K5 is not planar.

4. **Non-planarity of K3,3:** K3,3 has V = 6, E = 9, and is triangle-free. But 2(6) - 4 = 8 < 9, so K3,3 is not planar.

### Kuratowski's Theorem

**Theorem.** A graph is planar if and only if it contains no subgraph that is a subdivision of K5 or K3,3.

A **subdivision** of a graph is obtained by replacing edges with paths (inserting degree-2 vertices along edges).

This gives a complete characterization of planarity. The equivalent Wagner's theorem states: G is planar iff it has no K5 or K3,3 as a minor.

---

## 3. Graph Algorithms

### Breadth-First Search (BFS)

Explores vertices level by level from a source vertex s. Uses a queue.

```
function BFS(G, s):
    for each vertex u in V:
        dist[u] = infinity
        parent[u] = null
    dist[s] = 0
    Q = empty queue
    enqueue(Q, s)
    while Q is not empty:
        u = dequeue(Q)
        for each neighbor v of u:
            if dist[v] == infinity:
                dist[v] = dist[u] + 1
                parent[v] = u
                enqueue(Q, v)
```

**Properties:**
- Time: O(V + E) with adjacency list.
- Finds shortest paths (fewest edges) from s to all reachable vertices.
- BFS tree has the property that every non-tree edge connects vertices at the same level or adjacent levels.
- Can detect bipartiteness: a graph is bipartite iff BFS finds no edge between vertices at the same level.

### Depth-First Search (DFS)

Explores as deep as possible before backtracking. Uses a stack (or recursion).

```
function DFS(G):
    for each vertex u in V:
        color[u] = WHITE
        parent[u] = null
    time = 0
    for each vertex u in V:
        if color[u] == WHITE:
            DFS-Visit(G, u)

function DFS-Visit(G, u):
    time = time + 1
    discover[u] = time
    color[u] = GRAY
    for each neighbor v of u:
        if color[v] == WHITE:
            parent[v] = u
            DFS-Visit(G, v)
    color[u] = BLACK
    time = time + 1
    finish[u] = time
```

**Properties:**
- Time: O(V + E).
- Classifies edges into: tree edges, back edges (to an ancestor), forward edges (to a descendant), cross edges (neither).
- A directed graph has a cycle iff DFS finds a back edge.
- An undirected graph has a cycle iff DFS finds a back edge (an edge to an already-discovered vertex that is not the parent).

### Topological Sort

A linear ordering of vertices of a DAG such that for every directed edge (u, v), u appears before v.

**Algorithm 1 (DFS-based):** Run DFS on the entire graph. Output vertices in reverse order of finish times.

**Algorithm 2 (Kahn's algorithm):**
```
function TopologicalSort(G):
    compute in-degree of every vertex
    Q = queue of all vertices with in-degree 0
    L = empty list
    while Q is not empty:
        u = dequeue(Q)
        append u to L
        for each neighbor v of u:
            in-degree[v] = in-degree[v] - 1
            if in-degree[v] == 0:
                enqueue(Q, v)
    if |L| != |V|:
        error "graph has a cycle"
    return L
```

**Time:** O(V + E). A topological sort exists iff the graph is a DAG.

### Shortest Paths

| Algorithm | Graph Type | Time Complexity | Notes |
|---|---|---|---|
| BFS | Unweighted | O(V + E) | Finds shortest by edge count |
| Dijkstra | Non-negative weights | O((V + E) log V) with binary heap | Greedy, does not work with negative edges |
| Bellman-Ford | General weights | O(VE) | Detects negative cycles |
| Floyd-Warshall | All pairs | O(V^3) | Dynamic programming on intermediate vertices |

### Minimum Spanning Tree

| Algorithm | Approach | Time Complexity |
|---|---|---|
| Kruskal | Sort edges, add lightest edge that doesn't create a cycle | O(E log E) |
| Prim | Grow tree from a vertex, always add lightest edge crossing the cut | O((V + E) log V) with binary heap |

**Cut property:** For any cut of the graph, the minimum weight edge crossing the cut is in every MST (assuming unique edge weights).

---

## 4. Bipartite Graphs and Matchings

### Bipartite Graphs

**Definition.** G = (V, E) is bipartite if V can be partitioned into two sets L and R such that every edge has one endpoint in L and one in R.

**Characterization.** G is bipartite if and only if it contains no odd-length cycle.

**Detection.** Run BFS/DFS and attempt to 2-color the graph. If a contradiction is found (an edge between two vertices of the same color), the graph is not bipartite.

### Matchings

**Definition.** A matching M is a set of edges with no shared vertices. A matching is:
- **Maximal** if no edge can be added to it.
- **Maximum** if it has the largest possible number of edges.
- **Perfect** if every vertex is matched (|M| = |V|/2).

### Hall's Marriage Theorem

**Theorem.** A bipartite graph G = (L union R, E) has a matching that saturates all of L (a complete matching from L to R) if and only if for every subset S of L:

```
|N(S)| >= |S|
```

where N(S) is the set of vertices in R adjacent to at least one vertex in S.

**Proof sketch (necessity).** If there is a complete matching, then each vertex in S is matched to a distinct vertex in N(S), so |N(S)| >= |S|.

**Proof sketch (sufficiency).** By contradiction. Suppose Hall's condition holds but the maximum matching M does not saturate L. Let u be an unmatched vertex in L. Consider alternating paths from u (paths that alternate between unmatched and matched edges). If an augmenting path exists (ending at an unmatched vertex in R), we can increase the matching size, contradicting maximality. If no augmenting path exists, the set of vertices reachable by alternating paths from u violates Hall's condition.

### Algorithms for Maximum Matching

- **Augmenting path algorithm** (bipartite): O(VE). Find augmenting paths via BFS.
- **Hopcroft-Karp** (bipartite): O(E * sqrt(V)).
- **Edmonds' blossom algorithm** (general graphs): O(V^3).

---

## 5. Graph Coloring

**Definition.** A proper k-coloring assigns one of k colors to each vertex so that no two adjacent vertices share the same color. The chromatic number chi(G) is the minimum k.

**Key Results:**

| Result | Statement |
|---|---|
| Greedy bound | chi(G) <= Delta(G) + 1, where Delta(G) is the maximum degree |
| Brooks' theorem | chi(G) <= Delta(G), unless G is a complete graph or odd cycle |
| Bipartite characterization | chi(G) <= 2 iff G is bipartite (no odd cycles) |
| Four color theorem | Every planar graph satisfies chi(G) <= 4 |
| Five color theorem | Every planar graph satisfies chi(G) <= 5 (easier to prove than four color) |
| Clique lower bound | chi(G) >= omega(G), where omega(G) is the clique number |

**Greedy Coloring Algorithm:**
```
function GreedyColor(G, ordering v1, v2, ..., vn):
    for i = 1 to n:
        color[vi] = smallest positive integer not used by any neighbor of vi that has already been colored
    return color
```

The greedy algorithm always uses at most Delta(G) + 1 colors, but the number used depends on the vertex ordering. Optimal orderings can yield chi(G) colors, but finding such an ordering is NP-hard in general.

**Chromatic number for special graphs:**
- Complete graph K_n: chi = n
- Cycle C_n (n >= 3): chi = 2 if n even, chi = 3 if n odd
- Tree: chi = 1 if one vertex, chi = 2 otherwise
- Bipartite graph (with edges): chi = 2
- Petersen graph: chi = 3
- Planar graph: chi <= 4 (Four Color Theorem)

---

## 6. Connectivity

### Vertex and Edge Connectivity

**Vertex connectivity kappa(G):** The minimum number of vertices whose removal disconnects G (or reduces it to a single vertex). A graph is k-connected if kappa(G) >= k.

**Edge connectivity lambda(G):** The minimum number of edges whose removal disconnects G.

**Whitney's inequality:** kappa(G) <= lambda(G) <= delta(G), where delta(G) is the minimum degree.

**Menger's Theorem (vertex version).** The maximum number of vertex-disjoint paths between two non-adjacent vertices u and v equals the minimum number of vertices (other than u, v) whose removal separates u from v.

**Menger's Theorem (edge version).** The maximum number of edge-disjoint paths between vertices u and v equals the minimum number of edges whose removal separates u from v.

### Articulation Points and Bridges

**Articulation point (cut vertex):** A vertex whose removal increases the number of connected components.

**Bridge:** An edge whose removal increases the number of connected components.

**Detection via DFS:** Both can be found in O(V + E) time using DFS discovery and low-link values. A vertex u is an articulation point if:
- u is the root of the DFS tree and has two or more children, OR
- u is not the root and has a child v with no back edge from the subtree of v reaching an ancestor of u (i.e., low[v] >= discover[u]).

An edge (u, v) is a bridge if low[v] > discover[u].

---

## 7. Euler and Hamilton Paths

### Euler Circuits and Paths

**Euler circuit:** A closed walk that visits every edge exactly once.

**Euler path:** An open walk that visits every edge exactly once.

**Existence conditions (undirected):**

| Type | Condition |
|---|---|
| Euler circuit | G is connected and every vertex has even degree |
| Euler path (not circuit) | G is connected and exactly two vertices have odd degree |

**Existence conditions (directed):**

| Type | Condition |
|---|---|
| Euler circuit | G is strongly connected and in-degree = out-degree for all vertices |
| Euler path | G is connected (weakly) and at most one vertex has out-degree - in-degree = 1, at most one has in-degree - out-degree = 1, all others have equal in/out-degree |

**Finding an Euler circuit (Hierholzer's algorithm):**
1. Start at any vertex. Follow edges (removing them as you go) until returning to the start, forming a circuit C.
2. While C contains a vertex v with remaining edges: start a new circuit from v, then splice it into C.
3. Time: O(E).

### Hamilton Cycles

**Hamilton cycle:** A cycle that visits every vertex exactly once.

**Hamilton path:** A path that visits every vertex exactly once.

**Key facts:**
- Determining whether a Hamilton cycle exists is NP-complete.
- **Dirac's theorem:** If n >= 3 and every vertex has degree >= n/2, then G has a Hamilton cycle.
- **Ore's theorem:** If n >= 3 and for every pair of non-adjacent vertices u, v: deg(u) + deg(v) >= n, then G has a Hamilton cycle.

---

## 8. Network Flow

### Max-Flow Min-Cut Theorem

**Setting.** A flow network is a directed graph with:
- A source vertex s and a sink vertex t.
- Each edge (u, v) has a non-negative capacity c(u, v).

**Flow.** A function f: E -> R satisfying:
1. Capacity constraint: 0 <= f(u, v) <= c(u, v) for all edges.
2. Conservation: for all vertices u except s and t, flow in = flow out.

**Value of flow:** |f| = total flow out of s = total flow into t.

**Cut.** A partition (S, T) of V with s in S and t in T. The capacity of the cut is sum of c(u, v) for all edges from S to T.

**Theorem (Max-Flow Min-Cut).** The maximum value of a flow equals the minimum capacity of a cut separating s and t.

### Ford-Fulkerson Algorithm

```
function FordFulkerson(G, s, t):
    initialize f(u, v) = 0 for all edges
    while there exists an augmenting path P from s to t in the residual graph:
        bottleneck = min capacity along P in residual graph
        for each edge (u, v) in P:
            if (u, v) is a forward edge:
                f(u, v) = f(u, v) + bottleneck
            else:  // (u, v) is a backward edge
                f(v, u) = f(v, u) - bottleneck
    return f
```

**Residual graph:** For each edge (u, v) with capacity c and flow f:
- Forward edge (u, v) with residual capacity c - f (if c - f > 0).
- Backward edge (v, u) with residual capacity f (if f > 0).

**Edmonds-Karp variant:** Choose augmenting paths via BFS (shortest in terms of edges). Time: O(V * E^2).

---

## 9. Adjacency Matrix Properties

For a graph G with adjacency matrix A (A[i][j] = 1 if edge exists, 0 otherwise):

**Walks.** (A^k)[i][j] counts the number of walks of length k from vertex i to vertex j.

**Eigenvalues.** For a d-regular graph, d is the largest eigenvalue of A. The number of connected components equals the multiplicity of eigenvalue d. The second-largest eigenvalue governs expansion properties.

**Spectral gap.** A large gap between the first and second eigenvalues indicates a good expander graph, useful in derandomization and network design.

---

## 10. Quick Reference: Graph Properties

| Property | How to Check | Time |
|---|---|---|
| Connected | BFS/DFS from any vertex | O(V + E) |
| Bipartite | BFS 2-coloring | O(V + E) |
| Acyclic (undirected) | DFS, check for back edges | O(V + E) |
| Acyclic (directed, DAG) | Topological sort succeeds | O(V + E) |
| Planar | Boyer-Myrvold or LR-planarity algorithm | O(V) |
| Eulerian | Check degree conditions | O(V + E) |
| Hamiltonian | NP-complete in general | Exponential worst case |
| k-colorable (k >= 3) | NP-complete in general | Exponential worst case |
| Strongly connected (directed) | Two DFS passes or Tarjan's algorithm | O(V + E) |
| Bridges/articulation points | DFS with low-link values | O(V + E) |
| Maximum matching (bipartite) | Hopcroft-Karp | O(E * sqrt(V)) |
| Maximum flow | Edmonds-Karp | O(V * E^2) |
| MST | Kruskal or Prim | O(E log V) |
