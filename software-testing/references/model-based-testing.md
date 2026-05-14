# Model-Based Testing

Finite state machine test generation, statechart testing, Petri net coverage criteria, and use case to test case mapping.

---

## 1. Finite State Machine (FSM) Test Generation

### FSM Fundamentals

A Finite State Machine is defined as a 5-tuple (S, I, O, T, s0) where:
- **S** = finite set of states
- **I** = finite set of input symbols (events)
- **O** = finite set of output symbols (actions)
- **T** = transition function: S x I -> S x O
- **s0** = initial state

### Example FSM: Turnstile

```
States: {Locked, Unlocked}
Inputs: {coin, push}
Outputs: {unlock, lock, none, alarm}
Initial state: Locked

Transition Table:
| Current State | Input | Next State | Output  |
|---------------|-------|------------|---------|
| Locked        | coin  | Unlocked   | unlock  |
| Locked        | push  | Locked     | alarm   |
| Unlocked      | coin  | Unlocked   | none    |
| Unlocked      | push  | Locked     | lock    |
```

State diagram:

```
         coin/unlock
  Locked -----------> Unlocked
    ^   <-----------     |
    |    push/lock       |
    |                    |
    +--- push/alarm      +--- coin/none
    (self-loop)          (self-loop)
```

### Transition Coverage

The simplest FSM coverage criterion: every transition is exercised at least once.

**Test suite for transition coverage of turnstile**:

| Test | Sequence | Transitions Covered |
|---|---|---|
| T1 | coin, push | Locked->Unlocked (coin), Unlocked->Locked (push) |
| T2 | push | Locked->Locked (push) |
| T3 | coin, coin | Locked->Unlocked (coin), Unlocked->Unlocked (coin) |

All 4 transitions are covered by T1 + T2 + T3. A minimal set would be:
- T1 covers transitions 1 and 4
- T2 covers transition 2
- T3 covers transition 3 (given T1 already covers transition 1, we just need the Unlocked->Unlocked(coin) part)

Optimal: coin, push, coin, coin, push (single sequence covering all transitions).

### The W-Method (Chow's Method)

The W-method generates a test suite that verifies both state behavior and transition correctness for a completely specified, minimal, deterministic FSM.

#### Definitions

- **Characterization Set (W)**: a set of input sequences that distinguishes every pair of states in the FSM. For each pair of states (si, sj), there exists a sequence in W that produces different output when applied starting from si vs sj.

- **State Cover (S-cover, P)**: a set of input sequences that, starting from the initial state, reaches each state exactly once. Includes the empty sequence (for the initial state).

- **Transition Cover (T-cover)**: extends P by appending each input symbol to each sequence in P. This reaches every state and takes every possible input from that state.

#### W-Method Algorithm

1. Compute the characterization set W
2. Compute the state cover P
3. Compute the transition cover by appending each input to each element of P
4. Test suite = (P union transition-cover) concatenated with each sequence in W

#### W-Method Applied to Turnstile

**Step 1: Characterization Set W**

We need sequences that distinguish Locked from Unlocked:
- Input "push": from Locked produces "alarm", from Unlocked produces "lock"
- So W = {push}

**Step 2: State Cover P**

- Locked (initial): reached by empty sequence ε
- Unlocked: reached by "coin"
- P = {ε, coin}

**Step 3: Transition Cover**

Append each input to each element of P:
- ε + coin = coin
- ε + push = push
- coin + coin = coin.coin
- coin + push = coin.push

Transition cover = {coin, push, coin.coin, coin.push}

**Step 4: Test Suite**

Concatenate each element of (P union transition-cover) with each element of W:

| Prefix | + W element | Full Test | Purpose |
|---|---|---|---|
| ε | push | push | Verify initial state is Locked |
| coin | push | coin.push | Verify state after coin is Unlocked |
| coin | push | coin.push | Verify Locked->Unlocked transition |
| push | push | push.push | Verify Locked->Locked(push) transition |
| coin.coin | push | coin.coin.push | Verify Unlocked->Unlocked(coin) transition |
| coin.push | push | coin.push.push | Verify Unlocked->Locked(push) transition |

After removing duplicates, the test suite is:
- push
- coin.push
- push.push
- coin.coin.push
- coin.push.push

For each test, verify the output sequence matches the expected output from the FSM specification.

### UIO Sequences (Unique Input/Output)

A UIO sequence for a state s is an input sequence such that the resulting output sequence uniquely identifies state s -- no other state produces the same output for that input.

#### Finding UIO Sequences

For each state, search for short input sequences whose output is unique to that state.

**Turnstile UIO sequences**:

| State | UIO Sequence | Output | Unique? |
|---|---|---|---|
| Locked | push | alarm | Yes (Unlocked produces "lock" for push) |
| Unlocked | push | lock | Yes (Locked produces "alarm" for push) |

Both states have UIO sequences of length 1.

#### UIO-Based Test Generation

For each transition (si, input, sj, output):
1. Navigate to state si using the state cover
2. Apply the input and verify the output
3. Apply the UIO sequence for sj and verify the output (confirms we arrived at the correct target state)

| Transition | Navigate to si | Input/Output | UIO for sj | Expected UIO Output |
|---|---|---|---|---|
| Locked->Unlocked (coin) | ε | coin/unlock | push | lock |
| Locked->Locked (push) | ε | push/alarm | push | alarm |
| Unlocked->Unlocked (coin) | coin | coin/none | push | lock |
| Unlocked->Locked (push) | coin | push/lock | push | alarm |

Full test sequences:
1. coin.push (verify: unlock, lock)
2. push.push (verify: alarm, alarm)
3. coin.coin.push (verify: unlock, none, lock)
4. coin.push.push (verify: unlock, lock, alarm)

#### UIO vs W-Method Comparison

| Aspect | W-Method | UIO Method |
|---|---|---|
| Test suite size | O(pn^2|W|) | O(pn) where p=|inputs| |
| Completeness | Guaranteed for minimal FSMs | Not guaranteed (UIO may not exist for all states) |
| When UIO doesn't exist | N/A | Fall back to W-method for those states |
| Practical advantage | More thorough | Shorter test sequences |

---

## 2. Statechart Testing

### Statecharts vs Flat FSMs

Statecharts (Harel, 1987) extend FSMs with:
- **Hierarchical states**: superstates containing substates
- **Concurrent regions**: orthogonal components that execute in parallel
- **History states**: remember the last active substate when re-entering a superstate
- **Guards**: boolean conditions on transitions
- **Actions**: entry/exit/do actions on states; actions on transitions

### Example Statechart: Media Player

```
[MediaPlayer] (superstate)
  |
  +-- [Off]  (initial substate)
  |
  +-- [On] (superstate)
  |     |
  |     +-- [Stopped] (initial substate of On)
  |     |
  |     +-- [Playing] (superstate)
  |     |     |
  |     |     +-- [Normal] (initial substate of Playing)
  |     |     +-- [FastForward]
  |     |     +-- [Rewind]
  |     |
  |     +-- [Paused]
  |
  +-- H (history pseudo-state for On)
```

Transitions:
- Off -> On (powerOn): enters On at Stopped
- On -> Off (powerOff): exits whatever substate
- Stopped -> Playing/Normal (play)
- Playing -> Stopped (stop)
- Playing -> Paused (pause)
- Paused -> Playing/H (resume): returns to last Playing substate via history
- Playing/Normal -> Playing/FastForward (ffwd)
- Playing/Normal -> Playing/Rewind (rwd)
- Playing/FastForward -> Playing/Normal (normal)
- Playing/Rewind -> Playing/Normal (normal)

### Statechart Testing Strategies

#### Strategy 1: Flatten and Test as FSM

Convert the statechart to an equivalent flat FSM by enumerating all concrete state configurations:

| Flat State | Composition |
|---|---|
| S1 | Off |
| S2 | On.Stopped |
| S3 | On.Playing.Normal |
| S4 | On.Playing.FastForward |
| S5 | On.Playing.Rewind |
| S6 | On.Paused |

Then enumerate all transitions and apply FSM test generation methods (W-method, UIO).

**Problem**: state explosion with concurrent regions. If region A has m states and region B has n states, the flat FSM has m*n states.

#### Strategy 2: Hierarchical Testing (Recommended)

Test at each level of the hierarchy independently:

**Level 1 (Top)**: Test MediaPlayer as {Off, On} with transitions powerOn and powerOff.

**Level 2 (On superstate)**: Test On as {Stopped, Playing, Paused} with play, stop, pause, resume.

**Level 3 (Playing superstate)**: Test Playing as {Normal, FastForward, Rewind} with ffwd, rwd, normal.

**Integration**: Test transitions that cross hierarchy boundaries (e.g., powerOff from any substate of On).

#### Strategy 3: Test Concurrent Regions Independently

For statecharts with orthogonal regions, test each region independently, then test synchronization points (events that affect multiple regions simultaneously).

### Statechart-Specific Test Obligations

| Obligation | Description | Example |
|---|---|---|
| All states visited | Every state (including substates) is entered at least once | Enter FastForward, Rewind, Paused |
| All transitions covered | Every transition is exercised | Cover all 10+ transitions |
| Entry/exit actions | Verify actions fire on state entry and exit | Verify "stop playback" on exit from Playing |
| History state | Verify correct substate is restored after history transition | Pause from FastForward, resume should return to FastForward |
| Guard conditions | Test both true and false for each guard | If transition has [battery > 10%], test with 11% and 9% |
| Default entry | Verify initial substates are entered correctly | Entering On goes to Stopped, entering Playing goes to Normal |

### History State Test Cases

| Test | Sequence | Expected Final State |
|---|---|---|
| H1 | powerOn, play, ffwd, pause, resume | On.Playing.FastForward (history restored) |
| H2 | powerOn, play, rwd, pause, resume | On.Playing.Rewind (history restored) |
| H3 | powerOn, play, pause, resume | On.Playing.Normal (history = Normal) |

---

## 3. Petri Net Coverage Criteria

### Petri Net Fundamentals

A Petri net is a 4-tuple (P, T, F, M0) where:
- **P** = finite set of places (circles)
- **T** = finite set of transitions (bars/rectangles)
- **F** = flow relation (arcs connecting places to transitions and transitions to places)
- **M0** = initial marking (distribution of tokens across places)

### Example: Producer-Consumer

```
Places: {Buffer_Empty, Buffer_Full, Producer_Ready, Consumer_Ready, Producer_Done, Consumer_Done}

Transitions:
  t1: Produce    (inputs: Producer_Ready, Buffer_Empty)
                 (outputs: Producer_Done, Buffer_Full)

  t2: Reset_Producer  (inputs: Producer_Done)
                      (outputs: Producer_Ready)

  t3: Consume    (inputs: Consumer_Ready, Buffer_Full)
                 (outputs: Consumer_Done, Buffer_Empty)

  t4: Reset_Consumer  (inputs: Consumer_Done)
                      (outputs: Consumer_Ready)

Initial marking M0:
  Producer_Ready: 1 token
  Consumer_Ready: 1 token
  Buffer_Empty: 1 token
  All others: 0 tokens
```

### Coverage Criteria for Petri Nets

#### Transition Coverage

Every transition fires at least once.

| Step | Firing | Tokens Before | Tokens After |
|---|---|---|---|
| 1 | t1 (Produce) | PR=1, BE=1 | PD=1, BF=1 |
| 2 | t3 (Consume) | CR=1, BF=1 | CD=1, BE=1 |
| 3 | t2 (Reset_Producer) | PD=1 | PR=1 |
| 4 | t4 (Reset_Consumer) | CD=1 | CR=1 |

All 4 transitions fired. Test sequence: t1, t3, t2, t4.

#### Place Coverage

Every place contains at least one token during the test execution.

Check from the trace above:
- Producer_Ready: has token at M0 (covered)
- Buffer_Empty: has token at M0 (covered)
- Consumer_Ready: has token at M0 (covered)
- Producer_Done: has token after step 1 (covered)
- Buffer_Full: has token after step 1 (covered)
- Consumer_Done: has token after step 2 (covered)

All places covered by the same test sequence.

#### Marking Reachability Coverage

Test that specific markings (token distributions) are reachable.

| Marking | Description | Reachable via |
|---|---|---|
| M0 | Initial state | Start |
| M1 | After produce | t1 |
| M2 | After produce + consume | t1, t3 |
| M3 | Back to initial | t1, t3, t2, t4 |

#### Conflict Coverage

When multiple transitions are enabled simultaneously and share input places (conflict), test each resolution.

In the producer-consumer example, t1 and other transitions may be in conflict if they compete for the same token in a shared place.

### Petri Net Testing of Concurrent Systems

| Property | Test Approach |
|---|---|
| Deadlock freedom | Verify that from every reachable marking, at least one transition is enabled |
| Liveness | Verify that every transition can fire from every reachable marking (eventually) |
| Boundedness | Verify that no place ever accumulates more than k tokens |
| Mutual exclusion | Verify that conflicting places never both have tokens simultaneously |
| Fairness | Verify that no transition is starved (always passed over in conflict resolution) |

### Practical Testing from Petri Nets

1. Enumerate reachable markings (build reachability graph)
2. Treat reachability graph as an FSM (markings = states, firings = transitions)
3. Apply FSM test generation (transition coverage, W-method) to the reachability graph
4. Map each abstract test (sequence of firings) to concrete test actions in the system

---

## 4. Use Case to Test Case Mapping

### Use Case Structure

A well-formed use case includes:
- **Preconditions**: what must be true before the use case begins
- **Main Success Scenario (MSS)**: the happy path, numbered steps
- **Extensions (Alternate Flows)**: branches at specific steps in the MSS
- **Exception Flows**: error conditions at specific steps
- **Postconditions**: what must be true after the use case completes

### Example Use Case: Online Purchase

```
Use Case: Purchase Item
Actor: Customer
Preconditions: Customer is logged in, item is in catalog

Main Success Scenario:
  1. Customer searches for item
  2. System displays search results
  3. Customer selects item
  4. System displays item details and price
  5. Customer adds item to cart
  6. System updates cart
  7. Customer proceeds to checkout
  8. System displays order summary
  9. Customer enters payment information
  10. System validates payment
  11. System processes order
  12. System sends confirmation email
  13. System displays order confirmation

Extensions:
  2a. No results found:
    2a1. System displays "no results" message
    2a2. Use case ends

  5a. Item out of stock:
    5a1. System displays "out of stock" message
    5a2. System offers waitlist option
    5a3. Use case returns to step 1

  10a. Payment validation fails:
    10a1. System displays error message
    10a2. Use case returns to step 9

  10b. Payment declined:
    10b1. System displays "payment declined"
    10b2. Customer may retry (return to step 9) or cancel
    10b3. If cancel, use case ends

Postconditions:
  Success: Order created, payment charged, confirmation sent
  Failure: No order created, no payment charged
```

### Mapping to Test Cases

#### Step 1: Identify Test Scenarios

Each path through the use case (MSS + each extension) becomes a test scenario.

| Scenario ID | Path | Description |
|---|---|---|
| SC1 | MSS (steps 1-13) | Happy path: successful purchase |
| SC2 | Steps 1-2, then 2a | Search returns no results |
| SC3 | Steps 1-5, then 5a | Item out of stock |
| SC4 | Steps 1-10, then 10a, then 9-13 | Payment invalid, retry succeeds |
| SC5 | Steps 1-10, then 10b, retry, 9-13 | Payment declined, retry succeeds |
| SC6 | Steps 1-10, then 10b, cancel | Payment declined, customer cancels |

#### Step 2: Define Test Cases from Scenarios

**Test Case TC1: Successful Purchase**

| Field | Value |
|---|---|
| Preconditions | Customer logged in; item "Widget-X" exists in catalog, in stock |
| Steps | 1. Search for "Widget-X" 2. Verify results contain Widget-X 3. Select Widget-X 4. Verify details and price ($29.99) 5. Click "Add to Cart" 6. Verify cart shows Widget-X 7. Click "Checkout" 8. Verify order summary shows Widget-X, $29.99 9. Enter valid credit card 10-11. Submit order 12-13. Verify confirmation page and email |
| Expected Result | Order created, $29.99 charged, confirmation email received |
| Postcondition Check | Order exists in system, payment recorded |

**Test Case TC2: No Search Results**

| Field | Value |
|---|---|
| Preconditions | Customer logged in; item "Nonexistent-Item" not in catalog |
| Steps | 1. Search for "Nonexistent-Item" 2. Verify "no results found" message |
| Expected Result | No results displayed, informative message shown |
| Postcondition Check | No cart changes, no order created |

**Test Case TC3: Out of Stock**

| Field | Value |
|---|---|
| Preconditions | Customer logged in; item "Widget-X" in catalog but stock = 0 |
| Steps | 1. Search for "Widget-X" 2-3. Select Widget-X 4. Verify details 5. Click "Add to Cart" 5a. Verify "out of stock" message and waitlist option |
| Expected Result | Item not added to cart, waitlist offered |
| Postcondition Check | Cart unchanged |

**Test Case TC4: Payment Invalid, Retry Succeeds**

| Field | Value |
|---|---|
| Preconditions | Customer logged in; item in stock; first payment info is invalid |
| Steps | 1-8. Normal flow to checkout 9. Enter invalid card number 10. Submit -> verify error message 9'. Enter valid card number 10'-13. Submit -> verify success |
| Expected Result | First attempt fails gracefully; second attempt succeeds |
| Postcondition Check | Order created with second payment method |

#### Step 3: Add Boundary and Edge Cases

Beyond the use case flows, add test cases for boundaries within steps:

| Test | Step | Boundary |
|---|---|---|
| TC7 | Step 1 | Search with empty string |
| TC8 | Step 1 | Search with special characters |
| TC9 | Step 5 | Add same item twice (quantity handling) |
| TC10 | Step 5 | Add item when cart is at maximum capacity |
| TC11 | Step 9 | Expired credit card |
| TC12 | Step 9 | Card number with wrong length |
| TC13 | Step 10 | Payment timeout (network issue) |

### Use Case Coverage Matrix

Track which test cases cover which use case elements:

| Test Case | MSS Steps | Extensions | Preconditions | Postconditions |
|---|---|---|---|---|
| TC1 | 1-13 | - | Logged in, in stock | Order, payment, email |
| TC2 | 1-2 | 2a | Logged in, not in catalog | No order |
| TC3 | 1-5 | 5a | Logged in, out of stock | No cart change |
| TC4 | 1-13 | 10a | Logged in, in stock | Order (retry) |
| TC5 | 1-10 | 10b (retry) | Logged in, in stock | Order (retry) |
| TC6 | 1-10 | 10b (cancel) | Logged in, in stock | No order |

### Coverage Criteria for Use Cases

| Criterion | Requirement | Test Count |
|---|---|---|
| MSS Coverage | Execute the main success scenario | 1 |
| All Extensions | Execute each extension at least once | 1 per extension |
| All Combinations | All pairs of extensions that could co-occur | Combinatorial |
| All Loops | Repeat steps that loop (e.g., retry payment 0, 1, 2, max times) | 3-4 per loop |

---

## 5. Model-Based Testing Process Summary

### End-to-End Workflow

```
1. Build the Model
   - Identify system states, events, actions
   - Choose formalism: FSM (simple), Statechart (hierarchical), Petri net (concurrent)
   - Validate model against requirements

2. Verify the Model
   - Check for completeness (all states reachable, no dead states)
   - Check for consistency (no conflicting transitions)
   - Check for determinism (each state+input has exactly one transition)

3. Select Coverage Criterion
   - Transition coverage (minimum)
   - State coverage
   - W-method or UIO (for FSMs)
   - Hierarchical coverage (for statecharts)
   - Marking reachability (for Petri nets)

4. Generate Test Sequences
   - Apply the chosen algorithm
   - Map abstract events to concrete test inputs
   - Map expected abstract outputs to concrete assertions

5. Execute Tests
   - Run test sequences against the system under test
   - Compare actual outputs with expected outputs

6. Analyze Results
   - Failed tests may indicate implementation faults or model errors
   - Update model if requirements have changed
   - Regenerate tests when the model changes
```

### Model Selection Guide

| System Characteristic | Recommended Model | Rationale |
|---|---|---|
| Sequential states, no hierarchy | FSM | Simple, well-supported tools |
| Hierarchical states, modes | Statechart | Manages complexity via nesting |
| Concurrent subsystems | Petri net or Statechart with regions | Models true concurrency |
| User-facing workflows | Use case model | Natural for stakeholder validation |
| Communication protocol | FSM | Protocol specs are naturally FSM-based |
| Embedded controller | Statechart | Modes, timeouts, history states |
| Multi-threaded system | Petri net | Token flow models thread interaction |
| API state machine | FSM | Request/response sequences are FSM transitions |

### Common Model-Based Testing Pitfalls

| Pitfall | Consequence | Prevention |
|---|---|---|
| Model too abstract | Generated tests miss implementation-specific faults | Add detail at transition level (guards, actions) |
| Model too detailed | State explosion, unmanageable test suite | Use hierarchy and abstraction |
| Model diverges from code | Tests pass on wrong behavior | Keep model in sync with requirements; treat as living artifact |
| Missing transitions | Untested behaviors | Review model for completeness; check every state x input pair |
| Ignoring non-functional aspects | Model-based tests cover only functional paths | Supplement with performance, security, usability testing |
| No oracle for outputs | Cannot verify correctness of outputs | Define expected outputs in the model (Mealy machine) |
