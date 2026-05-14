# Number Theory and RSA Cryptography Reference

Building from basic divisibility through modular arithmetic to the RSA cryptosystem, with complete proofs and a worked numerical example.

## 1. Divisibility

**Definition.** For integers a and b with a != 0, we say a divides b (written a | b) if there exists an integer k such that b = ka.

**Properties of Divisibility:**

1. **Reflexivity**: a | a for all a != 0 (since a = 1 * a).
2. **Transitivity**: if a | b and b | c, then a | c.
3. **Linearity**: if a | b and a | c, then a | (sb + tc) for all integers s, t.
4. **Comparison**: if a | b and b != 0, then |a| <= |b|.
5. **Product**: if a | b, then a | bc for any integer c.

**Division Algorithm.** For any integer a and positive integer d, there exist unique integers q (quotient) and r (remainder) such that:

```
a = dq + r,    where 0 <= r < d
```

We write r = a mod d and q = a div d.

---

## 2. Greatest Common Divisor

**Definition.** The greatest common divisor of integers a and b (not both zero), written gcd(a, b), is the largest positive integer that divides both a and b.

**Properties:**
- gcd(a, b) = gcd(b, a)
- gcd(a, 0) = |a|
- gcd(a, b) = gcd(a, b mod a)
- a and b are coprime (relatively prime) iff gcd(a, b) = 1

### Euclidean Algorithm

Computes gcd(a, b) by repeated application of the division algorithm:

```
gcd(a, b) = gcd(b, a mod b)     [recursive step]
gcd(a, 0) = a                    [base case]
```

**Worked Example:** gcd(252, 198)

```
gcd(252, 198) = gcd(198, 252 mod 198) = gcd(198, 54)
gcd(198, 54)  = gcd(54, 198 mod 54)   = gcd(54, 36)
gcd(54, 36)   = gcd(36, 54 mod 36)    = gcd(36, 18)
gcd(36, 18)   = gcd(18, 36 mod 18)    = gcd(18, 0)
gcd(18, 0)    = 18
```

So gcd(252, 198) = 18.

**Running Time.** The Euclidean algorithm runs in O(log(min(a, b))) division steps. More precisely, the number of steps is at most 5 times the number of digits in the smaller input.

### Bezout's Identity

**Theorem.** For any integers a and b (not both zero), there exist integers s and t such that:

```
gcd(a, b) = sa + tb
```

The coefficients s and t can be found by the **Extended Euclidean Algorithm**, which works backwards through the Euclidean algorithm steps.

**Worked Example:** Find s, t such that gcd(252, 198) = 252s + 198t.

Working forward (from above):
```
252 = 1 * 198 + 54    -->  54 = 252 - 1 * 198
198 = 3 * 54 + 36     -->  36 = 198 - 3 * 54
54  = 1 * 36 + 18     -->  18 = 54 - 1 * 36
36  = 2 * 18 + 0
```

Working backward (back-substitution):
```
18 = 54 - 1 * 36
   = 54 - 1 * (198 - 3 * 54)
   = 4 * 54 - 1 * 198
   = 4 * (252 - 1 * 198) - 1 * 198
   = 4 * 252 - 5 * 198
```

So s = 4, t = -5, and indeed 4(252) - 5(198) = 1008 - 990 = 18. Verified.

**Key Corollary.** gcd(a, b) = 1 if and only if there exist integers s, t with sa + tb = 1.

---

## 3. Primes and the Fundamental Theorem of Arithmetic

**Definition.** An integer p > 1 is prime if its only positive divisors are 1 and p. An integer greater than 1 that is not prime is composite.

**Fundamental Theorem of Arithmetic (FTA).** Every integer n > 1 can be written uniquely (up to order) as a product of primes:

```
n = p1^{a1} * p2^{a2} * ... * pk^{ak}
```

where p1 < p2 < ... < pk are primes and each ai >= 1.

**Proof sketch (existence):** By strong induction. If n is prime, done. If composite, n = ab with 1 < a, b < n. By strong IH, both a and b have prime factorizations. Their concatenation gives a prime factorization of n.

**Proof sketch (uniqueness):** Uses the key lemma: if prime p | ab, then p | a or p | b (proved via Bezout's identity). Suppose n has two prime factorizations. By the lemma, each prime in one factorization must appear in the other, leading to the same factorization.

### Distribution of Primes

**Prime Number Theorem.** Let pi(n) denote the number of primes <= n. Then:

```
pi(n) ~ n / ln(n)    as n -> infinity
```

Equivalently, the probability that a randomly chosen number near n is prime is approximately 1/ln(n).

**Sieve of Eratosthenes.** To find all primes up to n: list integers from 2 to n. Starting with the smallest unmarked number p, mark all multiples of p (starting from p^2). Repeat until p^2 > n. The remaining unmarked numbers are prime. Time complexity: O(n log log n).

---

## 4. Modular Arithmetic

**Definition.** For integers a, b and positive integer m, we say a is congruent to b modulo m, written:

```
a ≡ b (mod m)
```

if and only if m | (a - b), equivalently, a mod m = b mod m.

**Arithmetic Properties.** If a ≡ a' (mod m) and b ≡ b' (mod m), then:

- a + b ≡ a' + b' (mod m)
- a - b ≡ a' - b' (mod m)
- a * b ≡ a' * b' (mod m)
- a^k ≡ a'^k (mod m) for non-negative integer k

**WARNING:** Division is NOT generally valid in modular arithmetic. The equation ax ≡ b (mod m) has a solution iff gcd(a, m) | b.

### Multiplicative Inverse

**Definition.** The multiplicative inverse of a modulo m is an integer a^{-1} such that:

```
a * a^{-1} ≡ 1 (mod m)
```

**Existence.** a^{-1} mod m exists if and only if gcd(a, m) = 1.

**Computation.** Use the Extended Euclidean Algorithm: find s, t such that sa + tm = 1. Then s ≡ a^{-1} (mod m).

**Worked Example:** Find 7^{-1} mod 30.

Apply the Extended Euclidean Algorithm:
```
30 = 4 * 7 + 2    -->  2 = 30 - 4 * 7
7  = 3 * 2 + 1    -->  1 = 7 - 3 * 2
2  = 2 * 1 + 0
```

Back-substitution:
```
1 = 7 - 3 * 2
  = 7 - 3 * (30 - 4 * 7)
  = 13 * 7 - 3 * 30
```

So 7^{-1} ≡ 13 (mod 30). Check: 7 * 13 = 91 = 3 * 30 + 1 ≡ 1 (mod 30). Verified.

### Fast Modular Exponentiation

To compute a^k mod m efficiently, use **repeated squaring** (binary exponentiation):

```
function mod_exp(a, k, m):
    result = 1
    a = a mod m
    while k > 0:
        if k is odd:
            result = (result * a) mod m
        k = k >> 1       // integer division by 2
        a = (a * a) mod m
    return result
```

Time complexity: O(log k) multiplications modulo m. Each multiplication involves numbers < m, so the total bit complexity is O(log(k) * log^2(m)).

---

## 5. Euler's Totient Function

**Definition.** Euler's totient function phi(n) counts the number of integers in {1, 2, ..., n} that are coprime to n.

**Formulas:**

| n | phi(n) | Formula |
|---|---|---|
| Prime p | p - 1 | All integers 1 through p-1 are coprime to p |
| Prime power p^k | p^k - p^{k-1} | Exactly p^{k-1} multiples of p in {1,...,p^k} |
| Product mn, gcd(m,n)=1 | phi(m) * phi(n) | Multiplicativity via Chinese Remainder Theorem |
| General n = p1^{a1} ... pk^{ak} | n * prod(1 - 1/pi) | Product formula over prime divisors |

**Worked Examples:**
- phi(12) = phi(4) * phi(3) = (4 - 2)(3 - 1) = 2 * 2 = 4. The coprime residues: {1, 5, 7, 11}.
- phi(100) = phi(4) * phi(25) = 2 * 20 = 40.
- phi(p * q) = (p - 1)(q - 1) for distinct primes p, q. This is the formula used in RSA.

---

## 6. Euler's Theorem and Fermat's Little Theorem

### Euler's Theorem

**Theorem.** If gcd(a, n) = 1, then:

```
a^{phi(n)} ≡ 1 (mod n)
```

**Proof sketch.** Let r1, r2, ..., r_{phi(n)} be the phi(n) integers in {1, ..., n} coprime to n (the "reduced residue system"). Consider the products ar1, ar2, ..., ar_{phi(n)} modulo n. Since gcd(a, n) = 1, multiplying by a permutes the reduced residue system. Therefore:

```
(ar1)(ar2)...(ar_{phi(n)}) ≡ r1 * r2 * ... * r_{phi(n)} (mod n)
a^{phi(n)} * (r1 * r2 * ... * r_{phi(n)}) ≡ r1 * r2 * ... * r_{phi(n)} (mod n)
```

Since each ri is coprime to n, their product is coprime to n. Dividing both sides (multiplying by the inverse):

```
a^{phi(n)} ≡ 1 (mod n)
```

### Fermat's Little Theorem

**Theorem.** If p is prime and p does not divide a, then:

```
a^{p-1} ≡ 1 (mod p)
```

This is Euler's theorem with n = p, since phi(p) = p - 1.

**Corollary.** For prime p and any integer a: a^p ≡ a (mod p).

**Worked Example:** Compute 7^{222} mod 11.

Since 11 is prime and 11 does not divide 7, Fermat's little theorem gives 7^{10} ≡ 1 (mod 11).

```
222 = 10 * 22 + 2
7^{222} = (7^{10})^{22} * 7^2 ≡ 1^{22} * 49 ≡ 49 mod 11 ≡ 5 (mod 11)
```

So 7^{222} mod 11 = 5.

---

## 7. The RSA Cryptosystem

RSA (Rivest-Shamir-Adleman) is a public-key cryptosystem whose security rests on the difficulty of factoring large integers.

### Key Generation

1. Choose two large distinct primes p and q (each typically 1024 bits or more).
2. Compute n = p * q (the modulus, ~2048 bits).
3. Compute phi(n) = (p - 1)(q - 1).
4. Choose the public exponent e such that:
   - 1 < e < phi(n)
   - gcd(e, phi(n)) = 1
   - Common choice: e = 65537 = 2^16 + 1 (efficient for repeated squaring, and usually coprime to phi(n))
5. Compute the private exponent d = e^{-1} mod phi(n) using the Extended Euclidean Algorithm.
6. **Public key**: (n, e). Published openly.
7. **Private key**: d. Kept secret. Also discard p, q, and phi(n) after computing d.

### Encryption

To encrypt a message m (an integer with 0 <= m < n):

```
c = m^e mod n
```

### Decryption

To decrypt ciphertext c:

```
m = c^d mod n
```

### Correctness Proof

**Claim:** Decryption recovers the original message: c^d ≡ m (mod n).

**Proof.** We have c^d = (m^e)^d = m^{ed} (mod n).

Since ed ≡ 1 (mod phi(n)), we can write ed = 1 + k * phi(n) for some non-negative integer k.

**Case 1: gcd(m, n) = 1.** By Euler's theorem:

```
m^{ed} = m^{1 + k*phi(n)} = m * (m^{phi(n)})^k ≡ m * 1^k = m (mod n)
```

**Case 2: gcd(m, n) != 1.** Since n = pq with p, q distinct primes, and 0 <= m < n, if gcd(m, n) != 1, then either p | m or q | m (but not both, since that would mean n | m, contradicting m < n).

Without loss of generality, assume p | m and gcd(m, q) = 1.

By Fermat's little theorem: m^{q-1} ≡ 1 (mod q), so:
```
m^{ed} = m^{1 + k(p-1)(q-1)} = m * (m^{q-1})^{k(p-1)} ≡ m * 1 = m (mod q)
```

Since p | m, we also have m^{ed} ≡ 0 ≡ m (mod p).

By the Chinese Remainder Theorem (since gcd(p, q) = 1):
```
m^{ed} ≡ m (mod pq)    i.e.,    m^{ed} ≡ m (mod n)
```

Therefore c^d ≡ m (mod n) in all cases. QED.

### Worked Example with Actual Numbers

**Key Generation:**

1. Choose primes: p = 61, q = 53.
2. Compute n = 61 * 53 = 3233.
3. Compute phi(n) = (61 - 1)(53 - 1) = 60 * 52 = 3120.
4. Choose e = 17 (verify: gcd(17, 3120) = 1, since 17 is prime and 17 does not divide 3120).
5. Compute d = 17^{-1} mod 3120:

   Apply Extended Euclidean Algorithm:
   ```
   3120 = 183 * 17 + 9     -->  9 = 3120 - 183 * 17
   17   = 1 * 9 + 8        -->  8 = 17 - 1 * 9
   9    = 1 * 8 + 1        -->  1 = 9 - 1 * 8
   8    = 8 * 1 + 0
   ```

   Back-substitution:
   ```
   1 = 9 - 1 * 8
     = 9 - 1 * (17 - 1 * 9)
     = 2 * 9 - 1 * 17
     = 2 * (3120 - 183 * 17) - 1 * 17
     = 2 * 3120 - 367 * 17
   ```

   So d ≡ -367 ≡ 3120 - 367 = 2753 (mod 3120).

   Verification: 17 * 2753 = 46801 = 15 * 3120 + 1 ≡ 1 (mod 3120). Confirmed.

6. **Public key**: (n, e) = (3233, 17).
7. **Private key**: d = 2753.

**Encryption:** Encrypt message m = 65.

```
c = 65^17 mod 3233
```

Using repeated squaring:
```
65^1  = 65                          mod 3233 = 65
65^2  = 4225                        mod 3233 = 992
65^4  = 992^2 = 984064              mod 3233 = 984064 mod 3233 = 2149
65^8  = 2149^2 = 4618201            mod 3233 = 4618201 mod 3233 = 2452
65^16 = 2452^2 = 6012304            mod 3233 = 6012304 mod 3233 = 2195
```

```
65^17 = 65^16 * 65^1
      = 2195 * 65 mod 3233
      = 142675 mod 3233
      = 2790
```

Verify: 142675 / 3233 = 44.12..., 44 * 3233 = 142252, 142675 - 142252 = 423. Let me recompute more carefully.

Actually, let us compute step by step:
```
65^2 mod 3233: 65 * 65 = 4225. 4225 - 3233 = 992. So 65^2 ≡ 992.
992^2 mod 3233: 992 * 992 = 984064. 984064 / 3233 ≈ 304.4. 304 * 3233 = 982832. 984064 - 982832 = 1232. So 65^4 ≡ 1232.
1232^2 mod 3233: 1232 * 1232 = 1517824. 1517824 / 3233 ≈ 469.4. 469 * 3233 = 1516277. 1517824 - 1516277 = 1547. So 65^8 ≡ 1547.
1547^2 mod 3233: 1547 * 1547 = 2393209. 2393209 / 3233 ≈ 740.2. 740 * 3233 = 2392420. 2393209 - 2392420 = 789. So 65^16 ≡ 789.
```

```
65^17 = 65^16 * 65 mod 3233
      = 789 * 65 mod 3233
      = 51285 mod 3233
      = 51285 - 15 * 3233
      = 51285 - 48495
      = 2790
```

So c = 2790.

**Decryption:** Decrypt c = 2790.

```
m = 2790^2753 mod 3233
```

By Euler's theorem and the construction of d, this returns m = 65. In practice, this is computed via repeated squaring using the binary representation of 2753.

We can verify correctness: ed = 17 * 2753 = 46801 = 1 + 15 * 3120 = 1 + 15 * phi(n), so m^{ed} = m * (m^{phi(n)})^15 ≡ m * 1 ≡ m (mod n).

---

## 8. Security of RSA

**Why RSA is hard to break:**
- To compute d from the public key (n, e), an attacker needs phi(n) = (p-1)(q-1).
- Computing phi(n) requires knowing the factorization of n = pq.
- Factoring large integers is believed to be computationally intractable (no known polynomial-time classical algorithm).

**Parameter recommendations (NIST):**
- Minimum key size: n should be at least 2048 bits (p and q each ~1024 bits).
- For long-term security (beyond 2030): use 3072-bit or 4096-bit keys.

**Common attacks and mitigations:**

| Attack | Description | Mitigation |
|---|---|---|
| Small e with small m | If m^e < n, ciphertext reveals m directly | Use padding (OAEP) |
| Common modulus | Reusing n with different e values | Each user generates own n |
| Timing attacks | Measuring decryption time | Constant-time implementation |
| Chosen ciphertext | Exploiting malleability of RSA | Use OAEP padding |
| Factoring advances | Better algorithms for factoring | Use sufficiently large keys |

---

## 9. Chinese Remainder Theorem

**Theorem.** If gcd(m1, m2) = 1, then for any integers a1, a2, the system:

```
x ≡ a1 (mod m1)
x ≡ a2 (mod m2)
```

has a unique solution modulo m1 * m2.

**Construction.** Let M1 = m2, M2 = m1. Find y1 = M1^{-1} mod m1 and y2 = M2^{-1} mod m2. Then:

```
x = a1 * M1 * y1 + a2 * M2 * y2 (mod m1 * m2)
```

**Worked Example:** Solve x ≡ 2 (mod 3), x ≡ 3 (mod 5).

M1 = 5, M2 = 3. y1 = 5^{-1} mod 3 = 2 (since 5 * 2 = 10 ≡ 1 mod 3). y2 = 3^{-1} mod 5 = 2 (since 3 * 2 = 6 ≡ 1 mod 5).

```
x = 2 * 5 * 2 + 3 * 3 * 2 = 20 + 18 = 38 ≡ 8 (mod 15)
```

Check: 8 mod 3 = 2. 8 mod 5 = 3. Verified.

**Application to RSA:** CRT allows faster decryption. Instead of computing m = c^d mod n directly, compute:

```
m_p = c^(d mod (p-1)) mod p
m_q = c^(d mod (q-1)) mod q
```

Then combine via CRT. This is roughly 4 times faster since the moduli are half the bit length of n.

---

## 10. Summary of Key Identities

| Result | Statement | Conditions |
|---|---|---|
| Division algorithm | a = qd + r, 0 <= r < d | d > 0 |
| Bezout's identity | gcd(a,b) = sa + tb | a, b not both 0 |
| Euler's theorem | a^{phi(n)} ≡ 1 (mod n) | gcd(a, n) = 1 |
| Fermat's little theorem | a^{p-1} ≡ 1 (mod p) | p prime, p does not divide a |
| CRT | Unique x mod m1*m2 exists | gcd(m1, m2) = 1 |
| RSA correctness | (m^e)^d ≡ m (mod n) | n = pq, ed ≡ 1 (mod phi(n)) |
| Multiplicativity of phi | phi(mn) = phi(m)*phi(n) | gcd(m, n) = 1 |
| phi of prime power | phi(p^k) = p^k - p^{k-1} | p prime |
