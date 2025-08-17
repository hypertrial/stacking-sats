# Problem Statement (Deep Dive)

---

**Watch the Old Deep Dive Walkthrough on Youtube:**
[![YouTube](https://img.shields.io/badge/Watch%20on-YouTube-red?logo=youtube&logoColor=white)](https://youtu.be/jYNRP98W1HA?si=tSKwPSo4L5H7McjN)

> Tackling this challenge means embracing its complexity as a **feature**, not a barrier.  
> Its rugged, non-convex landscape is a playground for your feature-driven creativity—every insight you contribute brings us one step closer to breakthrough solutions.  

---

Let $N$ be the number of days in one accumulation cycle (12 months). Our total investment budget is normalized to 1 and allocated over these $N$ days using a vector of daily weights $\mathbf{w} = (w_1, w_2, \dots, w_N)$.

Each weight must satisfy the following:

$$
w_i \geq \text{MIN\_WEIGHT} = 10^{-5},\quad 
\sum_{i=1}^N w_i = 1.
$$

This guarantees a strictly positive allocation every day, avoiding zero-buys while respecting the full budget constraint.

---

### Uniform DCA (Baseline)

(Daily) Uniform Dollar Cost Averaging corresponds to equal allocation on each day:

$$
w_i^{\text{uniform}} = \frac{1}{N},\quad \forall i \in \{1,\dots,N\}.
$$

This forms the baseline that your model aims to consistently outperform.

---

### Dynamic-DCA: Model Definition and Optimization Objective

The objective of this challenge is to build an optimal **data-driven model** that generates a valid allocation vector over the accumulation cycle and improves upon uniform DCA.

A valid model is a function:

$$
f(\text{features}) \mapsto \mathbf{w} \in \mathrm{int}\,\Delta^{N-1}
$$

where:
- **Features** are observable inputs derived from available features
- $\mathbf{w}$ is a vector of daily allocation weights
- The output $\mathbf{w}$ must satisfy the following constraints:

$$
\mathrm{int}\,\Delta^{N-1} = \left\{ \mathbf{w} \in \mathbb{R}^N \,\middle|\, w_i \geq 10^{-5},\; \sum_{i=1}^N w_i = 1 \right\}
$$

This is the (clipped) probability simplex, which ensures:

- **Strictly positive daily purchases**: $w_i \geq 10^{-5}$  
- **Full budget utilization**: $\sum_i w_i = 1$

The model must use **only current and past data**—no future information is allowed. This ensures the strategy is deployable in real time and avoids overfitting.

---

### Final Model Score

We score each strategy by:

$$
\mathrm{Score}(\mathbf w)
= 0.5\,\mathrm{RW\_spd\_pct}(\mathbf w)
\;+\;
0.5\,\mathrm{WinRate}(\mathbf w),
$$

where

- **Recency-Weighted SPD Percentile**:

  $$
  \mathrm{RW\_spd\_pct}(\mathbf w)
  = \sum_{i=0}^{N-1} w_i \,\mathrm{spd\_pct}_i,
  \quad
  w_i = \frac{\rho^{\,N-1-i}}{\sum_{j=0}^{N-1}\rho^{\,N-1-j}}
  $$

  - $0 < \rho < 1$ is the decay rate (e.g. $\rho=0.9$)  
  - $N$ is the total number of rolling windows (e.g. $4{,}750$)  
  - The $w_i$ sum to 1, shifting emphasis to more recent windows

> Note: 
> **Satoshis per Dollar (SPD)**  
>  Measures BTC accumulation per dollar invested:
> $$
  \text{spd} = \frac{1}{\text{BTC/USD}} \times 100{,}000{,}000
  $$

> Note:
> **SPD Percentile**  
>  Ranks your SPD between the worst-case (all at highest price) and best-case (all at lowest price) strategies:
> $$
  \text{spd\_pct} = \frac{\text{your SPD} - \text{worst SPD}}{\text{best SPD} - \text{worst SPD}} \times 100
  $$

  and

- **Win Rate** (indicator sum):

  $$
  \mathrm{WinRate}(\mathbf w)
  = \frac{1}{M}\sum_{k=1}^M
    \mathbf{1}\bigl(\mathrm{spd\_pct}^{(k)}(\mathbf w)
    > \mathrm{spd\_pct}^{(k)}_{\mathrm{DCA}}\bigr)
  $$

Your goal is to build a model that outputs weights $\mathbf w$ which maximize the Score:

$$
\max_{f}\;\mathrm{Score}\bigl(f(\mathrm{features})\bigr)
$$

That is, based on a set of daily features (BTC price, on-chain metrics, macro data, etc.), your model must output a valid weight vector, $\mathbf w \in \mathrm{int}\,\Delta^{N-1}$, that maximizes our combined **recency-weighted percentile** and **win-rate** metric.

---

### Not Just Optimization — A Valid Predictive Model

This challenge is **not** about cherry-picking hindsight weights:

> You must build a model that generalizes from observable features to weight allocations **without access to future data**.

In short, you’re solving a constrained optimization **indirectly**, via a predictive model that maps today’s features to tomorrow’s deployable allocation.  

---

## Deeper Dive: Feature-Driven Rules for Dynamic DCA

In this framework, a valid strategy (or model) is defined as a deterministic mapping:

$$
f: \mathcal{X} \rightarrow \mathrm{int}\,\Delta^{N-1}
$$

where:
- $\mathcal{X}$ is the space of observable features (derived from our data lake)
- $\mathrm{int}\,\Delta^{N-1}$ is the clipped simplex of valid weight vectors

At each cycle $t$, the model receives a feature vector $\mathbf{x}^{(t)}$ and outputs a valid allocation vector:

$$
\mathbf{w}^{(t)} = f\left( \mathbf{x}^{(t)}; \theta \right)
$$

with the constraints:

$$
w_i^{(t)} \geq 10^{-5},\quad 
\sum_{i=1}^N w_i^{(t)} = 1
$$

### Formalizing models this way: 

1. **Avoids hindsight bias.**  
   Allocations are **functionally generated**, not manually selected after seeing prices.

2. **Supports generalization.**  
   By choosing a parameterized function class (e.g., linear model, decision tree, neural net), models can be validated and tuned properly on historical cycles.

3. **Allows reproducibility and comparison.**  
   All strategies live in the same feasible space. Performance differences reflect model and feature quality—not optimization shortcuts.

4. **Encourages structural insight.**  
   Successful models reveal patterns and structural dynamics in Bitcoin’s behavior that improve DCA performance beyond uniform allocation.


**But it does not enforce the important non-forward looking criteria.** 

### Template for Non-Forward Looking Models

This template builds on the formal definiton of a model to outline a recursive Bayesian approach for developing models that ensure valid allocations while strictly adhering to the non-forward looking constraint . In this framework, each update depends only on all features observed up to the current day and on the most recent allocation, thereby simplifying the state history.

- **Initialization:**  
  Begin with a uniform prior over $N$ days:
  $$
  \mathbf{w}^{(0)} := \left[\frac{1}{N}, \frac{1}{N}, \dots, \frac{1}{N}\right] \in \mathrm{int}\left(\Delta^{N-1}\right)
  $$

  The pro of uniform weights is that they sum to 1 by definition. That is no longer the case when we have dynamic weights that are being allocated in a non-forward looking manner. 

- **Iterative Process:**  
  For each day $i$, update the allocation based on the features $(X_1, X_2, \dots, X_i)$ observed up to that day and on the allocation from the previous day, ensuring that each updated weight vector resides in the interior of the probability simplex:

  $$
  \begin{aligned}
    f\left(X_1,\, \mathbf{w}^{(0)}\right) &\mapsto \mathbf{w}_1 \in \mathrm{int}\left(\Delta^{N-1}\right) \quad \text{(@ day 1)} \\
    f\left(X_1, X_2,\, \mathbf{w}_1\right) &\mapsto \mathbf{w}_2 \in \mathrm{int}\left(\Delta^{N-1}\right) \quad \text{(@ day 2)} \\
    &\ \vdots \\
    f\left(X_1, \ldots, X_N,\, \mathbf{w}_{N-1}\right) &\mapsto \mathbf{w}_N \in \mathrm{int}\left(\Delta^{N-1}\right) \quad \text{(@ day N)}
  \end{aligned}
  $$

- **Final Allocation:**  
  Define the final weight vector for the cycle as:
  $$
  \mathbf{w} = \mathbf{w}_N \in \mathrm{int}\left(\Delta^{N-1}\right)
  $$

#### **Key Properties Ensured by the Template**

- **Non-Forward Looking:**  
  Each update $f\left(X_1, \ldots, X_i,\, \mathbf{w}_{i-1}\right)$ leverages only the features up to the current day and the immediately preceding allocation, strictly avoiding any forward-looking information.

- **Dynamic Budget Management:**  
  By relying on the most recent weight $\mathbf{w}_{i-1}$, the model can efficiently re-assess the remaining budget and rebalance allocations as needed, even allowing for early termination if too much of the budget is allocated early in the process.

- **Budget Conservation:**  
  The allocation vector remains on the simplex throughout, always satisfying:
  $$
  \sum_{i=1}^N w_i = 1 \quad \text{and} \quad w_i \geq 10^{-5} \quad \forall i
  $$

- **Adaptability:**  
  The function $f$ can be implemented using various model classes (e.g., linear models, decision trees, or neural networks), providing a flexible and reproducible framework across different strategies.


We recommend using this template as a starting point to ensure your model is not forward looking. 

---

## Geometry of the Optimization

When you optimize raw SPD over the **closed** simplex

$$
\Delta^{N-1}
= \{\mathbf w \in \mathbb{R}^N \mid w_i \ge 0,\;\sum_i w_i = 1\},
$$

the solution is trivially a **vertex**—i.e. put 100% of your budget on the single day with the lowest price. In hindsight, that perfectly times the market; in practice, with no forward-looking data, it’s impossible.

---

### Clipping to the **Open** Simplex

To rule out these “all-in” extremes, we force every weight to be at least $\varepsilon = 10^{-5}$:

$$
\mathrm{int}\,\Delta^{N-1}
= \{\mathbf w \in \mathbb{R}^N \mid w_i \ge \varepsilon,\;\sum_i w_i = 1\}.
$$

- **No degenerate corners**: true vertices are eliminated.  
- **Preserves DCA discipline**: every day gets a non-zero purchase.  
- **Keeps convexity**: any convex blend of two valid $\mathbf w$ is still valid.

This clipped region enforces a **trade-off** between:  
- Pushing toward low-price days (raw SPD)  
- Staying “close” to uniform DCA (diversified purchases)

---

### Introducing the **Final Model Score**

Rather than maximize SPD alone, we use:

$$
\mathrm{Score}(\mathbf w)
= 0.5\,\underbrace{\mathrm{RW\_spd\_pct}(\mathbf w)}_{\text{affine in }\mathbf w}
\;+\;
0.5\,\underbrace{\mathrm{WinRate}(\mathbf w)}_{\text{sum of indicator steps}}
$$

- **Recency-Weighted SPD Percentile** (affine in $\mathbf w$):  
  $$
    \mathrm{RW\_spd\_pct}(\mathbf w)
    = \sum_{i=0}^{N-1} w_i \,\mathrm{spd\_pct}_i.
  $$
  ⇒ **Linear** behavior in $\mathbf w$.

- **Win Rate** (indicator sum):  
  $$
    \mathrm{WinRate}(\mathbf w)
    = \frac{1}{M}\sum_{k=1}^M
      \mathbf{1}\!\Bigl(\mathrm{spd\_pct}^{(k)}(\mathbf w)
      > \mathrm{spd\_pct}^{(k)}_{\mathrm{DCA}}\Bigr).
  $$
  ⇒ **Non-convex**, with **flat plateaus** and **jump discontinuities**.

---

### The Resulting Landscape

1. **Convex Feasible Region**: $\mathbf w \in \mathrm{int}\,\Delta^{N-1}$  
   - Any mixture of valid strategies stays valid.

3. **Non-Convex Objective**  
   - **Flat regions** where WinRate is unchanged.  
   - **Sudden jumps** when crossing the DCA benchmark in a window.  
   - **Multiple local optima**, no simple analytic guarantee.

4. **No Vertex Guarantee**  
   - The best $\mathbf w$ need not lie on (or near) the boundary.

---

### Why This Mechanism Design?

- **Discipline + Diversity**: Clipping enforces daily buys, avoiding “all-in.”  
- **Balanced Performance**: Score blends **strength** (high recent SPD percentile) and **consistency** (beating DCA often).  
- **Deployable Models**: Convex domain ensures any candidate—even a mix—remains valid.  
- **Rich Search Terrain**: Non-convexity demands gradient-free or evolutionary optimizers to explore plateaus and jumps.

By moving from a trivial linear program on the closed simplex to a controlled, convex region plus a non-linear Score, we create a **mechanism** that:  
- **Prevents trivial hindsight solutions**,  
- **Encourages disciplined diversification**,  
- **Balances recent gains with consistency**,  
- **Yields a challenging yet deployable optimization problem**  
for anyone tackling dynamic DCA strategies.

### TLDR:

We turn a loose mandate—“accumulate as much Bitcoin as possible over a window”—into a **rigorous, deployable challenge**:

- **Indirect, Feature-Driven Search**  
  Students don’t hand-craft weight vectors directly. They learn a function, $f\colon\text{features}\to\mathbf w\in\mathrm{int}\,\Delta^{N-1}$, that must generalize, not over-fit. This extra mapping layer multiplies complexity and rewards true predictive insight over backtest hacking.

- **Discipline + Diversity**  
  Clipping at $w_i\ge10^{-5}$ guarantees every day gets some allocation, avoiding trivial “all-in” corner cases while preserving the convex search space.

- **Balanced, Skeptical Backtests**  
  A non-convex **Score** blends  
  1. **Recency-Weighted SPD Percentile** (linear in $\mathbf w$) and  
  2. **Win Rate** (indicator-based, non-convex)  
  so that simple over-fit examples are disregarded in favor of strategies that perform consistently across time.

- **Deployable, Convex Domain**  
  Despite the rugged Score landscape, the **open simplex** remains convex: any mix of valid strategies is still valid and deployable in real time.

- **Crowdsourced Creativity**  
  With hundreds of students exploring diverse model classes and feature sets, we’re effectively **brute-forcing** the search in parallel. Top ideas emerge, get ensembled, and drive collective progress.

- **Rich Learning & Impact**  
  This mechanism design:  
  - Turns a trivial hindsight Linear Program into a compelling non-convex optimization.  
  - Guards against over-confidence in backtests, demanding true generalization.  
  - Provides students with hands-on experience in a real world Bitcoin quant problem. 

By carefully constraining the domain and crafting a composite, non-convex metric, we ensure our backtests are **meaningful**, our solutions **deployable**.
