## Problem Statement

---

### 🧩 What Are You Solving?

We want to generalize Dollar Cost Averaging (DCA) to maximize Bitcoin accumulation by allowing **dynamic purchases** (variable purchase amounts), while still preserving DCA’s core benefit: **regular, systematic purchases**.

---

### ⚙️ How You’ll Solve It

Build a **feature-driven model** (in Python) that maps historical features to daily purchase proportions (weights).  

> Each weight represents a portion of a fixed budget (normalized to 1) distributed over an investment window.


---

### 🎯 Objective

Maximize your model’s **SPD Percentile** across every 12-month rolling investment window (sliding daily—3,075 windows in total) over the backtest period `2016-01-01` → `2025-06-01`, while also ensuring **consistent outperformance** across all those windows.

#### Key Metrics

- **Satoshis per Dollar (SPD)**  
  Measures BTC accumulation per dollar invested:  
  $$
  \text{spd} = \frac{1}{\text{BTC/USD}} \times 100{,}000{,}000
  $$

- **SPD Percentile**  
  Ranks your SPD between the worst-case (all at highest price) and best-case (all at lowest price) strategies:
  
  $$
  \text{spd\_pct} = \frac{\text{your SPD} - \text{worst SPD}}{\text{best SPD} - \text{worst SPD}} \times 100
  $$

- **Win Rate**  
  The share of rolling investment windows where your strategy has a higher SPD Percentile than **Uniform DCA** (daily uniform purchases):  
  $$
  \text{win\_rate} = \frac{\#\text{ wins}}{\text{total windows}} \times 100
  $$

- **Recency-Weighted SPD Percentile**  
  A weighted average of all rolling-window SPD percentiles (each 12 months long) that places more emphasis on recent windows via exponential decay:  
  $$
  \text{RW\_spd\_pct}
  = \sum_{i=0}^{N-1} w_i \,\text{spd\_pct}_i,
  \quad
  w_i = \frac{\rho^{\,N-1-i}}{\displaystyle\sum_{j=0}^{N-1}\rho^{\,N-1-j}}
  $$  
  - $(0 < \rho < 1)$ is the decay rate (e.g. $(\rho=0.9)$)  
  - $(N)$ is the total number of rolling windows ($4,750$)  
  - Weights $(w_i)$ sum to 1, shifting emphasis toward more recent windows

> _Note: weights here reflect each window’s influence, not purchase fractions._  

### **Final Model Score**

Combine strength (recency-weighted percentile) and consistency (win rate) equally:

$$
\boxed{
\text{score}
= 0.5 \times \text{RW\_spd\_pct}
\;+\;
0.5 \times \text{win\_rate}
}
$$

---

### ✅ Model Constraints

1. **Positive Daily Purchases**  
   Every day must have: $\text{allocation}_t \geq 1 \times 10^{-5}$ 

2. **Budget Completeness**  
   Total daily allocations per investment window must sum to 1

3. **No Forward-Looking Data**  
   Use only current and past data—no peeking into the future

---

## 🏁 Evaluation Criteria

Among **valid** models, winners are those that:

- **Maximize Final Model Score**  
  Achieve the highest combined score of recency-weighted SPD percentile and win rate.

- **Interpretability**  
  Provide clear, interpretable logic or feature importance—no opaque “black-box” hacks.

- **Demonstrate Novel, Rigorous Bitcoin Analytics**  
  Leverage non-trivial, statistically sound techniques rather than simplistic heuristics.

- **Reproducible Under Specified Environment**  
  Code must adhere to our data formats, library versions, and environment constraints, ensuring it can be executed end-to-end by our evaluation engine.

- **Adaptable to Rolling Deployment**  
  Model logic must be modular and self-contained so it can be applied to any upcoming 12-month window using only historical data.

  
---

## 📦 Deliverables

**Model Code:**  
Submit your implementation using the Model Development template. 
