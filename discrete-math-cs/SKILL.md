---
name: discrete-math-cs
description: "Discrete mathematics foundations for computer science: proof techniques, induction, number theory, RSA cryptography, graph theory, combinatorics, and probability."
origin: ECC
---

# Discrete Mathematics for Computer Science

Based on **Mathematics for Computer Science** by Eric Lehman, F. Thomson Leighton, and Albert R. Meyer (MIT, 2018).

## When to Activate

Use this skill when the task involves:

- **Proving algorithm correctness** -- loop invariants, structural induction on recursive data, termination arguments
- **Analyzing computational complexity** -- counting operations, summation identities, asymptotic bounds
- **Graph problems** -- shortest paths, connectivity, matching, coloring, network flow, planarity
- **Cryptography questions** -- modular arithmetic, RSA key generation, Euler's theorem applications
- **Probability and counting problems** -- combinatorial arguments, expectation calculations, tail bounds
- **Formal logic** -- propositional and predicate logic, satisfiability, proof construction

## Topics Index

### Proofs & Logic

**Propositional Logic.** Propositions are statements that are either true or false. Logical connectives: AND (conjunction, `p AND q`), OR (disjunction, `p OR q`), NOT (negation, `NOT p`), IMPLIES (implication, `p IMPLIES q`), IFF (biconditional, `p IFF q`). An implication `p IMPLIES q` is false only when `p` is true and `q` is false. The contrapositive `NOT q IMPLIES NOT p` is logically equivalent to the original implication.

**Truth Tables and Normal Forms.** Every propositional formula can be converted to Conjunctive Normal Form (CNF) -- a conjunction of disjunctions -- or Disjunctive Normal Form (DNF) -- a disjunction of conjunctions. SAT (Boolean satisfiability) asks whether a CNF formula has a satisfying assignment; it is NP-complete.

**Predicate Logic.** Extends propositional logic with variables, predicates, and quantifiers. Universal quantifier: `FOR ALL x. P(x)` asserts P holds for every x in the domain. Existential quantifier: `EXISTS x. P(x)` asserts P holds for at least one x. De Morgan's laws for quantifiers: `NOT (FOR ALL x. P(x))` is equivalent to `EXISTS x. NOT P(x)`.

**Proof Techniques.**

| Method | When to Use | Structure |
|---|---|---|
| Direct proof | Goal follows naturally from definitions and axioms | Assume P, derive Q |
| Proof by contradiction | Direct approach is unclear; assume negation leads to impossibility | Assume NOT Q, derive a contradiction |
| Proof by contrapositive | Easier to prove `NOT Q IMPLIES NOT P` than `P IMPLIES Q` | Assume NOT Q, derive NOT P |
| Proof by cases | Statement naturally splits into exhaustive sub-cases | Cover all cases, prove each |
| Well ordering principle | Proving no counterexample exists among natural numbers | Assume a non-empty set of counterexamples, derive contradiction from the smallest element |

See `references/proof-techniques.md` for full templates and worked examples.

### Induction & Recursion

**Ordinary Induction.** To prove `FOR ALL n >= b. P(n)`: (1) Base case -- prove P(b). (2) Inductive step -- assume P(n) (inductive hypothesis), prove P(n+1). Validity rests on the well-ordering principle for natural numbers.

**Strong Induction.** The inductive hypothesis assumes P(b), P(b+1), ..., P(n) all hold, then proves P(n+1). Equivalent in power to ordinary induction but often more convenient when the proof for n+1 depends on values earlier than n.

**Structural Induction.** Proves properties of recursively defined data structures (trees, lists, formulas). Base case covers atomic/leaf elements; inductive step assumes the property holds for sub-structures and proves it for the composed structure.

**State Machines and Invariants.** Model computation as a state machine with states S, start state q0, and transition relation. The **Invariant Principle**: if a property P holds for q0 and is preserved by every transition, then P holds for every reachable state. Applications: partial correctness (if the machine halts, the output is correct), termination (showing a non-negative integer decreasing function exists), the Stable Marriage algorithm (Gale-Shapley).

### Mathematical Data Types

**Sets.** A set is an unordered collection of distinct elements. Operations: union (A union B), intersection (A intersect B), difference (A - B), complement, symmetric difference. Power set P(A) has 2^|A| elements. Russell's paradox: the set of all sets that do not contain themselves leads to contradiction, motivating axiomatic set theory.

**Sequences and Functions.** A sequence is an ordered list (may have repeats). A function f: A -> B maps each element of domain A to exactly one element of codomain B. Injection (one-to-one): f(a1) = f(a2) implies a1 = a2. Surjection (onto): every b in B has a preimage. Bijection: both injective and surjective -- establishes that |A| = |B|.

**Binary Relations.** A relation R on A is a subset of A x A. Properties: reflexive (a R a for all a), symmetric (a R b implies b R a), antisymmetric (a R b and b R a implies a = b), transitive (a R b and b R c implies a R c). An equivalence relation is reflexive, symmetric, and transitive -- partitions A into equivalence classes. A partial order is reflexive, antisymmetric, and transitive.

**Infinite Sets.** A set is countably infinite if there exists a bijection with the natural numbers. The rationals are countable (diagonal argument). The reals are uncountable (Cantor's diagonal argument). The halting problem: no program can decide whether an arbitrary program halts -- proved by diagonalization.

### Number Theory

**Divisibility.** Integer a divides b (written a | b) if b = ka for some integer k. Properties: if a | b and a | c then a | (sb + tc) for any integers s, t.

**GCD and the Euclidean Algorithm.** gcd(a, b) is the largest positive integer dividing both a and b. Euclidean algorithm: gcd(a, b) = gcd(b, a mod b), with base case gcd(a, 0) = a. Runs in O(log(min(a,b))) steps. **Bezout's identity**: gcd(a, b) = sa + tb for some integers s, t, computable via the Extended Euclidean Algorithm.

**Primes.** A prime p > 1 has no positive divisors other than 1 and p. **Fundamental Theorem of Arithmetic**: every integer > 1 has a unique prime factorization (up to ordering). There are infinitely many primes (Euclid's proof by contradiction). The Prime Number Theorem: pi(n) ~ n / ln(n). Sieve of Eratosthenes finds all primes up to n in O(n log log n).

**Modular Arithmetic.** a is congruent to b modulo m (written a = b (mod m)) iff m | (a - b). Arithmetic operations (+, -, *) are compatible with congruence. Multiplicative inverse of a modulo m exists iff gcd(a, m) = 1, and can be found via the Extended Euclidean Algorithm.

**Euler's Theorem and Fermat's Little Theorem.** Euler's totient phi(n) counts integers in {1, ..., n} coprime to n. For prime p: phi(p) = p - 1. For prime powers: phi(p^k) = p^k - p^(k-1). Multiplicative: phi(mn) = phi(m)phi(n) when gcd(m,n) = 1. **Euler's theorem**: if gcd(a, n) = 1, then a^phi(n) = 1 (mod n). **Fermat's little theorem** (special case): a^(p-1) = 1 (mod p) for prime p when p does not divide a.

**RSA Cryptosystem.** Key generation: choose large primes p, q; compute n = pq, phi(n) = (p-1)(q-1); choose e with gcd(e, phi(n)) = 1; compute d = e^(-1) mod phi(n). Public key: (n, e). Private key: d. Encrypt: c = m^e mod n. Decrypt: m = c^d mod n. Correctness: c^d = m^(ed) = m^(1 + k*phi(n)) = m * (m^phi(n))^k = m * 1^k = m (mod n), by Euler's theorem.

See `references/number-theory-rsa.md` for detailed derivations and a worked RSA example.

### Graph Theory

**Directed Graphs.** A digraph G = (V, E) where E is a set of ordered pairs. A **DAG** (directed acyclic graph) has no directed cycles. Every finite DAG has a topological sort -- a linear ordering of vertices such that for every edge (u, v), u appears before v. Computed via repeated removal of vertices with in-degree 0 or via DFS finish times.

**Partial Orders.** A partial order on a set is a reflexive, antisymmetric, transitive relation. Represented by Hasse diagrams (transitive reduction of the "less than" relation). A lattice is a partial order where every pair of elements has a unique least upper bound (join) and greatest lower bound (meet).

**Adjacency Representation.** Adjacency matrix A: A[i][j] = 1 if edge (i,j) exists. For undirected graphs, A is symmetric. A^k[i][j] counts the number of walks of length k from i to j.

**Simple Graphs.** No self-loops or parallel edges. Degree deg(v) is the number of edges incident to v. **Handshaking Lemma**: sum of all degrees = 2|E|, so the number of odd-degree vertices is even. Two graphs are isomorphic if there is a bijection between their vertex sets preserving adjacency.

**Bipartite Graphs and Matchings.** A graph is bipartite iff it contains no odd-length cycle, equivalently iff it is 2-colorable. A matching is a set of edges with no shared vertices. **Hall's theorem**: a bipartite graph G = (L union R, E) has a matching saturating all of L iff for every subset S of L, |N(S)| >= |S|, where N(S) is the set of neighbors of S.

**Graph Coloring.** A proper k-coloring assigns one of k colors to each vertex so no two adjacent vertices share a color. The chromatic number chi(G) is the minimum k for which a proper k-coloring exists. For any graph: chi(G) <= Delta(G) + 1 (greedy bound). Brooks' theorem: chi(G) <= Delta(G) unless G is a complete graph or an odd cycle.

**Connectivity.** A graph is connected if every pair of vertices has a path between them. A cut vertex (articulation point) is a vertex whose removal disconnects the graph. A bridge is an edge whose removal disconnects the graph. k-connected: remains connected after removing any k-1 vertices.

**Euler and Hamilton Paths.** An Euler circuit visits every edge exactly once and returns to the start. Exists iff the graph is connected and every vertex has even degree. An Euler path (not circuit) exists iff exactly two vertices have odd degree. A Hamilton cycle visits every vertex exactly once -- determining existence is NP-complete.

**Planar Graphs.** A graph is planar if it can be drawn in the plane without edge crossings. **Euler's formula**: for a connected planar graph, V - E + F = 2, where F is the number of faces. Corollary: E <= 3V - 6 for simple planar graphs (V >= 3). **Kuratowski's theorem**: a graph is planar iff it contains no subdivision of K5 or K3,3.

**Communication Networks.** Routing: finding paths in a network. **Min-cut Max-flow theorem**: the maximum flow from source to sink equals the minimum capacity of a cut separating them. Ford-Fulkerson algorithm finds max flow by iteratively augmenting along paths with available capacity.

See `references/graph-theory-essentials.md` for definitions, algorithms, and key theorem proofs.

### Counting & Combinatorics

**Fundamental Counting Rules.** Sum rule: if task can be done in one of n1 or n2 ways (disjoint), total = n1 + n2. Product rule: if task involves two independent stages with n1 and n2 choices, total = n1 * n2.

**Bijection Principle.** If there is a bijection between sets A and B, then |A| = |B|. Useful when counting A directly is hard but B is easier.

**Permutations and Combinations.** P(n, k) = n! / (n-k)! counts ordered selections of k from n. C(n, k) = n! / (k!(n-k)!) counts unordered selections. Key identities: C(n, k) = C(n, n-k), C(n, k) = C(n-1, k-1) + C(n-1, k) (Pascal's identity). Multinomial coefficient: n! / (k1! k2! ... kr!) counts arrangements of n objects with ki of type i.

**Pigeonhole Principle.** If n+1 objects are placed into n boxes, at least one box contains two or more objects. Generalized: if kn+1 objects go into n boxes, some box has at least k+1. Applications: guaranteed collisions in hashing, repeated remainders, Ramsey-type results.

**Inclusion-Exclusion Principle.** |A1 union A2 union ... union An| = sum|Ai| - sum|Ai intersect Aj| + sum|Ai intersect Aj intersect Ak| - ... + (-1)^(n+1)|A1 intersect ... intersect An|. Classic application: counting derangements D(n) = n! * sum_{k=0}^{n} (-1)^k / k!, which approaches n!/e.

**Generating Functions.** The ordinary generating function for sequence a0, a1, a2, ... is G(x) = sum_{n>=0} a_n x^n. Useful for solving recurrences and counting problems. Example: the generating function for Fibonacci numbers satisfies G(x) = x / (1 - x - x^2).

**Binomial Theorem.** (x + y)^n = sum_{k=0}^{n} C(n,k) x^(n-k) y^k. Special cases: (1+1)^n = 2^n (sum of all binomial coefficients), (1-1)^n = 0 (alternating sum is zero).

See `references/counting-probability.md` for identities, examples, and distribution tables.

### Probability

**Sample Spaces and Events.** A sample space S is the set of all possible outcomes. An event A is a subset of S. For finite uniform sample spaces: Pr[A] = |A| / |S|.

**Conditional Probability.** Pr[A | B] = Pr[A intersect B] / Pr[B]. Product rule: Pr[A intersect B] = Pr[A | B] * Pr[B]. **Bayes' theorem**: Pr[A | B] = Pr[B | A] * Pr[A] / Pr[B]. Law of total probability: Pr[B] = sum_i Pr[B | Ai] * Pr[Ai] for a partition {Ai}.

**Independence.** Events A and B are independent iff Pr[A intersect B] = Pr[A] * Pr[B], equivalently Pr[A | B] = Pr[A]. Mutual independence of n events requires the product rule for every subset.

**Random Variables.** A random variable X is a function from the sample space to the reals. Key distributions:

| Distribution | PMF/PDF | E[X] | Var(X) |
|---|---|---|---|
| Bernoulli(p) | Pr[X=1] = p | p | p(1-p) |
| Binomial(n,p) | C(n,k)p^k(1-p)^(n-k) | np | np(1-p) |
| Geometric(p) | (1-p)^(k-1) p | 1/p | (1-p)/p^2 |
| Poisson(lambda) | e^(-lambda) lambda^k / k! | lambda | lambda |

**Expectation.** E[X] = sum_x x * Pr[X = x]. **Linearity of expectation**: E[X + Y] = E[X] + E[Y], regardless of dependence. This is the single most useful property in discrete probability.

**Variance.** Var(X) = E[(X - E[X])^2] = E[X^2] - (E[X])^2. For independent X, Y: Var(X + Y) = Var(X) + Var(Y). Standard deviation: sigma = sqrt(Var(X)).

**Tail Bounds.**

| Bound | Statement | When to Use |
|---|---|---|
| Markov | Pr[X >= a] <= E[X] / a (X >= 0) | Only know the mean; weakest bound |
| Chebyshev | Pr[|X - mu| >= k*sigma] <= 1/k^2 | Know mean and variance |
| Chernoff | Pr[X >= (1+delta)mu] <= (e^delta / (1+delta)^(1+delta))^mu | Sum of independent bounded RVs; tightest |

**Random Walks.** A particle moves +1 or -1 each step with probabilities p and 1-p. For the symmetric walk (p = 1/2) on {0, ..., n}: expected time to reach boundary from position k is k(n-k). Gambler's ruin probability: with initial fortune k, probability of reaching n before 0 is k/n (symmetric case).

## Key Concepts

### Proof Method Selection Guide

1. **Can you go directly from hypothesis to conclusion?** Use direct proof.
2. **Is the conclusion naturally split into cases?** Use proof by cases.
3. **Is the negation of the conclusion easier to work with?** Use proof by contrapositive.
4. **Does assuming the negation lead to an obvious impossibility?** Use proof by contradiction.
5. **Is the statement parameterized by a natural number?** Consider induction (ordinary or strong).
6. **Is the statement about a recursively defined structure?** Use structural induction.
7. **Are you showing no counterexample exists among integers?** Consider the well-ordering principle.

### RSA Algorithm Steps

1. Select two large distinct primes p and q (typically 1024+ bits each).
2. Compute n = p * q.
3. Compute phi(n) = (p - 1)(q - 1).
4. Choose public exponent e such that 1 < e < phi(n) and gcd(e, phi(n)) = 1. Common choice: e = 65537.
5. Compute private exponent d such that d * e = 1 (mod phi(n)) using the Extended Euclidean Algorithm.
6. Public key: (n, e). Private key: d.
7. Encrypt message m (where 0 <= m < n): c = m^e mod n.
8. Decrypt ciphertext c: m = c^d mod n.

### Graph Theory Key Theorems

- **Handshaking Lemma**: sum of degrees = 2|E|
- **Euler's Formula** (planar graphs): V - E + F = 2
- **Hall's Marriage Theorem**: complete matching in bipartite graph iff |N(S)| >= |S| for all S
- **Kuratowski's Theorem**: G is planar iff it has no K5 or K3,3 subdivision
- **Min-Cut Max-Flow**: max flow = min cut capacity
- **Brooks' Theorem**: chi(G) <= Delta(G) for connected graphs that are not complete graphs or odd cycles

## Problem-Solving Patterns

| Pattern | Applicable When | Approach |
|---|---|---|
| Invariant argument | Proving a property holds throughout a process | Define invariant, verify at start, show each step preserves it |
| Double counting | Two ways to count the same quantity | Count a set in two different ways, equate results |
| Bijective proof | Showing two sets have equal size | Construct explicit bijection |
| Pigeonhole | Proving existence of collision/repeat | Identify objects and boxes, apply pigeonhole |
| Linearity of expectation | Computing expected value of a complex sum | Decompose into indicator variables, sum expectations |
| Indicator variables | Counting elements satisfying a property | Define Xi = 1 if element i qualifies, compute E[sum Xi] |
| Generating functions | Solving recurrences or counting with constraints | Encode sequence as power series, manipulate algebraically |
| Induction + invariant | Proving algorithm correctness | Induction on iterations with loop invariant as IH |

## Common Pitfalls

| Pitfall | Description | How to Avoid |
|---|---|---|
| Assuming the conclusion | Using what you want to prove as part of your argument | Clearly separate assumptions from what is being derived |
| Wrong induction base | Starting induction at the wrong value | Verify the base case matches the claim's domain |
| Forgetting strong vs weak | Using P(n) when you need P(k) for all k <= n | If the recursive step needs earlier values, use strong induction |
| Off-by-one in counting | Miscounting endpoints or boundary elements | Check small cases manually; use bijection to a known set |
| Confusing independence and disjointness | Disjoint events are not independent (unless one has probability 0) | Apply definitions precisely: independent means Pr[A and B] = Pr[A]*Pr[B] |
| Misapplying Bayes' theorem | Confusing Pr[A given B] with Pr[B given A] | Write out the full formula; identify the prior and the likelihood |
| Modular arithmetic errors | Applying division without checking invertibility | Verify gcd(a, m) = 1 before computing modular inverse |
| Planar graph edge bound | Applying E <= 3V - 6 to multigraphs or disconnected graphs | Check that the graph is simple and connected before applying |

## Source Material

**Mathematics for Computer Science**, Eric Lehman, F. Thomson Leighton, and Albert R. Meyer. MIT OpenCourseWare, 2018. Available at: [MIT OCW 6.042J](https://ocw.mit.edu/courses/6-042j-mathematics-for-computer-science-spring-2015/).
