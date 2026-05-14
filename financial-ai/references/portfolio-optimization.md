# Portfolio Optimization

Portfolio optimization applies mathematical and computational techniques to construct
investment portfolios that maximize return for a given level of risk, or minimize risk
for a target return. Modern approaches extend classical Markowitz mean-variance theory
with genetic algorithms, multi-objective optimization, and AI-driven risk estimation.

**Sources:** Chen & Takamura (2025), *Agent AI for Finance*; Soldatos & Kyriazis (2022),
*Big Data and AI in Digital Finance*

---

## 1. Classical Markowitz Mean-Variance Optimization

### 1.1 Theoretical Foundation

The Markowitz (1952) framework formulates portfolio selection as a quadratic optimization
problem. The investor chooses asset weights to minimize portfolio variance subject to
a target expected return.

**Mathematical formulation:**

```
Minimize:    w^T * Sigma * w          (portfolio variance)
Subject to:  w^T * mu >= r_target     (minimum return constraint)
             w^T * 1 = 1              (full investment constraint)
             w_i >= 0  for all i      (no short selling, optional)

Where:
  w     = vector of portfolio weights (n x 1)
  Sigma = covariance matrix of asset returns (n x n)
  mu    = vector of expected returns (n x 1)
  r_target = minimum acceptable expected return
```

### 1.2 Efficient Frontier

The set of optimal portfolios forms the efficient frontier in risk-return space:

```
Expected Return
     ^
     |          * (Maximum return portfolio)
     |         /
     |        /  <- Efficient frontier
     |       /
     |      *
     |     /
     |    * (Minimum variance portfolio)
     |   /
     |  * (Sub-optimal portfolios below frontier)
     |
     +-------------------------> Risk (Std Dev)
```

**Key portfolios on the frontier:**
- **Minimum variance portfolio:** Lowest possible risk regardless of return
- **Maximum Sharpe ratio portfolio:** Best risk-adjusted return (tangency portfolio)
- **Target return portfolio:** Minimum risk for a specified return level

### 1.3 Practical Challenges

| Challenge | Description | Common Solution |
|---|---|---|
| Estimation error | Expected returns and covariances estimated with noise | Shrinkage estimators (Ledoit-Wolf), Bayesian priors (Black-Litterman) |
| Concentration | Optimizer produces extreme weights | Weight constraints, diversification penalty |
| Instability | Small input changes cause large weight shifts | Regularization, resampled efficient frontier |
| Transaction costs | Frequent rebalancing erodes returns | Turnover constraints, no-trade zones |
| Non-normal returns | Fat tails and skewness violate Gaussian assumption | CVaR optimization, higher-moment models |

### 1.4 Black-Litterman Model

The Black-Litterman (1992) model combines market equilibrium returns with investor views:

```
Step 1: Derive implied equilibrium returns from market capitalization weights
  pi = delta * Sigma * w_market

Step 2: Express investor views as linear constraints on expected returns
  P * mu = q + epsilon,  epsilon ~ N(0, Omega)

Step 3: Combine equilibrium and views via Bayesian updating
  mu_BL = [(tau * Sigma)^(-1) + P^T * Omega^(-1) * P]^(-1)
           * [(tau * Sigma)^(-1) * pi + P^T * Omega^(-1) * q]

Where:
  pi    = implied equilibrium returns
  delta = risk aversion coefficient
  tau   = scaling factor for uncertainty in equilibrium
  P     = pick matrix (which assets are in each view)
  q     = view return vector
  Omega = uncertainty in views
```

---

## 2. Risk Metrics

### 2.1 Value at Risk (VaR)

VaR measures the maximum expected loss over a time horizon at a given confidence level.

```
VaR_alpha = -inf{ x : P(Loss > x) <= 1 - alpha }

Example: 1-day 95% VaR of $1M means:
  "There is a 95% probability that the portfolio will not lose more than
   $1M over the next trading day."
```

**VaR computation methods:**

| Method | Approach | Pros | Cons |
|---|---|---|---|
| Historical | Use empirical return distribution | No distribution assumption | Sensitive to sample period |
| Parametric | Assume normal distribution | Fast computation | Underestimates tail risk |
| Monte Carlo | Simulate many return paths | Flexible, handles non-linearity | Computationally expensive |
| Cornish-Fisher | Adjust normal VaR for skew/kurtosis | Better tail capture than parametric | Approximation can fail in extremes |

### 2.2 Conditional Value at Risk (CVaR / Expected Shortfall)

CVaR measures the expected loss given that VaR has been breached:

```
CVaR_alpha = E[Loss | Loss > VaR_alpha]

Properties:
  - Always >= VaR (it measures the average of the worst losses)
  - Coherent risk measure (satisfies subadditivity, unlike VaR)
  - Better captures tail risk behavior
  - Can be optimized using linear programming
```

**CVaR optimization formulation:**

```
Minimize:    CVaR_alpha(w) = zeta + (1/(1-alpha)) * E[max(loss(w) - zeta, 0)]
Subject to:  w^T * 1 = 1
             w_i >= 0

This is equivalent to a linear program when using scenario-based approximation.
```

### 2.3 Sharpe Ratio

The Sharpe ratio measures risk-adjusted return:

```
Sharpe Ratio = (E[R_p] - R_f) / sigma_p

Where:
  E[R_p] = expected portfolio return
  R_f    = risk-free rate
  sigma_p = portfolio standard deviation

Variants:
  - Sortino Ratio: Uses downside deviation instead of total std dev
  - Calmar Ratio: Uses maximum drawdown instead of std dev
  - Information Ratio: Measures active return per tracking error
```

### 2.4 Maximum Drawdown

```
Drawdown(t) = (Peak(t) - Value(t)) / Peak(t)
Maximum Drawdown = max over all t of Drawdown(t)

Where Peak(t) = max over s <= t of Value(s)
```

### 2.5 Risk Metric Comparison

| Metric | Measures | Tail Sensitivity | Coherent | Optimizable |
|---|---|---|---|---|
| Variance | Symmetric dispersion | Low | Yes | Quadratic program |
| VaR | Quantile loss threshold | Medium | No | Difficult (non-convex) |
| CVaR | Expected tail loss | High | Yes | Linear program |
| Sharpe | Risk-adjusted return | Low | N/A | Quadratic fractional |
| Max Drawdown | Peak-to-trough decline | High | No | Path-dependent |

---

## 3. Genetic Algorithm-Based Portfolio Optimization

### 3.1 Why Genetic Algorithms for Portfolios

Traditional optimizers struggle with:
- Non-convex constraints (cardinality limits, minimum lot sizes)
- Mixed-integer variables (number of holdings, buy/sell decisions)
- Multiple competing objectives
- Complex real-world constraints that cannot be expressed as linear/quadratic programs

Genetic algorithms (GAs) handle all of these naturally through population-based stochastic search.

### 3.2 Encoding and Operators

**Chromosome representation:**

```
Portfolio Chromosome (real-valued encoding):
  [w_1, w_2, w_3, ..., w_n]

  Where w_i = weight of asset i in the portfolio
  Constraint: sum(w_i) = 1, w_i >= 0

Alternative: Binary encoding for asset inclusion + real-valued weights
  [include_1, w_1, include_2, w_2, ..., include_n, w_n]
  Where include_i in {0, 1} and w_i is weight if included
```

**Genetic operators for portfolio optimization:**

| Operator | Implementation | Purpose |
|---|---|---|
| Selection | Tournament selection (k=3) | Favor fit individuals while maintaining diversity |
| Crossover | Blend crossover (BLX-alpha) | Combine parent weights with exploration |
| Mutation | Gaussian perturbation + renormalization | Explore nearby weight configurations |
| Repair | Normalize weights to sum to 1, clip to bounds | Enforce feasibility after operators |
| Elitism | Preserve top 5-10% unchanged | Prevent loss of best solutions |

### 3.3 Fitness Function Design

The fitness function encodes the optimization objective:

```
# Single-objective fitness (maximize Sharpe ratio)
fitness(chromosome) = (expected_return(w) - risk_free_rate) / std_dev(w)

# Penalized fitness for constraint violations
fitness(chromosome) = sharpe_ratio(w)
    - penalty_weight * max(0, num_assets(w) - max_holdings)
    - penalty_weight * max(0, sector_concentration(w) - max_sector_weight)
    - penalty_weight * max(0, turnover(w, w_current) - max_turnover)
```

### 3.4 GA Configuration for Portfolio Optimization

```
Typical GA Parameters:
  Population size:     100-500 (larger for more assets)
  Generations:         200-1000
  Crossover rate:      0.7-0.9
  Mutation rate:       0.01-0.05
  Elite fraction:      0.05-0.10
  Tournament size:     3-5
  Convergence:         Stop if best fitness unchanged for 50 generations
```

### 3.5 Constraint Handling

| Constraint | Type | GA Implementation |
|---|---|---|
| Full investment (sum = 1) | Equality | Normalization repair operator |
| Long-only (w >= 0) | Bound | Clip and renormalize |
| Max weight per asset | Bound | Clip to upper bound, redistribute excess |
| Cardinality (max N holdings) | Integer | Binary inclusion genes, penalty function |
| Sector limits | Linear inequality | Penalty function or repair operator |
| Turnover limit | Path-dependent | Include previous weights in fitness |
| Minimum lot size | Integer | Round to lots, adjust for feasibility |

---

## 4. Multi-Objective Portfolio Optimization

### 4.1 Problem Formulation

Real portfolio optimization involves multiple competing objectives:

```
Minimize:    f_1(w) = -E[R_p]          (maximize return, expressed as minimization)
Minimize:    f_2(w) = sigma_p           (minimize risk)
Minimize:    f_3(w) = -ESG_score(w)    (maximize ESG, optional)
Minimize:    f_4(w) = illiquidity(w)    (minimize liquidity risk, optional)

Subject to:  Feasibility constraints (full investment, bounds, cardinality, etc.)
```

### 4.2 Pareto Optimality

A portfolio is Pareto optimal if no other portfolio improves one objective without
worsening another. The set of all Pareto optimal portfolios forms the Pareto front.

```
Expected Return
     ^
     |    * * *
     |   *       *    <- Pareto front (no portfolio dominates another on this curve)
     |  *           *
     |   . . .        *
     |  .   .   .       *
     | .  .  .  .  .
     +-----------------------------> Risk
     
     * = Pareto optimal portfolios
     . = Dominated (sub-optimal) portfolios
```

### 4.3 NSGA-II for Portfolio Optimization

The Non-dominated Sorting Genetic Algorithm II (NSGA-II) is the standard multi-objective
evolutionary algorithm for portfolio optimization:

```
NSGA-II Algorithm:
  1. Initialize random population P_0 of size N
  2. For each generation:
     a. Create offspring Q_t via selection, crossover, mutation
     b. Combine R_t = P_t union Q_t
     c. Non-dominated sorting: rank solutions into fronts F_1, F_2, ...
     d. Fill next population P_{t+1}:
        - Add fronts F_1, F_2, ... until N reached
        - For the critical front (partially included), sort by crowding distance
        - Select solutions with largest crowding distance (diversity preservation)
  3. Return final Pareto front approximation
```

**Crowding distance** ensures diversity along the Pareto front by favoring solutions
in sparse regions over solutions in crowded regions.

### 4.4 Decision Support from Pareto Front

The Pareto front presents the investor with trade-off choices:

| Selection Method | Description | Best For |
|---|---|---|
| Knee point | Portfolio at the Pareto front "elbow" | Balanced risk-return trade-off |
| Target return | Minimum risk portfolio at target return | Return-constrained mandates |
| Maximum Sharpe | Tangent line from risk-free rate | Unconstrained risk-adjusted optimization |
| Risk budget | Allocate specific risk to each objective | Multi-mandate institutional investors |
| Preference function | Weight objectives by investor priorities | Customized investor profiles |

---

## 5. Advanced Optimization Techniques

### 5.1 Robust Portfolio Optimization

Account for uncertainty in input estimates:

```
Minimize:    max over (mu, Sigma) in Uncertainty_Set [ Risk(w, Sigma) ]
Subject to:  min over (mu, Sigma) in Uncertainty_Set [ Return(w, mu) ] >= r_target

Uncertainty set types:
  - Box uncertainty: mu in [mu_hat - epsilon, mu_hat + epsilon]
  - Ellipsoidal uncertainty: (mu - mu_hat)^T * S^(-1) * (mu - mu_hat) <= kappa
  - Factor model uncertainty: Uncertainty in factor loadings
```

### 5.2 Regime-Aware Optimization

Financial markets exhibit distinct regimes (bull, bear, crisis, recovery):

```
For each regime r in {bull, bear, crisis, recovery}:
  Estimate: mu_r, Sigma_r from regime-classified historical data
  Compute: P(regime = r) from current market indicators

Portfolio = sum over r: P(regime = r) * OptimalPortfolio(mu_r, Sigma_r)
```

### 5.3 Transaction Cost-Aware Optimization

```
Minimize:    Risk(w) + lambda * TransactionCost(w, w_current)

TransactionCost(w, w_current) = sum_i [ c_i * |w_i - w_current_i| * PortfolioValue ]

Where:
  c_i = per-unit transaction cost for asset i (spread + commission + market impact)
  w_current = current portfolio weights before rebalancing

Market impact model (Almgren-Chriss):
  impact_i = sigma_i * sqrt(|delta_w_i| * PortfolioValue / ADV_i)
```

### 5.4 Factor-Based Optimization

```
Return model:  r_i = alpha_i + sum_k (beta_ik * F_k) + epsilon_i

Factor covariance decomposition:
  Sigma = B * Sigma_F * B^T + D

Where:
  B = factor loading matrix (n x k)
  Sigma_F = factor covariance matrix (k x k, with k << n)
  D = diagonal matrix of idiosyncratic variances

Optimization in factor space reduces dimensionality from n to k.
```

---

## 6. AI-Enhanced Portfolio Optimization

### 6.1 Deep Learning for Return Prediction

| Model | Input | Output | Use in Optimization |
|---|---|---|---|
| LSTM | Historical returns, features | Return forecast | Expected return estimate (mu) |
| Transformer | Multi-asset price history | Cross-asset forecasts | Joint return distribution |
| GNN | Asset relationship graph | Node-level predictions | Covariance structure estimation |
| VAE | Historical return distributions | Simulated scenarios | Scenario-based CVaR optimization |

### 6.2 Reinforcement Learning for Dynamic Allocation

```
State:    s_t = (current_weights, market_features, portfolio_value, time)
Action:   a_t = target_weights (continuous action space)
Reward:   r_t = risk_adjusted_return(s_t, a_t) - transaction_cost(a_t)
Policy:   pi(a_t | s_t) learned via DDPG, PPO, or SAC

Advantages over static optimization:
  - Naturally handles transaction costs (part of reward)
  - Adapts to changing market conditions (policy learns regime response)
  - Can incorporate non-standard objectives (drawdown avoidance, tax efficiency)
```

### 6.3 LLM-Assisted Portfolio Construction

- **Constraint extraction:** Parse investment policy statements using LLMs to automatically
  extract optimization constraints (asset class bounds, ESG requirements, etc.)
- **Narrative explanation:** Generate natural language explanations of portfolio allocation
  rationale for client reports
- **Scenario generation:** Use LLMs to generate forward-looking macro scenarios for
  stress testing portfolios
- **View generation:** Translate analyst reports into Black-Litterman model views using
  NLP-based opinion extraction

---

## 7. Performance Evaluation

### 7.1 Backtesting Framework

```
Walk-Forward Backtesting:
  For each rebalancing date t in [t_1, t_2, ..., t_T]:
    1. Use data up to t for estimation (expanding or rolling window)
    2. Solve optimization problem to get weights w_t
    3. Hold portfolio w_t until next rebalancing date t+1
    4. Record realized return: r_t = w_t^T * actual_returns(t to t+1)
    5. Account for transaction costs on weight changes

  Compute:
    - Cumulative return, annualized return
    - Annualized volatility, maximum drawdown
    - Sharpe ratio, Sortino ratio, Calmar ratio
    - Turnover per rebalancing period
    - Information ratio vs. benchmark
```

### 7.2 Statistical Significance

| Test | Purpose | Null Hypothesis |
|---|---|---|
| Sharpe ratio test (Ledoit-Wolf) | Compare two strategies | Equal Sharpe ratios |
| Bootstrap confidence interval | Estimate metric uncertainty | Point estimate is sufficient |
| Spanning test (Huberman-Kandel) | New asset adds value? | Existing frontier spans new asset |
| Out-of-sample R-squared | Predictive power | No predictive improvement |

### 7.3 Benchmark Comparison

| Benchmark | Description | When to Use |
|---|---|---|
| 1/N (Equal weight) | Naive diversification | Baseline for any strategy |
| Market cap weight | Passive index tracking | Active vs. passive comparison |
| Risk parity | Equal risk contribution | Risk-focused comparison |
| Minimum variance | Lowest risk feasible portfolio | Risk-minimization comparison |
| 60/40 Stock/Bond | Traditional balanced portfolio | Multi-asset comparison |

---

## Summary

Portfolio optimization has evolved from Markowitz mean-variance to a rich ecosystem of
techniques incorporating genetic algorithms for complex constraints, multi-objective
Pareto optimization for competing goals, and AI-enhanced methods for dynamic allocation.
The practical implementation requires careful attention to estimation error, transaction
costs, and regime changes, with genetic algorithms providing a flexible framework for
handling the non-convex, multi-constraint reality of institutional portfolio management.
