# supplier-allocation-milp

[![CI](https://github.com/arshdeepdandiala/supplier-allocation-milp/actions/workflows/ci.yml/badge.svg)](https://github.com/arshdeepdandiala/supplier-allocation-milp/actions/workflows/ci.yml)
![Python](https://img.shields.io/badge/python-3.11%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Order allocation across suppliers with quantity discounts, capacity and risk limits, formulated as a mixed-integer program.

> Work in progress. Sections below are filled in as each milestone lands.

## 1. Business problem

A mid-size industrial distributor buys several product families from a pool of approved suppliers each purchasing cycle. Suppliers offer all-units quantity discounts (a bigger order lowers the price of every unit), but each has limited capacity, a minimum order quantity, a fixed cost to place and manage an order, and a different defect rate. Buyers today send most volume to whoever quotes the lowest unit price, which leaves money on the table at the discount breaks and concentrates risk in one supplier.

**Question:** how much of each product should we order from each supplier to minimize total purchasing cost, while meeting demand, staying within supplier capacity, keeping average quality acceptable, and never relying on one supplier for more than a set share of any product?

## 2. Mathematical formulation

Single purchasing cycle, deterministic demand. One mixed-integer linear program.

### Sets

| Symbol | Meaning |
| --- | --- |
| $I$ | suppliers, indexed by $i$ |
| $J$ | products, indexed by $j$ |
| $K_{ij}$ | price tiers that supplier $i$ offers for product $j$, indexed by $k$ |

### Parameters

| Symbol | Meaning | Unit |
| --- | --- | --- |
| $d_j$ | demand for product $j$ this cycle | units |
| $c_{ijk}$ | unit price of product $j$ from supplier $i$ in tier $k$ | \$/unit |
| $L_{ijk}, U_{ijk}$ | lower and upper order quantity of tier $k$ (the first tier's $L$ is the minimum order quantity) | units |
| $\mathit{cap}_{ij}$ | capacity of supplier $i$ for product $j$ | units |
| $f_i$ | fixed cost of using supplier $i$ this cycle (ordering, freight minimum, admin) | \$ |
| $q_i$ | defect rate of supplier $i$ | fraction |
| $\bar{q}_j$ | maximum allowed average defect rate for product $j$ | fraction |
| $\alpha$ | maximum share of any product's demand from one supplier | fraction |

### Decision variables

| Symbol | Meaning |
| --- | --- |
| $x_{ijk} \ge 0$ | units of product $j$ bought from supplier $i$ at tier $k$ |
| $z_{ijk} \in \{0,1\}$ | 1 if the order of product $j$ to supplier $i$ falls in tier $k$ |
| $y_i \in \{0,1\}$ | 1 if supplier $i$ receives any order this cycle |

### Objective

Minimize purchase cost plus fixed supplier costs:

```math
\min \; \sum_{i \in I} \sum_{j \in J} \sum_{k \in K_{ij}} c_{ijk}\, x_{ijk} \;+\; \sum_{i \in I} f_i\, y_i
```

### Constraints

```math
\begin{aligned}
&\text{(1) Demand} && \textstyle\sum_{i}\sum_{k} x_{ijk} = d_j && \forall j \\
&\text{(2) Tier bounds} && L_{ijk}\, z_{ijk} \le x_{ijk} \le \tilde{U}_{ijk}\, z_{ijk} && \forall i,j,k \\
&\text{(3) One tier per order} && \textstyle\sum_{k} z_{ijk} \le 1 && \forall i,j \\
&\text{(4) Capacity} && \textstyle\sum_{k} x_{ijk} \le \mathit{cap}_{ij} && \forall i,j \\
&\text{(5) Supplier activation} && z_{ijk} \le y_i && \forall i,j,k \\
&\text{(6) Diversification} && \textstyle\sum_{k} x_{ijk} \le \alpha\, d_j && \forall i,j \\
&\text{(7) Quality} && \textstyle\sum_{i}\sum_{k} q_i\, x_{ijk} \le \bar{q}_j\, d_j && \forall j
\end{aligned}
```

where $\tilde{U}_{ijk} = \min(U_{ijk},\ \mathit{cap}_{ij},\ \alpha d_j)$.

### Modelling choices

- **All-units discounts as a multiple-choice model.** Each order picks at most one tier (3), and the tier's binary switches its quantity range on (2). Because the price applies to every unit, cost stays linear in $x_{ijk}$ once the tier is fixed.
- **Tight big-M.** The upper bound in (2) plays the role of big-M. Using $\tilde{U}_{ijk}$, the smallest bound the other constraints already imply, instead of an arbitrary large number gives a tighter LP relaxation, so branch-and-bound proves optimality faster.
- **Disaggregated activation.** (5) links each tier binary to $y_i$ one by one rather than through a single $\sum x \le M y_i$ constraint. It adds rows but gives a much stronger relaxation, which is the usual trade-off in facility-location-style models.
- **Minimum order quantity for free.** The first tier's lower bound $L_{ij1}$ is the MOQ, so (2) and (3) together say "order nothing, or at least the MOQ" without an extra constraint.
- **Linear quality constraint.** (7) is the average defect rate $\sum q_i x / d_j \le \bar{q}_j$ multiplied through by the constant $d_j$, so it stays linear.

### Baseline (current practice)

For each product, rank suppliers by their lowest-tier unit price and fill demand from the cheapest one until its capacity runs out, then the next. The baseline ignores fixed costs, discount breaks, the share cap and quality. It is scored with the same cost function, and its constraint violations are reported.

### Symbols in code

| Formulation | Code (`model.py`) |
| --- | --- |
| $I, J, K_{ij}$ | `m.I`, `m.J`, `m.IJK` |
| $x_{ijk}, z_{ijk}, y_i$ | `m.x[i, j, k]`, `m.z[i, j, k]`, `m.y[i]` |
| $d_j, c_{ijk}, f_i$ | `m.d[j]`, `m.c[i, j, k]`, `m.f[i]` |
| $\alpha$ | `m.alpha` |

## 3. Approach and solver choice

## 4. Results

## 5. Sensitivity analysis

## 6. Limitations and next steps

## 7. How to run

```bash
git clone https://github.com/arshdeepdandiala/supplier-allocation-milp
cd supplier-allocation-milp
pip install -e ".[dev]"
pytest
```
