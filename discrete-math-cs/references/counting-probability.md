# Counting and Probability Reference

Combinatorial identities, inclusion-exclusion, generating functions, probability distributions, expectation, variance, and tail bounds.

## 1. Fundamental Counting Principles

### Sum Rule (Rule of Addition)

If a task can be performed in one of n1 ways OR one of n2 ways, and these sets of ways are disjoint, then the total number of ways is n1 + n2.

**Generalized:** For pairwise disjoint sets A1, A2, ..., Ak:

```
|A1 union A2 union ... union Ak| = |A1| + |A2| + ... + |Ak|
```

### Product Rule (Rule of Multiplication)

If a task consists of two independent stages, the first with n1 choices and the second with n2 choices, the total number of ways is n1 * n2.

**Example:** License plates with 3 letters followed by 4 digits: 26^3 * 10^4 = 175,760,000.

### Bijection Principle

If there is a bijection between sets A and B, then |A| = |B|. Useful when counting A is hard but counting B is easy.

**Example:** The number of subsets of {1, 2, ..., n} equals the number of binary strings of length n (each element is either included or not), giving 2^n.

### Complement Counting

|A| = |U| - |A_bar|, where U is the universal set. Sometimes it is easier to count what you do NOT want and subtract.

**Example:** How many 4-digit numbers (1000-9999) have at least one repeated digit? Total 4-digit numbers: 9000. Numbers with all distinct digits: 9 * 9 * 8 * 7 = 4536. Answer: 9000 - 4536 = 4464.

---

## 2. Permutations and Combinations

### Permutations (Ordered Selections)

The number of ways to choose k items from n distinct items, where order matters:

```
P(n, k) = n! / (n - k)! = n * (n-1) * (n-2) * ... * (n-k+1)
```

**Special case:** P(n, n) = n! (the number of permutations of n objects).

### Combinations (Unordered Selections)

The number of ways to choose k items from n distinct items, where order does not matter:

```
C(n, k) = n! / (k! * (n - k)!)
```

Also written as "n choose k" or (n k).

### Key Identities

| Identity | Formula | Proof Idea |
|---|---|---|
| Symmetry | C(n, k) = C(n, n-k) | Choosing k to include = choosing n-k to exclude |
| Pascal's identity | C(n, k) = C(n-1, k-1) + C(n-1, k) | Element n is either in the subset or not |
| Vandermonde's identity | C(m+n, r) = sum_{k=0}^{r} C(m,k)*C(n,r-k) | Choose k from first m, r-k from last n |
| Hockey stick | C(r,r) + C(r+1,r) + ... + C(n,r) = C(n+1,r+1) | Induction or committee-chair argument |
| Sum of row | sum_{k=0}^{n} C(n,k) = 2^n | Count all subsets |
| Alternating sum | sum_{k=0}^{n} (-1)^k C(n,k) = 0 | Set x = -1 in binomial theorem |

### Binomial Theorem

```
(x + y)^n = sum_{k=0}^{n} C(n, k) * x^{n-k} * y^k
```

**Special cases:**
- (1 + 1)^n = 2^n: sum of all binomial coefficients.
- (1 - 1)^n = 0: alternating sum is zero (for n >= 1).
- (1 + x)^n = sum C(n,k) x^k: generating function for binomial coefficients.

### Multinomial Coefficient

The number of ways to arrange n objects where there are k1 of type 1, k2 of type 2, ..., kr of type r (and k1 + k2 + ... + kr = n):

```
n! / (k1! * k2! * ... * kr!)
```

**Example:** The number of anagrams of "MISSISSIPPI" = 11! / (1! * 4! * 4! * 2!) = 34,650.

### Stars and Bars

The number of ways to place n indistinguishable objects into k distinguishable bins:

```
C(n + k - 1, k - 1)
```

**Example:** Solutions to x1 + x2 + x3 = 10 with xi >= 0: C(12, 2) = 66.

---

## 3. Pigeonhole Principle

### Basic Form

If n + 1 objects are placed into n boxes, at least one box contains at least 2 objects.

### Generalized Form

If kn + 1 objects are placed into n boxes, at least one box contains at least k + 1 objects.

More generally: if N objects go into n boxes, some box contains at least ceil(N/n) objects.

### Worked Examples

**Example 1: Birthdays.** In a group of 367 people, at least two share a birthday (since there are at most 366 possible birthdays).

**Example 2: Socks.** A drawer contains 10 black and 10 white socks. How many must you draw (in the dark) to guarantee a matching pair? Answer: 3. The "boxes" are the 2 colors; with 3 socks, one color appears at least twice.

**Example 3: Subsequences.** In any sequence of n^2 + 1 distinct real numbers, there exists a monotonically increasing subsequence of length n + 1 or a monotonically decreasing subsequence of length n + 1 (Erdos-Szekeres theorem).

**Proof sketch:** Assign to each element ai the pair (li, di) where li is the length of the longest increasing subsequence ending at ai and di is the length of the longest decreasing subsequence ending at ai. If neither exceeds n, then all pairs are in {1,...,n} x {1,...,n}, giving at most n^2 distinct pairs. But the n^2 + 1 elements must have distinct pairs (if i < j and ai < aj, then lj > li; if ai > aj, then dj > di). By pigeonhole, some pair must violate this constraint, so some li or di exceeds n.

**Example 4: Modular arithmetic.** Among any n + 1 integers, two have the same remainder when divided by n (by pigeonhole on the n possible remainders).

---

## 4. Inclusion-Exclusion Principle

### Formula

For finite sets A1, A2, ..., An:

```
|A1 union A2 union ... union An| =
  sum_{i} |Ai|
  - sum_{i<j} |Ai intersect Aj|
  + sum_{i<j<k} |Ai intersect Aj intersect Ak|
  - ...
  + (-1)^{n+1} |A1 intersect A2 intersect ... intersect An|
```

### Two-Set Case

```
|A union B| = |A| + |B| - |A intersect B|
```

### Three-Set Case

```
|A union B union C| = |A| + |B| + |C|
                      - |A intersect B| - |A intersect C| - |B intersect C|
                      + |A intersect B intersect C|
```

### Worked Example: Derangements

A **derangement** is a permutation of {1, 2, ..., n} where no element appears in its original position.

Let Ai = {permutations where element i is in position i} (a "fixed point").

The number of permutations with AT LEAST ONE fixed point is |A1 union A2 union ... union An|.

By inclusion-exclusion:
- |Ai| = (n-1)! (fix element i, permute the rest)
- |Ai intersect Aj| = (n-2)! (fix elements i and j)
- In general, the intersection of k of the Ai's has size (n-k)!
- There are C(n, k) ways to choose which k sets to intersect.

```
|A1 union ... union An| = C(n,1)(n-1)! - C(n,2)(n-2)! + C(n,3)(n-3)! - ...
                        = n!/1! - n!/2! + n!/3! - ... + (-1)^{n+1} n!/n!
                        = sum_{k=1}^{n} (-1)^{k+1} * n! / k!
```

The number of derangements is:

```
D(n) = n! - |A1 union ... union An|
     = n! * sum_{k=0}^{n} (-1)^k / k!
     = n! * (1 - 1/1! + 1/2! - 1/3! + ... + (-1)^n / n!)
```

As n -> infinity, D(n) -> n!/e (since the series converges to e^{-1}).

**Values:** D(0) = 1, D(1) = 0, D(2) = 1, D(3) = 2, D(4) = 9, D(5) = 44.

**Probability:** The probability that a random permutation is a derangement is approximately 1/e ≈ 0.3679, regardless of n.

### Euler's Totient via Inclusion-Exclusion

If n = p1^{a1} * p2^{a2} * ... * pk^{ak}, then:

```
phi(n) = n * (1 - 1/p1) * (1 - 1/p2) * ... * (1 - 1/pk)
```

This follows from inclusion-exclusion: subtract multiples of each prime, add back multiples of pairs, etc.

---

## 5. Generating Functions

### Ordinary Generating Functions (OGF)

The OGF for a sequence a0, a1, a2, ... is the formal power series:

```
G(x) = sum_{n=0}^{infinity} a_n * x^n
```

**Key examples:**

| Sequence | OGF |
|---|---|
| 1, 1, 1, 1, ... | 1 / (1 - x) |
| 1, c, c^2, c^3, ... | 1 / (1 - cx) |
| 1, 0, 1, 0, 1, 0, ... | 1 / (1 - x^2) |
| C(n,0), C(n,1), ..., C(n,n) | (1 + x)^n |
| 0, 1, 1, 2, 3, 5, 8, ... (Fibonacci) | x / (1 - x - x^2) |
| 1, 1, 2, 5, 14, ... (Catalan) | (1 - sqrt(1 - 4x)) / (2x) |

### Operations on Generating Functions

| Operation | Effect on Sequence |
|---|---|
| c * G(x) | Multiply each term by c |
| G(x) + H(x) | Add corresponding terms |
| x^k * G(x) | Shift sequence right by k (prepend k zeros) |
| G'(x) | (n+1)*a_{n+1} replaces a_n |
| G(x) * H(x) | Convolution: c_n = sum_{k=0}^{n} a_k * b_{n-k} |

### Solving Recurrences with Generating Functions

**Example:** Solve the Fibonacci recurrence F(n) = F(n-1) + F(n-2), F(0) = 0, F(1) = 1.

Let G(x) = sum F(n) x^n. Then:

```
G(x) = x + x * G(x) + x^2 * G(x)
G(x) - x*G(x) - x^2*G(x) = x
G(x)(1 - x - x^2) = x
G(x) = x / (1 - x - x^2)
```

Using partial fractions with roots phi = (1 + sqrt(5))/2 and psi = (1 - sqrt(5))/2:

```
G(x) = (1/sqrt(5)) * (1/(1 - phi*x) - 1/(1 - psi*x))
```

Extracting coefficients: F(n) = (phi^n - psi^n) / sqrt(5). This is the closed-form Binet formula.

---

## 6. Probability Foundations

### Sample Spaces

A **sample space** S is the set of all possible outcomes of an experiment. An **event** A is a subset of S.

**Probability axioms (Kolmogorov):**
1. Pr[A] >= 0 for every event A.
2. Pr[S] = 1.
3. For mutually exclusive events A, B: Pr[A union B] = Pr[A] + Pr[B].

For finite uniform sample spaces: Pr[A] = |A| / |S|.

### Basic Rules

| Rule | Formula |
|---|---|
| Complement | Pr[A_bar] = 1 - Pr[A] |
| Union | Pr[A union B] = Pr[A] + Pr[B] - Pr[A intersect B] |
| Monotonicity | If A is a subset of B, then Pr[A] <= Pr[B] |
| Union bound | Pr[A1 union ... union An] <= sum Pr[Ai] |

### Conditional Probability

```
Pr[A | B] = Pr[A intersect B] / Pr[B],    provided Pr[B] > 0
```

**Product rule (chain rule):**
```
Pr[A intersect B] = Pr[A | B] * Pr[B] = Pr[B | A] * Pr[A]
Pr[A1 intersect A2 intersect ... intersect An] = Pr[A1] * Pr[A2|A1] * Pr[A3|A1 intersect A2] * ...
```

### Law of Total Probability

If B1, B2, ..., Bn is a partition of S (disjoint, union equals S), then:

```
Pr[A] = sum_{i=1}^{n} Pr[A | Bi] * Pr[Bi]
```

### Bayes' Theorem

```
Pr[A | B] = Pr[B | A] * Pr[A] / Pr[B]
```

With a partition B1, ..., Bn:

```
Pr[Bi | A] = Pr[A | Bi] * Pr[Bi] / sum_{j=1}^{n} Pr[A | Bj] * Pr[Bj]
```

**Worked Example:** A test for a disease has 99% sensitivity (Pr[positive | disease] = 0.99) and 95% specificity (Pr[negative | no disease] = 0.95). The disease prevalence is 1% (Pr[disease] = 0.01). What is Pr[disease | positive]?

```
Pr[positive] = Pr[pos | disease]*Pr[disease] + Pr[pos | no disease]*Pr[no disease]
             = 0.99 * 0.01 + 0.05 * 0.99
             = 0.0099 + 0.0495
             = 0.0594

Pr[disease | positive] = 0.0099 / 0.0594 ≈ 0.1667
```

Despite the good test, only ~17% of positive results are true positives because the disease is rare. This is the **base rate fallacy**.

### Independence

Events A and B are independent iff Pr[A intersect B] = Pr[A] * Pr[B].

Equivalently: Pr[A | B] = Pr[A] (when Pr[B] > 0).

**Mutual independence:** Events A1, ..., An are mutually independent iff for every subset I of {1, ..., n}:

```
Pr[intersection of Ai for i in I] = product of Pr[Ai] for i in I
```

Note: pairwise independence does NOT imply mutual independence.

---

## 7. Random Variables

A **random variable** X is a function from the sample space S to the real numbers.

### Discrete Probability Distributions

| Distribution | Notation | PMF: Pr[X = k] | E[X] | Var(X) | When to Use |
|---|---|---|---|---|---|
| Bernoulli | Bern(p) | p if k=1, (1-p) if k=0 | p | p(1-p) | Single yes/no trial |
| Binomial | Bin(n, p) | C(n,k) p^k (1-p)^{n-k} | np | np(1-p) | Number of successes in n independent trials |
| Geometric | Geom(p) | (1-p)^{k-1} p, k=1,2,... | 1/p | (1-p)/p^2 | Number of trials until first success |
| Negative Binomial | NB(r, p) | C(k-1, r-1) p^r (1-p)^{k-r} | r/p | r(1-p)/p^2 | Trials until r-th success |
| Poisson | Pois(lambda) | e^{-lambda} lambda^k / k! | lambda | lambda | Rare events in a large population |
| Hypergeometric | Hyp(N, K, n) | C(K,k)C(N-K,n-k)/C(N,n) | nK/N | nK(N-K)(N-n)/(N^2(N-1)) | Sampling without replacement |
| Uniform | Unif{a,...,b} | 1/(b-a+1) for a <= k <= b | (a+b)/2 | ((b-a+1)^2 - 1)/12 | Equally likely outcomes |

---

## 8. Expectation

### Definition

```
E[X] = sum_{x} x * Pr[X = x]
```

### Linearity of Expectation

For any random variables X, Y (not necessarily independent):

```
E[X + Y] = E[X] + E[Y]
E[cX] = c * E[X]
```

**This is the most powerful tool in discrete probability.** It does not require independence.

### Indicator Variable Method

Define indicator variables Xi = 1 if event i occurs, 0 otherwise. Then:

```
E[Xi] = Pr[event i occurs]
E[sum Xi] = sum E[Xi] = sum Pr[event i occurs]
```

**Example: Expected number of fixed points in a random permutation.**

Let Xi = 1 if element i is in position i. Then E[Xi] = 1/n.

```
E[number of fixed points] = E[X1 + X2 + ... + Xn] = n * (1/n) = 1
```

So the expected number of fixed points is 1, regardless of n.

**Example: Coupon collector problem.** There are n coupon types. Each trial gives a uniformly random coupon. Expected number of trials to collect all n types?

Let Xi = number of trials to get a new type when you already have i-1 types. Then Xi ~ Geom(p_i) where p_i = (n - i + 1)/n.

```
E[total trials] = sum_{i=1}^{n} E[Xi] = sum_{i=1}^{n} n/(n - i + 1) = n * sum_{j=1}^{n} 1/j = n * H_n
```

where H_n is the n-th harmonic number. H_n ≈ ln(n) + 0.5772 (Euler-Mascheroni constant).

For n = 100: E ≈ 100 * 5.187 ≈ 519 trials.

### Conditional Expectation

```
E[X | A] = sum_{x} x * Pr[X = x | A]
```

**Law of total expectation:** If B1, ..., Bn partition S:
```
E[X] = sum_{i} E[X | Bi] * Pr[Bi]
```

### Product of Expectations (Independent Variables Only)

If X and Y are independent: E[XY] = E[X] * E[Y].

Warning: This does NOT hold for dependent variables.

---

## 9. Variance and Standard Deviation

### Definition

```
Var(X) = E[(X - E[X])^2] = E[X^2] - (E[X])^2
```

**Standard deviation:** sigma(X) = sqrt(Var(X)).

### Properties

| Property | Formula | Condition |
|---|---|---|
| Non-negativity | Var(X) >= 0 | Always |
| Scaling | Var(cX) = c^2 * Var(X) | Always |
| Shift invariance | Var(X + c) = Var(X) | Always |
| Sum (independent) | Var(X + Y) = Var(X) + Var(Y) | X, Y independent |
| Sum (general) | Var(X + Y) = Var(X) + Var(Y) + 2*Cov(X,Y) | Always |

**Covariance:** Cov(X, Y) = E[XY] - E[X]*E[Y]. For independent variables, Cov(X, Y) = 0.

---

## 10. Tail Bounds

Tail bounds give upper limits on the probability that a random variable deviates far from its expected value. They are crucial for algorithm analysis and probabilistic arguments.

### Markov's Inequality

**Statement.** For a non-negative random variable X and any a > 0:

```
Pr[X >= a] <= E[X] / a
```

**Proof.** E[X] = sum_{x >= 0} x * Pr[X = x] >= sum_{x >= a} x * Pr[X = x] >= a * sum_{x >= a} Pr[X = x] = a * Pr[X >= a]. QED.

**When to use:** You only know E[X] and X >= 0. This is the weakest bound but has the fewest requirements.

**Example:** Average salary in a company is $60,000. At most what fraction earns >= $200,000? By Markov: Pr[salary >= 200000] <= 60000/200000 = 0.30.

### Chebyshev's Inequality

**Statement.** For any random variable X with mean mu and variance sigma^2, for any k > 0:

```
Pr[|X - mu| >= k * sigma] <= 1/k^2
```

Equivalently, for any t > 0:

```
Pr[|X - mu| >= t] <= sigma^2 / t^2
```

**Proof.** Apply Markov's inequality to the non-negative random variable (X - mu)^2:
Pr[(X - mu)^2 >= t^2] <= E[(X - mu)^2] / t^2 = Var(X) / t^2. QED.

**When to use:** You know both E[X] and Var(X). Stronger than Markov when variance is small.

**Example:** An algorithm's running time has E[T] = 100 and Var(T) = 25 (so sigma = 5). What is the probability that T deviates from 100 by more than 20?
Pr[|T - 100| >= 20] <= 25/400 = 0.0625 (at most 6.25%).

### Chernoff Bounds

**Statement (multiplicative form).** Let X = X1 + X2 + ... + Xn where Xi are independent Bernoulli random variables. Let mu = E[X]. Then for any delta > 0:

**Upper tail:**
```
Pr[X >= (1 + delta) * mu] <= (e^delta / (1 + delta)^{(1+delta)})^mu
```

**Simplified upper tail (for 0 < delta < 1):**
```
Pr[X >= (1 + delta) * mu] <= exp(-mu * delta^2 / 3)
```

**Lower tail:**
```
Pr[X <= (1 - delta) * mu] <= exp(-mu * delta^2 / 2)
```

**Two-sided:**
```
Pr[|X - mu| >= delta * mu] <= 2 * exp(-mu * delta^2 / 3)
```

**When to use:** X is a sum of independent bounded random variables. This gives exponentially decreasing tail probabilities, much tighter than Chebyshev.

**Example:** Flip a fair coin 1000 times. What is the probability of getting at least 600 heads?

mu = 500, want Pr[X >= 600] = Pr[X >= (1 + 0.2) * 500], so delta = 0.2.

```
Pr[X >= 600] <= exp(-500 * 0.04 / 3) = exp(-6.67) ≈ 0.00127
```

Compare with Chebyshev: Var(X) = 250, so Pr[|X - 500| >= 100] <= 250/10000 = 0.025. Chernoff is ~20x tighter.

### When to Use Which Bound

| Situation | Best Bound | Reason |
|---|---|---|
| Only know mean, X >= 0 | Markov | Only option |
| Know mean and variance | Chebyshev | Uses variance info for tighter bound |
| Sum of independent bounded RVs | Chernoff | Exponential decay, much tighter |
| Need quick back-of-envelope | Markov or Chebyshev | Simple formulas |
| Need tight concentration | Chernoff | Optimal for independent sums |
| Dependent random variables | Chebyshev (if variance known) | Chernoff requires independence |

---

## 11. Quick Reference: Common Counting Formulas

| Problem | Formula | Conditions |
|---|---|---|
| Arrangements of n distinct objects | n! | Order matters |
| k-permutations of n | P(n,k) = n!/(n-k)! | Order matters, no repetition |
| k-combinations of n | C(n,k) = n!/(k!(n-k)!) | Order doesn't matter, no repetition |
| k-selections with repetition | n^k | Order matters, repetition allowed |
| k-multisets from n types | C(n+k-1, k) | Order doesn't matter, repetition allowed |
| Arrangements with repeats | n!/(n1!*n2!*...*nr!) | Multinomial |
| Derangements | D(n) = n! * sum (-1)^k/k! | No fixed points |
| Surjections from n to k | sum_{i=0}^{k} (-1)^i C(k,i)(k-i)^n | Inclusion-exclusion |
| Stirling numbers S(n,k) | (1/k!) * sum_{i=0}^{k} (-1)^i C(k,i)(k-i)^n | Partitions of n into k non-empty subsets |
| Bell number B(n) | sum_{k=0}^{n} S(n,k) | Total partitions of n-element set |
| Catalan number C_n | C(2n,n)/(n+1) | Binary trees, valid parenthesizations, etc. |

---

## 12. Useful Approximations

| Approximation | Statement | When Valid |
|---|---|---|
| Stirling's formula | n! ≈ sqrt(2*pi*n) * (n/e)^n | Large n |
| Binomial ≈ Poisson | Bin(n, p) ≈ Pois(np) | Large n, small p, np moderate |
| Binomial ≈ Normal | Bin(n, p) ≈ N(np, np(1-p)) | Large n, np >= 5, n(1-p) >= 5 |
| Harmonic number | H_n = sum_{k=1}^{n} 1/k ≈ ln(n) + 0.5772 | Large n |
| (1 - 1/n)^n ≈ 1/e | Converges from below | Large n |
| (1 + x/n)^n ≈ e^x | Converges | Large n |
| C(n, k) ≈ n^k / k! | When k is much smaller than n | k << n |
