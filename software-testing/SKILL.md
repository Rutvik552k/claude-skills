---
name: software-testing
description: Systematic software testing methodology covering boundary value, equivalence class, decision table, path testing, data flow, integration, system, regression, and model-based testing.
origin: ECC
---

# Software Testing

Systematic software testing draws on Paul C. Jorgensen's *Software Testing: A Craftsman's Approach* to provide a structured, mathematically grounded methodology for designing test cases, measuring coverage, and choosing the right testing strategy for a given situation. Rather than treating testing as an afterthought, this skill treats it as a disciplined engineering activity with well-defined techniques, each suited to particular kinds of faults. The techniques span the full spectrum from unit-level functional testing (boundary value analysis, equivalence class partitioning, decision tables) through structural testing (path coverage, data flow analysis) to integration, system, regression, and model-based testing.

## When to Activate

- Designing test cases for a function, module, or system
- Choosing coverage criteria and evaluating whether existing tests are adequate
- Reviewing test adequacy for code under review or prior to release
- Planning integration testing strategy for a multi-module system
- Setting up or maintaining regression test suites
- Generating tests from models (finite state machines, statecharts, use cases)
- Evaluating tradeoffs between test thoroughness and test count
- Analyzing cyclomatic complexity or data flow to guide structural testing

## Topics Index

### Boundary Value Testing

- **Normal Boundary Value Testing**: test min, min+1, nominal, max-1, max for each variable while holding others at nominal; produces **4n + 1** test cases for n variables
- **Robust Boundary Value Testing**: extends normal BVA with min-1 and max+1 (just outside boundaries); produces **6n + 1** test cases
- **Worst-Case Boundary Value Testing**: Cartesian product of boundary values for all variables simultaneously; produces **5^n** test cases (exponential growth)
- **Robust Worst-Case Testing**: combines robust and worst-case; produces **7^n** test cases
- **Special Value Testing**: tester expertise drives selection of values known to cause failures (zero, empty string, null, powers of 2, leap year dates)
- **When to use**: functions with independent, continuous numeric input variables; most cost-effective when variables are truly independent

### Equivalence Class Testing

- **Weak Normal Equivalence Class Testing**: one test from each equivalence class; covers each class at least once
- **Strong Normal Equivalence Class Testing**: Cartesian product of all valid equivalence classes; tests all combinations
- **Weak Robust Equivalence Class Testing**: weak normal plus one test from each invalid class
- **Strong Robust Equivalence Class Testing**: Cartesian product including invalid classes; most thorough
- **Identifying Equivalence Classes**: partition input domain into classes where the program behaves identically; separate valid from invalid ranges; consider output-based partitioning
- **When to use**: inputs with clearly defined ranges, sets, or categories; complements BVA for non-numeric or categorical inputs

### Decision Table Testing

- **Structure**: conditions (inputs/states), actions (outputs/behaviors), and rules (columns mapping condition combinations to actions)
- **Limited-Entry Tables**: conditions are boolean (T/F/don't-care); each rule is a column of T/F/- values
- **Extended-Entry Tables**: conditions can have multiple values (enumerations); more expressive but larger
- **Table Collapse Rules**: merge rules that differ in only one condition and produce the same actions; use don't-care (-) entries
- **Test Case Generation**: one test case per rule; the number of rules before collapse is 2^n for n boolean conditions
- **When to use**: complex business logic with multiple interacting conditions; policy rules, pricing tiers, eligibility criteria

### Path Testing

- **Program Graphs**: nodes are statements or statement fragments; edges represent flow of control (sequential, branching, looping)
- **DD-Paths (Decision-to-Decision Paths)**: maximal chains of nodes with in-degree and out-degree of 1; compress the program graph into its essential branching structure
- **Cyclomatic Complexity V(G)**: McCabe's metric; **V(G) = e - n + 2p** where e = edges, n = nodes, p = connected components; equals the number of independent paths through the code; also equals the number of decision points + 1 (for a single-component graph)
- **Basis Paths**: a set of V(G) linearly independent paths that span the path space; every other path is a linear combination of basis paths
- **Statement Coverage**: every statement executed at least once (weakest structural criterion)
- **Branch Coverage (Decision Coverage)**: every branch (true/false) of every decision executed; subsumes statement coverage
- **Condition Coverage**: every individual boolean sub-expression evaluated to both true and false
- **Path Coverage**: every distinct path through the program executed (often infeasible due to loops; strongest structural criterion)
- **When to use**: white-box testing of control flow; identifying dead code or untested branches

### Data Flow Testing

- **Define-Use Pairs**: a variable is **defined** (d) at a node where it is assigned a value; it is **used** at a node where it is referenced, either in a **computation (c-use)** or a **predicate (p-use)**
- **DU-Paths**: a definition-clear path from a defining node to a using node for a given variable (no re-definition along the path)
- **All-Defs Coverage**: for every definition of every variable, at least one DU-path to some use is tested
- **All-Uses Coverage**: for every definition of every variable, at least one DU-path to every reachable use is tested
- **All-DU-Paths Coverage**: every DU-path from every definition to every use is tested (strongest data flow criterion)
- **Rapps-Weyuker Hierarchy**: all-paths subsumes all-DU-paths subsumes all-uses subsumes all-defs; all-uses subsumes all-p-uses and all-c-uses
- **When to use**: detecting faults related to incorrect variable definitions, uninitialized variables, or stale values; complements path testing

### Integration Testing

- **Decomposition-Based Strategies**:
  - **Top-Down**: start from the top module, replace called modules with **stubs** (dummy implementations); integrate downward layer by layer
  - **Bottom-Up**: start from leaf modules, replace calling modules with **drivers** (test harnesses); integrate upward
  - **Sandwich (Hybrid)**: top-down from top, bottom-up from bottom, meet in the middle
  - **Big-Bang**: integrate all modules at once (no incremental strategy; hard to isolate faults)
- **Call-Graph-Based Strategies**:
  - **Pairwise Integration**: test each caller-callee pair independently
  - **Neighborhood Integration**: test each node with all its immediate predecessors and successors in the call graph
- **Path-Based Integration**:
  - **MM-Paths (Method-to-Method Paths)**: sequences of method calls that represent end-to-end message passing through the system; derived from the call graph and method internal paths
- **When to use**: systems with multiple interacting modules, services, or classes; choose strategy based on module dependency structure, availability of stubs/drivers, and fault isolation needs

### System Testing

- **Functional Testing**:
  - **Business Process Testing**: end-to-end scenarios that mimic real user workflows across the system
  - **Use Case Testing**: derive test cases from use case steps, preconditions, postconditions, and alternate/exception flows
  - **State Transition Testing**: test all valid state transitions and attempt invalid transitions
- **Non-Functional Testing**:
  - **Performance Testing**: response time, throughput under expected load
  - **Stress Testing**: behavior beyond normal capacity; find breaking points
  - **Volume Testing**: large data sets, database capacity
  - **Security Testing**: authentication, authorization, injection, XSS, CSRF
  - **Usability Testing**: user experience, accessibility, learnability
- **Acceptance Testing**:
  - **Alpha Testing**: performed by internal users at the development site
  - **Beta Testing**: performed by external users in their own environment
  - **UAT (User Acceptance Testing)**: formal sign-off by the customer or business stakeholders
- **When to use**: validating the complete system against requirements and real-world conditions

### Object-Oriented Testing

- **Class Testing**: test each class in isolation; state-based testing verifies method sequences that transition object state correctly
- **State-Based Testing**: model object lifecycle as a state machine; test all state transitions, including invalid transitions
- **Polymorphism Testing**: verify correct behavior for each concrete type behind a polymorphic reference; test substitutability (Liskov)
- **Inheritance Testing**: test overridden methods in subclass context; verify inherited behavior is preserved or correctly modified
- **Interaction Testing**: test collaborations between objects; focus on message passing sequences
- **Design Pattern Testing**: patterns like Observer, Strategy, Factory have characteristic failure modes; test pattern contracts
- **When to use**: object-oriented codebases where state, polymorphism, and object interactions dominate fault risk

### Regression Testing

- **Retest-All**: re-run the entire test suite after every change (safe but expensive)
- **Selective Regression Testing**: analyze the change and run only tests affected by the modification (change impact analysis)
- **Test Prioritization**: order tests by fault detection probability, code coverage, or historical failure rate; run highest-priority tests first
- **Suite Minimization**: remove redundant tests that do not increase coverage; reduce suite size while maintaining coverage
- **CI/CD Integration**: automate regression suite execution on every commit or pull request; gate merges on test passage
- **When to use**: any codebase under active development; critical for maintaining confidence during refactoring, feature additions, and dependency updates

### Model-Based Testing

- **Finite State Machine (FSM) Testing**:
  - **Transition Coverage**: every transition exercised at least once
  - **W-Method**: Chow's method using a characterization set W to verify states and transitions; test suite length bounded by number of states and inputs
  - **UIO Sequences (Unique Input/Output)**: sequences that uniquely identify each state; shorter than W-method sequences when they exist
- **Statechart Testing**: hierarchical states (superstates/substates), concurrent regions (orthogonal components), history states; flatten or test hierarchically
- **Petri Net Testing**: places, transitions, tokens; coverage criteria include transition coverage, place coverage, and marking reachability
- **Use Case Models**: map use case steps to test steps; cover main success scenario, alternate flows, and exception flows
- **When to use**: systems with well-defined state behavior (protocols, UIs, embedded controllers); when a formal model exists or can be constructed

## Key Concepts

### Coverage Hierarchy

| Level | Criterion | Subsumes |
|---|---|---|
| 1 | Statement Coverage | (base level) |
| 2 | Branch Coverage | Statement |
| 3 | Condition Coverage | (independent of Branch) |
| 4 | Branch + Condition | Branch, Condition |
| 5 | Path Coverage | Branch + Condition |
| 6 | All-Defs | (base data flow level) |
| 7 | All-Uses | All-Defs |
| 8 | All-DU-Paths | All-Uses |

### Test Case Count Formulas

| Technique | Formula | Example (n=3, each var has 5 boundary values) |
|---|---|---|
| Normal BVA | 4n + 1 | 13 |
| Robust BVA | 6n + 1 | 19 |
| Worst-Case BVA | 5^n | 125 |
| Robust Worst-Case | 7^n | 343 |
| Decision Table (binary) | 2^n conditions | 8 rules (before collapse) |
| Cyclomatic Complexity | e - n + 2p | Depends on graph structure |

## Problem-Solving Patterns

| Pattern | When to Apply | Key Insight |
|---|---|---|
| Start with BVA | Numeric inputs with defined ranges | Faults cluster at boundaries; 4n+1 tests catch most boundary faults |
| Add equivalence classes | Inputs have distinct categories or types | Partition reduces infinite input space to finite representative set |
| Use decision tables | Multiple interacting boolean conditions | Makes implicit logic explicit; prevents missed combinations |
| Measure cyclomatic complexity | Assessing testability of a function | V(G) > 10 suggests the function is too complex; refactor before testing |
| Apply data flow testing | Suspect variable misuse or stale data | Define-use analysis catches faults that path testing misses |
| Choose integration order | Multi-module system design | Top-down finds interface faults early; bottom-up enables parallel development |
| Model the states | Protocol, UI, or stateful component | FSM model makes state-related faults visible and testable |
| Prioritize regression tests | Large suite, limited CI time budget | Historical failure rate and change proximity predict fault detection |
| Combine techniques | No single technique is sufficient | BVA + equivalence class + structural coverage gives complementary fault detection |

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---|---|---|
| Testing only happy paths | Bias toward expected behavior | Use robust BVA and invalid equivalence classes to force error paths |
| Ignoring variable interactions | Assuming independence when variables interact | Use worst-case BVA or strong equivalence class testing for interacting variables |
| 100% statement coverage = adequate | Statement coverage is the weakest criterion | Require branch coverage minimum; use data flow coverage for critical code |
| Exhaustive path testing | Loop paths make total path count infinite | Use basis paths (V(G) paths) as practical path coverage target |
| Big-bang integration | Appears faster than incremental | Fault isolation is nearly impossible; use incremental strategy instead |
| Skipping regression after refactoring | Refactoring is "safe" so tests seem unnecessary | Refactoring changes structure; regression ensures behavior is preserved |
| No model for stateful systems | "The code is the model" thinking | Build explicit FSM or statechart; it reveals missing transitions and unreachable states |
| Decision table explosion | Too many conditions (2^n rules) | Collapse rules with don't-care entries; split into sub-tables by concern |
| Testing getters and setters | Desire for high line coverage | Focus testing effort on complex logic, not trivial accessors |
| Treating all tests as equal priority | Flat test suites with no ordering | Prioritize by risk, change proximity, and historical failure data |

## Source Material

*Software Testing: A Craftsman's Approach*, 4th Edition, Paul C. Jorgensen, 2014. CRC Press. ISBN 978-1-4665-6068-0.
