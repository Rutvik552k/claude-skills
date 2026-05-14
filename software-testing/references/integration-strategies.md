# Integration Testing Strategies

Decomposition-based, call-graph-based, and path-based integration strategies with tradeoffs, stub/driver patterns, and decision criteria.

---

## 1. Top-Down vs Bottom-Up: Detailed Comparison

### Top-Down Integration

**Process**: Start with the top-level module. Replace each called module with a stub. Test the top module. Then replace one stub at a time with the real module (and add stubs for that module's callees). Repeat until all modules are integrated.

**Example call graph**:

```
        A
       / \
      B   C
     / \   \
    D   E   F
```

**Integration order (depth-first)**: A, B, D, E, C, F

| Step | Module Under Test | Real Modules | Stubs Needed |
|---|---|---|---|
| 1 | A | A | stub_B, stub_C |
| 2 | A + B | A, B | stub_C, stub_D, stub_E |
| 3 | A + B + D | A, B, D | stub_C, stub_E |
| 4 | A + B + D + E | A, B, D, E | stub_C |
| 5 | A + B + D + E + C | A, B, D, E, C | stub_F |
| 6 | All | A, B, D, E, C, F | (none) |

**Advantages**:
- Interface faults found early (top-level interfaces tested first)
- Working skeleton of the system available early for demos
- Control flow is exercised from the start
- Natural fit when architecture is well-defined top-down

**Disadvantages**:
- Stubs can be complex (must simulate realistic return values)
- Lower-level modules tested late; faults there found late
- Stubs may mask faults in lower modules
- Difficult when lower-level modules contain critical algorithms

### Bottom-Up Integration

**Process**: Start with leaf modules (no callees). Create drivers (test harnesses) that call them. Test leaf modules. Then integrate with their callers, replacing drivers. Repeat upward until the top module is reached.

**Integration order**: D, E, F, B, C, A

| Step | Module Under Test | Real Modules | Drivers Needed |
|---|---|---|---|
| 1 | D | D | driver_for_D |
| 2 | E | E | driver_for_E |
| 3 | F | F | driver_for_F |
| 4 | B (with D, E) | B, D, E | driver_for_B |
| 5 | C (with F) | C, F | driver_for_C |
| 6 | A (all) | All | (none) |

**Advantages**:
- No stubs needed (leaf modules are real code)
- Drivers are simpler than stubs (just call functions and check results)
- Critical lower-level modules tested early
- Parallel development: leaf modules can be tested independently

**Disadvantages**:
- Top-level control flow not tested until late
- No working system skeleton until final integration
- Interface mismatches between upper modules found late
- Hard to test end-to-end scenarios early

### Side-by-Side Comparison

| Factor | Top-Down | Bottom-Up |
|---|---|---|
| Stubs required | Yes (many, complex) | No |
| Drivers required | No | Yes (many, simpler) |
| Early system demo | Yes | No |
| Interface fault detection | Early | Late |
| Critical algorithm testing | Late | Early |
| Parallel development | Limited | Easy |
| Fault isolation | Good at top, poor at bottom | Good at bottom, poor at top |
| Complexity of test doubles | High (stubs simulate behavior) | Low (drivers just invoke) |
| Suitable for | UI-driven systems | Library/utility-heavy systems |

### Sandwich (Hybrid) Integration

Combines top-down and bottom-up: integrate from the top down for the upper layers and from the bottom up for the lower layers. Meet in the middle.

**Integration order**: A (with stubs), D, E, F (with drivers), then B, C (connect upper to lower).

**Advantages**:
- Balances early interface testing with early algorithm testing
- Reduces total number of stubs and drivers
- Parallel testing of upper and lower layers

**Disadvantages**:
- More complex planning
- The "middle" layer may still need both stubs and drivers temporarily
- Requires clear identification of the boundary between upper and lower layers

### Big-Bang Integration

Integrate all modules at once and test the complete system.

**Advantages**:
- No stubs or drivers needed
- Simple to execute (just build everything)

**Disadvantages**:
- Fault isolation is extremely difficult
- When a test fails, any module could be the cause
- Debugging is time-consuming
- Only practical for very small systems (fewer than 5 modules)

---

## 2. Stub and Driver Design Patterns

### Stub Patterns

#### Pattern 1: Constant Return Stub

Returns a fixed value regardless of input. Simplest but least realistic.

```python
def stub_calculate_tax(amount):
    return 0.10  # always returns 10% rate
```

**Use when**: caller only needs a valid return type, not a realistic value.

#### Pattern 2: Table-Driven Stub

Returns pre-configured values based on input lookup.

```python
class StubDatabase:
    _data = {
        "user_001": {"name": "Alice", "role": "admin"},
        "user_002": {"name": "Bob", "role": "user"},
    }

    def get_user(self, user_id):
        return self._data.get(user_id, None)
```

**Use when**: caller behavior depends on specific return values; need controlled test scenarios.

#### Pattern 3: Counting Stub

Tracks how many times it was called and with what arguments.

```python
class StubNotifier:
    def __init__(self):
        self.call_count = 0
        self.last_args = None

    def send_notification(self, user_id, message):
        self.call_count += 1
        self.last_args = (user_id, message)
        return True
```

**Use when**: need to verify the caller invokes the stub correctly (interaction testing).

#### Pattern 4: Stateful Stub

Maintains internal state that changes with successive calls.

```python
class StubIterator:
    def __init__(self, values):
        self._values = iter(values)

    def next_value(self):
        return next(self._values, None)
```

**Use when**: caller expects different results on successive calls (e.g., reading records from a data source).

#### Pattern 5: Exception-Throwing Stub

Raises an exception to test error handling in the caller.

```python
def stub_connect_to_server():
    raise ConnectionError("Simulated network failure")
```

**Use when**: testing the caller's error handling and recovery paths.

### Driver Patterns

#### Pattern 1: Simple Invocation Driver

Calls the module under test with fixed arguments and prints/asserts the result.

```python
def driver_for_sort():
    result = sort_module.merge_sort([3, 1, 4, 1, 5, 9])
    assert result == [1, 1, 3, 4, 5, 9], f"Expected sorted list, got {result}"
```

#### Pattern 2: Parameterized Driver

Reads test cases from a data source and runs each one.

```python
def driver_for_parser(test_cases_file):
    with open(test_cases_file) as f:
        for line in f:
            input_str, expected = line.strip().split("|")
            result = parser.parse(input_str)
            assert str(result) == expected, f"Failed on input: {input_str}"
```

#### Pattern 3: Interactive Driver

Prompts the tester for input values (useful during exploratory testing).

```python
def driver_for_calculator():
    while True:
        expr = input("Enter expression (or 'quit'): ")
        if expr == 'quit':
            break
        result = calculator.evaluate(expr)
        print(f"Result: {result}")
```

#### Pattern 4: Sequence Driver

Calls the module multiple times in a specific order to test stateful behavior.

```python
def driver_for_stack():
    stack = Stack()
    stack.push(10)
    stack.push(20)
    assert stack.pop() == 20
    assert stack.pop() == 10
    assert stack.is_empty()
```

---

## 3. Call Graph Analysis for Integration Order

### Constructing the Call Graph

A call graph is a directed graph where:
- Each node represents a module (function, method, class)
- Each edge represents a call relationship (caller -> callee)

### Example: E-Commerce System

```
                    OrderController
                    /      |      \
           OrderService  AuthService  NotificationService
            /      \         |
    ProductRepo  PaymentGateway  UserRepo
        |
    DatabaseAdapter
```

### Topological Sort for Bottom-Up Order

Perform a reverse topological sort to get bottom-up integration order:

1. DatabaseAdapter (leaf)
2. ProductRepo, PaymentGateway, UserRepo (depend only on leaves)
3. OrderService, AuthService, NotificationService (depend on repos)
4. OrderController (top)

### Fan-In and Fan-Out Analysis

| Module | Fan-In (called by) | Fan-Out (calls) | Risk |
|---|---|---|---|
| OrderController | 0 | 3 | High fan-out: integration risk |
| OrderService | 1 | 2 | Moderate |
| AuthService | 1 | 1 | Low |
| NotificationService | 1 | 0 | Leaf: test first |
| ProductRepo | 1 | 1 | Low |
| PaymentGateway | 1 | 0 | Leaf: test first |
| UserRepo | 1 | 0 | Leaf: test first |
| DatabaseAdapter | 1 | 0 | Leaf: test first |

**Principle**: High fan-out modules are integration risk points. High fan-in modules are impact risk points (a fault there affects many callers).

### Pairwise Integration from Call Graph

Test each caller-callee edge as an independent integration test:

| Test | Caller | Callee | Focus |
|---|---|---|---|
| P1 | OrderController | OrderService | Order creation interface |
| P2 | OrderController | AuthService | Authentication interface |
| P3 | OrderController | NotificationService | Notification interface |
| P4 | OrderService | ProductRepo | Product lookup interface |
| P5 | OrderService | PaymentGateway | Payment processing interface |
| P6 | AuthService | UserRepo | User credential interface |
| P7 | ProductRepo | DatabaseAdapter | Database access interface |

Total pairwise tests = number of edges in call graph = 7.

### Neighborhood Integration

For each node, test it with all its direct neighbors (callers and callees):

| Test | Center Node | Neighborhood | Modules Involved |
|---|---|---|---|
| N1 | OrderController | All callees | OrderController, OrderService, AuthService, NotificationService |
| N2 | OrderService | Caller + callees | OrderController, OrderService, ProductRepo, PaymentGateway |
| N3 | AuthService | Caller + callee | OrderController, AuthService, UserRepo |
| N4 | ProductRepo | Caller + callee | OrderService, ProductRepo, DatabaseAdapter |

Neighborhood integration tests more complex interactions than pairwise but requires fewer total test configurations than testing all paths.

---

## 4. MM-Path (Method-to-Method Path) Construction

### Definition

An MM-path is a sequence of method executions linked by messages (calls and returns). It traces the dynamic behavior of a use case or feature through the system's method calls.

### Construction Process

1. Start from the entry point (e.g., an API endpoint or UI event handler)
2. Trace the method calls and returns through the call graph
3. Include the internal path through each method (from entry to exit or to the next call)
4. An MM-path ends when control returns to the original entry point or reaches a terminal node

### Example: Place Order Feature

```
1. OrderController.placeOrder(userId, items)
   -> calls AuthService.validateToken(userId)
      -> calls UserRepo.findById(userId)
         -> returns User
      -> returns isValid (boolean)
   -> calls OrderService.createOrder(user, items)
      -> calls ProductRepo.checkInventory(items)
         -> calls DatabaseAdapter.query(sql)
            -> returns ResultSet
         -> returns InventoryStatus
      -> calls PaymentGateway.charge(user, total)
         -> returns PaymentResult
      -> returns Order
   -> calls NotificationService.sendConfirmation(order)
      -> returns void
   -> returns OrderResponse
```

### MM-Path Representation

```
MM-Path 1 (happy path):
  OrderController.placeOrder
    -> AuthService.validateToken
      -> UserRepo.findById
      <- User
    <- isValid=true
    -> OrderService.createOrder
      -> ProductRepo.checkInventory
        -> DatabaseAdapter.query
        <- ResultSet
      <- inStock=true
      -> PaymentGateway.charge
      <- success=true
    <- Order
    -> NotificationService.sendConfirmation
    <- void
  <- OrderResponse(success)

MM-Path 2 (auth failure):
  OrderController.placeOrder
    -> AuthService.validateToken
      -> UserRepo.findById
      <- null
    <- isValid=false
  <- OrderResponse(unauthorized)

MM-Path 3 (out of stock):
  OrderController.placeOrder
    -> AuthService.validateToken [...]
    <- isValid=true
    -> OrderService.createOrder
      -> ProductRepo.checkInventory
        -> DatabaseAdapter.query
        <- ResultSet
      <- inStock=false
    <- null (order not created)
  <- OrderResponse(outOfStock)

MM-Path 4 (payment failure):
  OrderController.placeOrder
    -> AuthService.validateToken [...]
    <- isValid=true
    -> OrderService.createOrder
      -> ProductRepo.checkInventory [...]
      <- inStock=true
      -> PaymentGateway.charge
      <- success=false
    <- null (payment failed)
  <- OrderResponse(paymentFailed)
```

### Test Cases from MM-Paths

| MM-Path | Test Scenario | Key Assertions |
|---|---|---|
| 1 | Valid user, items in stock, payment succeeds | Order created, confirmation sent |
| 2 | Invalid user token | 401 returned, no order created |
| 3 | Valid user, item out of stock | Out-of-stock error, no payment attempted |
| 4 | Valid user, in stock, payment declined | Payment error, order rolled back |

---

## 5. Decision Criteria for Choosing Integration Strategy

### Decision Matrix

| Factor | Top-Down | Bottom-Up | Sandwich | Big-Bang |
|---|---|---|---|---|
| System size > 20 modules | Good | Good | Best | Poor |
| System size < 5 modules | Overkill | Overkill | Overkill | Acceptable |
| Critical algorithms at bottom | Poor | Best | Good | Poor |
| UI-heavy system | Best | Poor | Good | Poor |
| Tight schedule, parallel teams | Poor | Best | Good | Poor |
| Need early demo/prototype | Best | Poor | Good | Poor |
| Fault isolation priority | Good | Good | Best | Poor |
| Minimal test infrastructure | Poor (stubs) | Poor (drivers) | Poor (both) | Best |
| Third-party component integration | Varies | Best (test wrapper first) | Good | Poor |
| Microservice architecture | N/A | Best (test each service) | N/A | Poor |

### Decision Flowchart (Textual)

```
1. Is the system very small (< 5 modules)?
   Yes -> Big-bang may be acceptable
   No  -> Continue

2. Are there critical algorithms in leaf modules?
   Yes -> Favor bottom-up (test algorithms early)
   No  -> Continue

3. Is an early working prototype needed?
   Yes -> Favor top-down (skeleton first)
   No  -> Continue

4. Do you have multiple development teams working in parallel?
   Yes -> Favor bottom-up (independent leaf testing)
   No  -> Continue

5. Is the system large (> 20 modules) with both critical UI and algorithms?
   Yes -> Use sandwich (combine top-down for UI, bottom-up for algorithms)
   No  -> Default to bottom-up (simpler test infrastructure)
```

### Risk-Based Strategy Selection

| Risk Profile | Recommended Strategy | Rationale |
|---|---|---|
| High interface risk | Top-down | Exercises interfaces early |
| High algorithm risk | Bottom-up | Tests algorithms in isolation first |
| High schedule risk | Bottom-up + parallel | Teams can test independently |
| High visibility risk | Top-down | Early demo capability |
| Balanced risk | Sandwich | Covers both interface and algorithm risks |
| Low risk, small system | Big-bang | Minimizes infrastructure overhead |

---

## 6. Integration Testing Anti-Patterns

| Anti-Pattern | Problem | Solution |
|---|---|---|
| Testing through the UI only | Slow, brittle, misses internal interface faults | Add API-level integration tests |
| No integration tests, only unit + E2E | Gap between unit and system level | Add pairwise or neighborhood integration tests |
| Overly complex stubs | Stubs contain as much logic as real modules | Simplify stubs; use table-driven pattern |
| Ignoring error paths in integration | Happy path works, error handling untested | Add MM-paths for failure scenarios |
| Testing everything together from day one | Fault isolation impossible | Use incremental strategy (any variety) |
| Integration tests that depend on external services | Flaky, slow, environment-dependent | Use contract tests or stub external services |
| No clear integration order | Random module integration | Derive order from call graph analysis |
