# Proof Techniques Reference

Comprehensive templates and worked examples for the core proof methods in discrete mathematics.

## 1. Direct Proof

### Template

**Claim:** If P, then Q.

**Proof.** Assume P holds. [Derive intermediate results using definitions, axioms, and previously proven theorems.] Therefore Q holds. QED.

### Worked Example: Sum of Two Even Numbers Is Even

**Claim:** If a and b are even integers, then a + b is even.

**Proof.** Assume a and b are even integers. By definition, there exist integers k and l such that a = 2k and b = 2l. Then:

```
a + b = 2k + 2l = 2(k + l)
```

Since k + l is an integer, a + b = 2(k + l) is even by definition. QED.

### Worked Example: Product of Two Odd Numbers Is Odd

**Claim:** If a and b are odd integers, then a * b is odd.

**Proof.** Since a is odd, a = 2k + 1 for some integer k. Since b is odd, b = 2l + 1 for some integer l. Then:

```
a * b = (2k + 1)(2l + 1) = 4kl + 2k + 2l + 1 = 2(2kl + k + l) + 1
```

Since 2kl + k + l is an integer, a * b is odd by definition. QED.

### Worked Example: Divisibility Is Transitive

**Claim:** If a | b and b | c, then a | c.

**Proof.** Since a | b, we have b = ka for some integer k. Since b | c, we have c = lb for some integer l. Substituting: c = lb = l(ka) = (lk)a. Since lk is an integer, a | c. QED.

---

## 2. Proof by Contradiction

### Template

**Claim:** P is true.

**Proof.** Assume for contradiction that P is false (i.e., assume NOT P). [Derive logical consequences until reaching a statement that contradicts a known fact, an axiom, or the assumption itself.] This is a contradiction. Therefore P must be true. QED.

### Worked Example: sqrt(2) Is Irrational

**Claim:** sqrt(2) is irrational.

**Proof.** Assume for contradiction that sqrt(2) is rational. Then sqrt(2) = a/b where a, b are integers with b != 0 and gcd(a, b) = 1 (the fraction is in lowest terms).

Squaring both sides: 2 = a^2 / b^2, so a^2 = 2b^2.

This means a^2 is even, which implies a is even (since the square of an odd number is odd). Write a = 2k for some integer k.

Substituting: (2k)^2 = 2b^2, so 4k^2 = 2b^2, hence b^2 = 2k^2.

This means b^2 is even, so b is even.

But now both a and b are even, contradicting gcd(a, b) = 1. This is a contradiction. Therefore sqrt(2) is irrational. QED.

### Worked Example: Infinitely Many Primes

**Claim:** There are infinitely many primes.

**Proof.** Assume for contradiction that there are finitely many primes: p1, p2, ..., pn. Consider the number N = p1 * p2 * ... * pn + 1. Since N > 1, by the Fundamental Theorem of Arithmetic, N has a prime divisor p. This prime p must be one of p1, ..., pn (since those are all the primes). But then p | N and p | (p1 * p2 * ... * pn), so p | (N - p1 * p2 * ... * pn) = 1. No prime divides 1, a contradiction. Therefore there are infinitely many primes. QED.

### Worked Example: No Largest Integer

**Claim:** There is no largest integer.

**Proof.** Assume for contradiction that there exists a largest integer M. Consider M + 1. Since M is an integer, M + 1 is also an integer, and M + 1 > M. This contradicts the assumption that M is the largest integer. QED.

---

## 3. Proof by Contrapositive

### Template

**Claim:** If P, then Q.

**Proof.** We prove the contrapositive: if NOT Q, then NOT P. Assume NOT Q. [Derive NOT P using valid reasoning steps.] Therefore NOT P. Since the contrapositive is logically equivalent to the original statement, the claim holds. QED.

### Worked Example: n^2 Even Implies n Even

**Claim:** For any integer n, if n^2 is even, then n is even.

**Proof.** We prove the contrapositive: if n is odd, then n^2 is odd.

Assume n is odd. Then n = 2k + 1 for some integer k. We have:

```
n^2 = (2k + 1)^2 = 4k^2 + 4k + 1 = 2(2k^2 + 2k) + 1
```

Since 2k^2 + 2k is an integer, n^2 is odd. QED.

### Worked Example: If ab Is Odd, Then Both a and b Are Odd

**Claim:** For integers a and b, if ab is odd, then both a and b are odd.

**Proof.** Contrapositive: if a is even or b is even, then ab is even.

Assume without loss of generality that a is even. Then a = 2k for some integer k, so ab = 2kb. Since kb is an integer, ab is even. QED.

### Worked Example: Divisibility and Coprimality

**Claim:** For integers a, b, and prime p, if p | ab, then p | a or p | b.

**Proof.** Contrapositive: if p does not divide a and p does not divide b, then p does not divide ab.

Assume gcd(p, a) = 1 and gcd(p, b) = 1. By Bezout's identity, there exist integers s, t such that sp + ta = 1, and integers u, v such that up + vb = 1. Multiplying these equations:

```
(sp + ta)(up + vb) = 1
sup^2 + spvb + taup + tavb = 1
p(sup + svb + tau) + ab(tv) = 1
```

This shows gcd(p, ab) = 1, so p does not divide ab. QED.

---

## 4. Proof by Cases

### Template

**Claim:** P is true.

**Proof.** We consider all possible cases. Note that [argument that cases are exhaustive].

**Case 1:** [First case]. [Proof for this case.]

**Case 2:** [Second case]. [Proof for this case.]

...

Since all cases have been covered and P holds in each, the claim is proved. QED.

### Worked Example: |ab| = |a| * |b|

**Claim:** For all real numbers a and b, |ab| = |a| * |b|.

**Proof.** We consider four cases based on the signs of a and b.

**Case 1: a >= 0 and b >= 0.** Then ab >= 0, so |ab| = ab = |a| * |b|.

**Case 2: a >= 0 and b < 0.** Then ab <= 0, so |ab| = -ab = a * (-b) = |a| * |b|.

**Case 3: a < 0 and b >= 0.** Then ab <= 0, so |ab| = -ab = (-a) * b = |a| * |b|.

**Case 4: a < 0 and b < 0.** Then ab > 0, so |ab| = ab = (-a)(-b) = |a| * |b|.

In all cases, |ab| = |a| * |b|. QED.

### Worked Example: n^3 - n Is Divisible by 6

**Claim:** For every integer n, 6 | (n^3 - n).

**Proof.** Factor: n^3 - n = n(n-1)(n+1) = (n-1)n(n+1), the product of three consecutive integers.

**Divisibility by 2:** Among any two consecutive integers, one is even. So (n-1)n(n+1) is divisible by 2.

**Divisibility by 3:** Among any three consecutive integers, one is divisible by 3. This can be verified by cases on n mod 3:
- Case n = 0 (mod 3): n is divisible by 3.
- Case n = 1 (mod 3): n - 1 is divisible by 3.
- Case n = 2 (mod 3): n + 1 is divisible by 3.

Since (n-1)n(n+1) is divisible by both 2 and 3, and gcd(2,3) = 1, it is divisible by 6. QED.

---

## 5. Ordinary Induction

### Template

**Claim:** For all n >= b, P(n).

**Proof.** By induction on n.

**Base case (n = b):** [Verify P(b) directly.]

**Inductive step:** Assume P(n) holds for some arbitrary n >= b (inductive hypothesis). We must show P(n+1).

[Use the inductive hypothesis P(n) to derive P(n+1).]

By the principle of mathematical induction, P(n) holds for all n >= b. QED.

### Worked Example: Sum of First n Natural Numbers

**Claim:** For all n >= 1, 1 + 2 + ... + n = n(n+1)/2.

**Proof.** By induction on n.

**Base case (n = 1):** LHS = 1. RHS = 1(2)/2 = 1. LHS = RHS. Verified.

**Inductive step:** Assume 1 + 2 + ... + n = n(n+1)/2 for some n >= 1.

We must show: 1 + 2 + ... + n + (n+1) = (n+1)(n+2)/2.

```
LHS = [1 + 2 + ... + n] + (n+1)
    = n(n+1)/2 + (n+1)          [by inductive hypothesis]
    = n(n+1)/2 + 2(n+1)/2
    = (n+1)(n+2)/2
    = RHS
```

By induction, the claim holds for all n >= 1. QED.

### Worked Example: Sum of Geometric Series

**Claim:** For all n >= 0 and r != 1, sum_{i=0}^{n} r^i = (r^(n+1) - 1) / (r - 1).

**Proof.** By induction on n.

**Base case (n = 0):** LHS = r^0 = 1. RHS = (r^1 - 1) / (r - 1) = (r - 1)/(r - 1) = 1. Verified.

**Inductive step:** Assume sum_{i=0}^{n} r^i = (r^(n+1) - 1) / (r - 1).

```
sum_{i=0}^{n+1} r^i = [sum_{i=0}^{n} r^i] + r^(n+1)
                     = (r^(n+1) - 1)/(r - 1) + r^(n+1)
                     = (r^(n+1) - 1)/(r - 1) + r^(n+1)(r - 1)/(r - 1)
                     = (r^(n+1) - 1 + r^(n+2) - r^(n+1)) / (r - 1)
                     = (r^(n+2) - 1) / (r - 1)
```

This is the formula with n replaced by n+1. QED.

### Worked Example: Number of Subsets

**Claim:** A set with n elements has exactly 2^n subsets.

**Proof.** By induction on n.

**Base case (n = 0):** The empty set has one subset (itself). 2^0 = 1. Verified.

**Inductive step:** Assume every set with n elements has 2^n subsets. Let S be a set with n + 1 elements. Pick any element x in S, and let T = S - {x}, which has n elements.

Every subset of S either contains x or does not:
- Subsets not containing x: these are exactly the subsets of T, numbering 2^n.
- Subsets containing x: for each subset A of T, the set A union {x} is a subset of S containing x, giving 2^n such subsets.

Total: 2^n + 2^n = 2^(n+1). QED.

---

## 6. Strong Induction

### Template

**Claim:** For all n >= b, P(n).

**Proof.** By strong induction on n.

**Base case(s):** [Verify P(b) (and possibly P(b+1), ..., P(b+k) if the inductive step requires multiple predecessors).]

**Inductive step:** Assume P(k) holds for all b <= k <= n (strong inductive hypothesis). We must show P(n+1).

[Use any or all of P(b), P(b+1), ..., P(n) to derive P(n+1).]

By the principle of strong induction, P(n) holds for all n >= b. QED.

### Worked Example: Fundamental Theorem of Arithmetic (Existence)

**Claim:** Every integer n >= 2 can be written as a product of primes.

**Proof.** By strong induction on n.

**Base case (n = 2):** 2 is prime, so it is a product of primes (a product with one factor). Verified.

**Inductive step:** Assume every integer k with 2 <= k <= n can be written as a product of primes. Consider n + 1.

**Case 1:** n + 1 is prime. Then n + 1 is a product of primes (a single factor). Done.

**Case 2:** n + 1 is composite. Then n + 1 = a * b where 2 <= a, b <= n. By the strong inductive hypothesis, both a and b can be written as products of primes. Therefore n + 1 = a * b is also a product of primes. QED.

### Worked Example: Every Amount >= 12 Cents Can Be Made with 4-cent and 5-cent Stamps

**Claim:** For all n >= 12, n can be written as 4a + 5b for non-negative integers a and b.

**Proof.** By strong induction on n.

**Base cases:**
- n = 12: 12 = 4(3) + 5(0). Verified.
- n = 13: 13 = 4(2) + 5(1). Verified.
- n = 14: 14 = 4(1) + 5(2). Verified.
- n = 15: 15 = 4(0) + 5(3). Verified.

**Inductive step:** Assume the claim holds for all k with 12 <= k <= n, where n >= 15. We show it for n + 1.

Since n + 1 >= 16, we have (n + 1) - 4 = n - 3 >= 12. By the strong inductive hypothesis, n - 3 = 4a + 5b for some non-negative a, b. Then:

```
n + 1 = (n - 3) + 4 = 4a + 5b + 4 = 4(a + 1) + 5b
```

So n + 1 can be represented. QED.

### Worked Example: Binary Representation

**Claim:** Every positive integer n has a binary representation (can be written as a sum of distinct powers of 2).

**Proof.** By strong induction on n.

**Base case (n = 1):** 1 = 2^0. Verified.

**Inductive step:** Assume every positive integer k <= n has a binary representation. Consider n + 1.

**Case 1:** n + 1 is even. Then (n+1)/2 is a positive integer <= n. By the strong inductive hypothesis, (n+1)/2 = 2^{a1} + 2^{a2} + ... + 2^{am} with a1 < a2 < ... < am. So n + 1 = 2^{a1+1} + 2^{a2+1} + ... + 2^{am+1}, which is a sum of distinct powers of 2.

**Case 2:** n + 1 is odd. Then n is even and n >= 2 (since n + 1 >= 2). We have n = 2t for some positive integer t <= n. By the strong inductive hypothesis, t has a binary representation t = 2^{a1} + ... + 2^{am} with a1 >= 0. So n = 2^{a1+1} + ... + 2^{am+1}, and all exponents are >= 1. Therefore n + 1 = 2^0 + 2^{a1+1} + ... + 2^{am+1}, which uses distinct powers of 2 (since 0 < a1 + 1). QED.

---

## Which Proof Method to Choose: A Decision Guide

Use the following flowchart to select the right proof technique:

```
START: What are you trying to prove?
  |
  +-- A statement of the form "If P then Q"?
  |     |
  |     +-- Can you derive Q directly from P?
  |     |     YES --> Direct Proof
  |     |
  |     +-- Is "If NOT Q then NOT P" easier?
  |     |     YES --> Proof by Contrapositive
  |     |
  |     +-- Does assuming P and NOT Q lead to
  |     |   an obvious contradiction?
  |     |     YES --> Proof by Contradiction
  |     |
  |     +-- Does the problem split into 2-4
  |           exhaustive sub-cases?
  |           YES --> Proof by Cases
  |
  +-- A statement "For all n >= b, P(n)" about integers?
  |     |
  |     +-- Does proving P(n+1) only need P(n)?
  |     |     YES --> Ordinary Induction
  |     |
  |     +-- Does P(n+1) depend on P(k) for
  |     |   multiple k < n+1?
  |     |     YES --> Strong Induction
  |     |
  |     +-- Is the claim about a recursively
  |           defined structure (trees, lists)?
  |           YES --> Structural Induction
  |
  +-- An existence or non-existence claim?
  |     |
  |     +-- Non-existence among natural numbers?
  |     |     --> Well-Ordering Principle or
  |     |         Proof by Contradiction
  |     |
  |     +-- Existence claim?
  |           --> Constructive proof (exhibit example)
  |               or Contradiction (assume no example exists)
  |
  +-- An equivalence "P if and only if Q"?
        --> Prove both directions:
            "P implies Q" AND "Q implies P"
            (choose method for each direction independently)
```

### Quick Reference Table

| Situation | Recommended Method | Reason |
|---|---|---|
| Algebraic identity | Direct proof | Manipulate one side to equal the other |
| Irrationality | Contradiction | Assume rational form, derive impossibility |
| Divisibility property | Direct proof or cases | Factor and use definitions |
| Property of n for all n in N | Induction | Natural fit for statements parameterized by integers |
| Recursive algorithm correctness | Strong induction | Algorithm may recurse on values much smaller than n-1 |
| "At least one exists" | Pigeonhole or contradiction | Pigeonhole for combinatorial; contradiction for logical |
| Inequality | Direct proof, induction, or contradiction | Depends on whether it is parameterized |
| Graph property for all n-vertex graphs | Induction on n or on edges | Choose the parameter that simplifies the inductive step |
| Equivalence of definitions | Bidirectional proof | Prove each implication separately |
| Uniqueness | Assume two exist, show they are equal | Direct or contradiction |

### Common Mistakes to Avoid

1. **Circular reasoning**: Do not assume what you are trying to prove. In a direct proof of "P implies Q," you may assume P but not Q.

2. **Wrong direction in contrapositive**: The contrapositive of "P implies Q" is "NOT Q implies NOT P," not "NOT P implies NOT Q" (that is the inverse).

3. **Incomplete case analysis**: Ensure your cases are exhaustive. If proving something for all integers, cases like "n > 0, n = 0, n < 0" cover everything; "n > 0, n < 0" does not.

4. **Forgetting the base case in induction**: The inductive step alone proves nothing. Always verify at least one base case. If the inductive step uses P(n-1) and P(n-2), you need two base cases.

5. **Inductive hypothesis too weak or too strong**: The inductive hypothesis must match the claim exactly. Strengthening the claim (proving a stronger result) sometimes makes induction easier.

6. **Misidentifying the induction variable**: In problems involving multiple parameters, choose the right variable to induct on. Often it is the one that decreases in the recursive step.

7. **Using specific examples as proof**: Verifying P(1), P(2), P(3) does not prove P(n) for all n. Examples can only disprove universal statements (by counterexample).

8. **Contradiction vs. contrapositive confusion**: In a proof by contradiction of "P implies Q," you assume P and NOT Q and derive any contradiction. In a proof by contrapositive, you assume NOT Q and derive NOT P specifically.
