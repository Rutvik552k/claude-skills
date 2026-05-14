# Boundary Value Analysis and Equivalence Class Partitioning

Detailed worked examples, test case count comparisons, and templates for systematic application of boundary value and equivalence class testing techniques.

---

## 1. Boundary Value Analysis: Worked Example

### Problem: Commission Calculation Function

A sales commission function takes two inputs:

- **salesAmount**: integer in range [1, 100000] (dollars)
- **discountPercent**: integer in range [0, 50] (percent)

The function computes `commission = salesAmount * (100 - discountPercent) / 100 * commissionRate`.

### Step 1: Identify Boundaries for Each Variable

| Variable | min | min+1 | nominal | max-1 | max |
|---|---|---|---|---|---|
| salesAmount | 1 | 2 | 50000 | 99999 | 100000 |
| discountPercent | 0 | 1 | 25 | 49 | 50 |

### Step 2: Normal Boundary Value Test Cases (4n + 1 = 9 tests)

Hold all variables at nominal except the one being varied:

| Test ID | salesAmount | discountPercent | Notes |
|---|---|---|---|
| T1 | 1 | 25 | salesAmount at min |
| T2 | 2 | 25 | salesAmount at min+1 |
| T3 | 50000 | 25 | Both nominal (base test) |
| T4 | 99999 | 25 | salesAmount at max-1 |
| T5 | 100000 | 25 | salesAmount at max |
| T6 | 50000 | 0 | discountPercent at min |
| T7 | 50000 | 1 | discountPercent at min+1 |
| T8 | 50000 | 49 | discountPercent at max-1 |
| T9 | 50000 | 50 | discountPercent at max |

Note: T3 is the shared nominal test case, counted once. Total = 4(2) + 1 = 9.

### Step 3: Robust Boundary Value Test Cases (6n + 1 = 13 tests)

Add values just outside the valid boundaries:

| Test ID | salesAmount | discountPercent | Notes | Expected |
|---|---|---|---|---|
| T10 | 0 | 25 | salesAmount below min | Error/reject |
| T11 | 100001 | 25 | salesAmount above max | Error/reject |
| T12 | 50000 | -1 | discountPercent below min | Error/reject |
| T13 | 50000 | 51 | discountPercent above max | Error/reject |

Total = 9 normal + 4 robust = 13 = 6(2) + 1.

### Step 4: Worst-Case Boundary Value Test Cases (5^n = 25 tests)

Take the Cartesian product of the 5 boundary values for each variable:

| salesAmount values | discountPercent values |
|---|---|
| {1, 2, 50000, 99999, 100000} | {0, 1, 25, 49, 50} |

This produces 5 x 5 = 25 test cases. A selection:

| Test ID | salesAmount | discountPercent |
|---|---|---|
| W1 | 1 | 0 |
| W2 | 1 | 1 |
| W3 | 1 | 25 |
| W4 | 1 | 49 |
| W5 | 1 | 50 |
| W6 | 2 | 0 |
| W7 | 2 | 1 |
| ... | ... | ... |
| W25 | 100000 | 50 |

### Step 5: Robust Worst-Case (7^n = 49 tests)

Add the out-of-boundary values to the Cartesian product:

| salesAmount values | discountPercent values |
|---|---|
| {0, 1, 2, 50000, 99999, 100000, 100001} | {-1, 0, 1, 25, 49, 50, 51} |

This produces 7 x 7 = 49 test cases.

---

## 2. Equivalence Class Partitioning: Worked Example

### Problem: Date Validation Function

A function `isValidDate(year, month, day)` returns true if the date is valid. Constraints:

- **year**: 1900 to 2100 (inclusive)
- **month**: 1 to 12 (inclusive)
- **day**: depends on month and year (1-28, 1-29, 1-30, or 1-31)

### Step 1: Identify Equivalence Classes

#### Year Classes

| Class ID | Description | Type |
|---|---|---|
| Y1 | 1900 <= year <= 2100 | Valid |
| Y2 | year < 1900 | Invalid |
| Y3 | year > 2100 | Invalid |

#### Month Classes

| Class ID | Description | Type |
|---|---|---|
| M1 | month in {1, 3, 5, 7, 8, 10, 12} (31-day months) | Valid |
| M2 | month in {4, 6, 9, 11} (30-day months) | Valid |
| M3 | month = 2 (February) | Valid |
| M4 | month < 1 | Invalid |
| M5 | month > 12 | Invalid |

#### Day Classes (depend on month class)

| Class ID | Description | Condition | Type |
|---|---|---|---|
| D1 | 1 <= day <= 31 | month in M1 | Valid |
| D2 | 1 <= day <= 30 | month in M2 | Valid |
| D3 | 1 <= day <= 28 | month in M3, non-leap year | Valid |
| D4 | 1 <= day <= 29 | month in M3, leap year | Valid |
| D5 | day < 1 | any month | Invalid |
| D6 | day > max for month | depends on month | Invalid |

#### Leap Year Sub-Classes

| Class ID | Description | Type |
|---|---|---|
| L1 | year divisible by 4 but not by 100 | Leap year |
| L2 | year divisible by 400 | Leap year |
| L3 | year divisible by 100 but not by 400 | Not leap year |
| L4 | year not divisible by 4 | Not leap year |

### Step 2: Weak Normal Equivalence Class Tests

Select one representative from each valid class. Minimum test set:

| Test ID | year | month | day | Classes Covered | Expected |
|---|---|---|---|---|---|
| E1 | 2000 | 6 | 15 | Y1, M2, D2 | Valid |
| E2 | 1996 | 1 | 15 | Y1, M1, D1 | Valid |
| E3 | 2001 | 2 | 14 | Y1, M3, D3, L4 | Valid |
| E4 | 2000 | 2 | 29 | Y1, M3, D4, L2 | Valid |

### Step 3: Weak Robust Equivalence Class Tests

Add one test per invalid class:

| Test ID | year | month | day | Classes Covered | Expected |
|---|---|---|---|---|---|
| E5 | 1899 | 6 | 15 | Y2 | Invalid |
| E6 | 2101 | 6 | 15 | Y3 | Invalid |
| E7 | 2000 | 0 | 15 | M4 | Invalid |
| E8 | 2000 | 13 | 15 | M5 | Invalid |
| E9 | 2000 | 6 | 0 | D5 | Invalid |
| E10 | 2000 | 6 | 31 | D6 (30-day month) | Invalid |

### Step 4: Strong Normal Equivalence Class Tests

Cartesian product of all valid classes. For this example:

- Year: 1 valid class (Y1)
- Month: 3 valid classes (M1, M2, M3)
- Day: varies by month, but abstractly 4 valid classes (D1, D2, D3, D4)
- Leap year: 4 sub-classes (L1, L2, L3, L4) relevant only for February

Representative strong normal set:

| Test ID | year | month | day | Classes | Expected |
|---|---|---|---|---|---|
| S1 | 1996 | 1 | 15 | Y1, M1, D1, L1 | Valid |
| S2 | 2000 | 1 | 31 | Y1, M1, D1, L2 | Valid |
| S3 | 1900 | 1 | 15 | Y1, M1, D1, L3 | Valid |
| S4 | 2001 | 1 | 15 | Y1, M1, D1, L4 | Valid |
| S5 | 1996 | 4 | 15 | Y1, M2, D2, L1 | Valid |
| S6 | 2000 | 4 | 30 | Y1, M2, D2, L2 | Valid |
| S7 | 1900 | 9 | 15 | Y1, M2, D2, L3 | Valid |
| S8 | 2001 | 11 | 15 | Y1, M2, D2, L4 | Valid |
| S9 | 1996 | 2 | 29 | Y1, M3, D4, L1 | Valid |
| S10 | 2000 | 2 | 29 | Y1, M3, D4, L2 | Valid |
| S11 | 1900 | 2 | 28 | Y1, M3, D3, L3 | Valid |
| S12 | 2001 | 2 | 14 | Y1, M3, D3, L4 | Valid |

### Step 5: Strong Robust Equivalence Class Tests

Include cross-products with invalid classes. Add tests such as:

| Test ID | year | month | day | Invalid Class | Expected |
|---|---|---|---|---|---|
| R1 | 1899 | 0 | 0 | Y2, M4, D5 | Invalid (multiple) |
| R2 | 2101 | 13 | 32 | Y3, M5, D6 | Invalid (multiple) |
| R3 | 1899 | 6 | 31 | Y2, D6 | Invalid |
| R4 | 2000 | 2 | 30 | D6 (Feb max=29) | Invalid |
| R5 | 1900 | 2 | 29 | D6 (non-leap Feb max=28) | Invalid |

---

## 3. Test Case Count Comparison Table

| Technique | Formula | n=1 | n=2 | n=3 | n=4 | n=5 |
|---|---|---|---|---|---|---|
| Normal BVA | 4n + 1 | 5 | 9 | 13 | 17 | 21 |
| Robust BVA | 6n + 1 | 7 | 13 | 19 | 25 | 31 |
| Worst-Case BVA | 5^n | 5 | 25 | 125 | 625 | 3125 |
| Robust Worst-Case | 7^n | 7 | 49 | 343 | 2401 | 16807 |
| Weak Normal EC | max(classes per var) | varies | varies | varies | varies | varies |
| Strong Normal EC | product of class counts | varies | varies | varies | varies | varies |

### Growth Rates and Practical Implications

- Normal and Robust BVA grow **linearly** with n: practical for any number of variables.
- Worst-Case BVA grows **exponentially**: practical only for n <= 4 or so.
- Robust Worst-Case is even larger: use only for critical, high-risk functions with few variables.
- Equivalence class test counts depend on the number of classes identified per variable, not on variable count alone.

### Decision Guide: When to Use Which

| Situation | Recommended Technique | Rationale |
|---|---|---|
| Independent numeric variables, moderate risk | Normal BVA | Best cost-benefit ratio; faults cluster at boundaries |
| Need error handling verification | Robust BVA | Tests just-out-of-range values trigger error paths |
| Variables interact, critical function | Worst-Case BVA | Tests all boundary combinations; catches interaction faults |
| Categorical or enumerated inputs | Equivalence Class | BVA doesn't apply well to non-numeric types |
| Mixed numeric and categorical | BVA + Equivalence Class | Apply BVA to numeric, EC to categorical inputs |
| High-risk with few variables (n<=3) | Robust Worst-Case | Maximum thoroughness is affordable |

---

## 4. Templates

### Template A: Boundary Value Analysis Worksheet

```
Function Under Test: _______________
Number of Variables: n = ___

Variable 1: _______________
  Type: ___  Range: [___, ___]
  min = ___  min+1 = ___  nominal = ___  max-1 = ___  max = ___

Variable 2: _______________
  Type: ___  Range: [___, ___]
  min = ___  min+1 = ___  nominal = ___  max-1 = ___  max = ___

(Repeat for each variable)

Normal BVA Test Count: 4n + 1 = ___
Robust BVA Test Count: 6n + 1 = ___
Worst-Case BVA Test Count: 5^n = ___

Strategy Selected: [ ] Normal  [ ] Robust  [ ] Worst-Case  [ ] Robust Worst-Case
Justification: _______________

Test Cases:
| ID | Var1 | Var2 | ... | Expected Output | Actual Output | Pass/Fail |
|----|------|------|-----|-----------------|---------------|-----------|
|    |      |      |     |                 |               |           |
```

### Template B: Equivalence Class Partitioning Worksheet

```
Function Under Test: _______________

Variable 1: _______________
  Valid Classes:
    VC1: _______________  Representative value: ___
    VC2: _______________  Representative value: ___
  Invalid Classes:
    IC1: _______________  Representative value: ___
    IC2: _______________  Representative value: ___

Variable 2: _______________
  Valid Classes:
    VC3: _______________  Representative value: ___
  Invalid Classes:
    IC3: _______________  Representative value: ___

(Repeat for each variable)

Testing Level:
  [ ] Weak Normal:   ___ tests (= max valid classes per variable)
  [ ] Strong Normal: ___ tests (= product of valid class counts)
  [ ] Weak Robust:   ___ tests (= weak normal + one per invalid class)
  [ ] Strong Robust: ___ tests (= product of all classes)

Test Cases:
| ID | Var1 Class | Var1 Value | Var2 Class | Var2 Value | Expected | Actual | P/F |
|----|------------|------------|------------|------------|----------|--------|-----|
|    |            |            |            |            |          |        |     |
```

### Template C: Combined BVA + EC Checklist

```
1. Identify all input variables
2. For each variable:
   a. Determine type (numeric continuous, numeric discrete, enumerated, boolean)
   b. For numeric: identify boundaries, apply BVA
   c. For categorical: identify equivalence classes
   d. For boolean: include true and false
3. Determine variable independence:
   a. Independent -> Normal BVA is sufficient
   b. Interacting -> Consider worst-case BVA for interacting subset
4. Identify output-based equivalence classes:
   a. What distinct output categories exist?
   b. Map input combinations to output classes
5. Add special values: zero, empty, null, max_int, negative, boundary powers
6. Consolidate: remove duplicate test cases across techniques
7. Prioritize: boundary tests first, then equivalence classes, then special values
```

---

## 5. Additional Worked Example: Triangle Classification

### Problem

`classifyTriangle(a, b, c)` where a, b, c are side lengths (integers 1-200). Output: equilateral, isosceles, scalene, or not-a-triangle.

### BVA (Normal, n=3): 4(3) + 1 = 13 tests

| ID | a | b | c | Notes |
|---|---|---|---|---|
| 1 | 1 | 100 | 100 | a at min |
| 2 | 2 | 100 | 100 | a at min+1 |
| 3 | 100 | 100 | 100 | all nominal |
| 4 | 199 | 100 | 100 | a at max-1 |
| 5 | 200 | 100 | 100 | a at max |
| 6 | 100 | 1 | 100 | b at min |
| 7 | 100 | 2 | 100 | b at min+1 |
| 8 | 100 | 199 | 100 | b at max-1 |
| 9 | 100 | 200 | 100 | b at max |
| 10 | 100 | 100 | 1 | c at min |
| 11 | 100 | 100 | 2 | c at min+1 |
| 12 | 100 | 100 | 199 | c at max-1 |
| 13 | 100 | 100 | 200 | c at max |

### Equivalence Classes for Triangle

| Class | Description | Example |
|---|---|---|
| EC1 | Equilateral (a=b=c) | (50, 50, 50) |
| EC2 | Isosceles (exactly two equal) | (50, 50, 30) |
| EC3 | Scalene (all different, valid) | (30, 40, 50) |
| EC4 | Not-a-triangle (a+b <= c) | (1, 2, 100) |
| EC5 | Not-a-triangle (a+c <= b) | (1, 100, 2) |
| EC6 | Not-a-triangle (b+c <= a) | (100, 1, 2) |
| EC7 | Invalid input (a < 1) | (0, 50, 50) |
| EC8 | Invalid input (a > 200) | (201, 50, 50) |

Note how BVA alone misses the triangle inequality entirely -- this is why combining BVA with equivalence class partitioning is essential. BVA tests boundaries of individual variable ranges; equivalence classes capture relationships between variables.

---

## 6. Output-Based Equivalence Classes

An often-overlooked technique: partition by expected output rather than input.

### Example: Tax Bracket Calculator

```
Input: income (0 to 1,000,000)
Output classes:
  O1: 0% bracket    (income 0-10000)
  O2: 10% bracket   (income 10001-40000)
  O3: 20% bracket   (income 40001-85000)
  O4: 30% bracket   (income 85001-160000)
  O5: 35% bracket   (income 160001-1000000)
```

This gives 5 valid output classes. For each, select:
- A value well inside the class (nominal)
- Values at the boundaries between classes

Combined test set:

| ID | income | Output Class | Boundary? |
|---|---|---|---|
| 1 | 5000 | O1 (0%) | No (nominal) |
| 2 | 10000 | O1 (0%) | Yes (upper boundary) |
| 3 | 10001 | O2 (10%) | Yes (lower boundary) |
| 4 | 25000 | O2 (10%) | No (nominal) |
| 5 | 40000 | O2 (10%) | Yes (upper boundary) |
| 6 | 40001 | O3 (20%) | Yes (lower boundary) |
| 7 | 60000 | O3 (20%) | No (nominal) |
| 8 | 85000 | O3 (20%) | Yes (upper boundary) |
| 9 | 85001 | O4 (30%) | Yes (lower boundary) |
| 10 | 120000 | O4 (30%) | No (nominal) |
| 11 | 160000 | O4 (30%) | Yes (upper boundary) |
| 12 | 160001 | O5 (35%) | Yes (lower boundary) |
| 13 | 500000 | O5 (35%) | No (nominal) |

This approach naturally combines BVA (boundary values at class transitions) with equivalence class partitioning (one nominal value per class).
