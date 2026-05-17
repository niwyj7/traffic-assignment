
# Numerical Robustness of Logit-Based Stochastic User Equilibrium on the Sioux Falls Network

This project implements a **logit-based stochastic user-equilibrium (SUE) traffic assignment model** on the Sioux Falls benchmark network and studies its **numerical convergence, route-set sensitivity, demand perturbation sensitivity, and link-level vulnerability**.

Traffic assignment is formulated as a nonlinear fixed-point problem, solved with the Method of Successive Averages (MSA), and then stress-tested under perturbations.


## Dataset

The project uses three Sioux Falls files:

| File | Description |
|---|---|
| `SiouxFalls_node.csv` | Node coordinates for network visualization |
| `SiouxFalls_net.csv` | Directed links and polynomial link-cost coefficients |
| `SiouxFalls_od.csv` | Origin-destination demand matrix |

Dataset scale:

| Quantity | Value |
|---|---:|
| Nodes | 24 |
| Directed links | 76 |
| OD pairs | 528 |
| Total OD demand | 360,600 |

## Research Questions

1. Can logit-based stochastic user equilibrium be solved reliably as a fixed-point problem using MSA?
2. How does the route-choice sensitivity parameter $\theta\$ affect total travel cost and flow concentration?
3. How sensitive is the equilibrium flow to OD-demand perturbations?
4. Which links create the largest system-level cost increase under small adversarial-style cost perturbations?

## Mathematical Formulation

### Link cost function

Each directed link $a$ has flow $x_a$. The link travel cost is modelled as a polynomial:

$$
t_a(x_a)=a_{0,a}+a_{1,a}x_a+a_{2,a}x_a^2+a_{3,a}x_a^3+a_{4,a}x_a^4.
$$

The total system travel cost is:

$$
T(x)=\sum_{a\in A}x_a t_a(x_a).
$$

### Path cost

For path $r$, define $\delta_{ar}=1$ if link $a$ belongs to path $r$, and $0$ otherwise. The path cost is:

$$
C_r(x)=\sum_{a\in A}\delta_{ar}t_a(x_a).
$$

### Logit route choice

For OD pair $w=(o,d)$, demand $q_w$, and candidate route set $R_w$, the probability of choosing path $r$ in $R_w$ is (Gumbel distribution of error):

$$
P_r(x)=\frac{\exp(-\theta C_r(x))}{\sum_{s\in R_w}\exp(-\theta C_s(x))}.
$$

Here $\theta\$ controls route-choice sensitivity:

- small $\theta\$: more random route choice;
- large $\theta\$: stronger preference for low-cost paths.

The resulting path flow is:

$$
f_r=q_wP_r(x),
$$

and link flows are recovered by:

$$
x_a=\sum_w\sum_{r\in R_w}\delta_{ar}f_r.
$$

This defines a nonlinear fixed-point problem:

$$
x=F(x),
$$

where $F(x)$ maps the current link flow to the induced link flow after recomputing costs and logit route-choice probabilities.

### MSA fixed-point iteration

The Method of Successive Averages updates link flow as:

$$
x^{(k+1)}=x^{(k)}+\lambda_k\left(F(x^{(k)})-x^{(k)}\right),
$$

with:

$$
\lambda_k=\frac{1}{k+1}.
$$

Equivalently:

$$
x^{(k+1)}=(1-\lambda_k)x^{(k)}+\lambda_kF(x^{(k)}).
$$

## Implementation Pipeline

1. Load the node, link, and OD-demand data.
2. Build a directed NetworkX graph using free-flow cost as the path-generation weight.
3. Generate top-k shortest simple paths for each OD pair.
4. Build a path-link incidence matrix.
5. Solve logit-SUE with MSA.
6. Analyse convergence, total travel cost, high-flow links, route-choice sensitivity, route-set size sensitivity, demand perturbation sensitivity, and link-level vulnerability.

## Baseline Experiment

Baseline configuration:

| Parameter | Value |
|---|---:|
| Candidate paths per OD, $K$ | 5 |
| Route-choice sensitivity, $\theta\$ | 0.05 |
| Maximum MSA iterations | 120 |
| Stopping tolerance | $10^{-4}$ |

Baseline solver summary:

| Metric | Value |
|---|---:|
| MSA iterations | 32 |
| Final relative step | $9.56\times 10^{-5}$ |
| Final fixed-point residual | $2.98\times 10^{-3}$ |
| Total system travel cost | 8,172,994.86 |

The first relative-step value is excluded from the convergence plot below because the algorithm starts from near-zero flow, which makes the relative denominator artificially small. The fixed-point residual is the more meaningful diagnostic for MSA.


<img width="780" height="468" alt="image" src="https://github.com/user-attachments/assets/44e7a5a5-a498-43a7-8d12-f45abb2f6b07" />

<img width="777" height="468" alt="image" src="https://github.com/user-attachments/assets/82422278-a6f9-4ea0-994b-2c452e99b09c" />

## Baseline Equilibrium Flows

The highest-flow directed links in the baseline solution are concentrated around links connecting nodes 9, 10, 15, 16, 18, 20, and 22.

| Rank | Link | From | To | Equilibrium flow | Link cost | Flow × cost |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 25 | 10 | 9 | 25,442.77 | 8.03 | 204,267.43 |
| 2 | 24 | 9 | 10 | 25,335.83 | 7.94 | 201,280.42 |
| 3 | 42 | 15 | 10 | 24,793.19 | 16.20 | 401,704.09 |
| 4 | 27 | 10 | 15 | 24,706.70 | 16.06 | 396,803.79 |
| 5 | 54 | 18 | 16 | 23,269.53 | 3.88 | 90,275.87 |

<img width="640" height="596" alt="image" src="https://github.com/user-attachments/assets/f4c1a555-15d4-4a3f-9807-097682962073" />


## Route-Choice Sensitivity Analysis

The parameter $\theta$ controls how sharply users react to path-cost differences. As $\theta$ increases, route choice becomes more concentrated on low-cost paths. This can change both the system travel cost and the distribution of equilibrium flows.

<img width="768" height="468" alt="image" src="https://github.com/user-attachments/assets/a51a1d15-8464-4ef4-8d4f-d2921114f592" />

<img width="795" height="468" alt="image" src="https://github.com/user-attachments/assets/c0a4772a-7bd6-4f39-8d95-5b198a768ff9" />


## Route-Set Size Sensitivity

For each OD pair, only a finite candidate route set is used. Top-k shortest paths provide a tractable approximation to the full route-choice set.

The experiment compares $K\in\{1,3,5,8\}$. Larger $K\$ allows more alternative routes and reduces the dependence of the result on a restricted route set.

<img width="764" height="468" alt="image" src="https://github.com/user-attachments/assets/993e33d9-d832-4d70-9c9e-da42a09a5d04" />

## Demand Perturbation Sensitivity

To test numerical robustness, OD demands are perturbed as:

$$
q_w'=q_w(1+\epsilon_w), \qquad \epsilon_w\sim\mathcal{N}(0,\sigma^2),
$$

with nonnegative clipping to avoid negative demand.

The relative demand perturbation is:

$$
\frac{\|q'-q\|}{\|q\|},
$$

and the relative equilibrium-flow response is:


```math
\frac{\|x^*(q')-x^*(q)\|}{\|x^*(q)\|}.
```

An empirical condition-number-style sensitivity is defined as:


```math
\kappa_{\mathrm{emp}}
=
\frac{
\|x^*(q')-x^*(q)\| / \|x^*(q)\|
}{
\|q'-q\| / \|q\|
}.
```
<img width="786" height="468" alt="image" src="https://github.com/user-attachments/assets/c9c41ab5-a4a1-4d7e-99e5-5c9594e6267a" />


<img width="786" height="468" alt="image" src="https://github.com/user-attachments/assets/26eb00c0-b251-45ee-8542-ec2814730195" />


## Link-Level Vulnerability Experiment

To simulate an adversarial-style local perturbation, the free-flow cost of one high-flow link is increased by 10%:

$$
a'_{0,j}=1.1a_{0,j}.
$$

The vulnerability score is the increase in total system travel cost:

$$
\Delta T_j=T_j^{\mathrm{perturbed}}-T^{\mathrm{baseline}}.
$$

This asks whether a small cost change on one link can produce a disproportionate network-level effect.

<img width="889" height="490" alt="image" src="https://github.com/user-attachments/assets/1efb4472-c393-47fe-891a-da488a663801" />


The scatter plot below compares baseline link flow with vulnerability. A high-flow link is not automatically the most vulnerable link; vulnerability depends on network substitutability and how the perturbation redistributes flows.

<img width="791" height="468" alt="image" src="https://github.com/user-attachments/assets/9a0b3906-551a-4f47-91e1-45bc8e744e77" />

## Extensions

Possible extensions:

- Compare MSA with alternative fixed-point or projection algorithms.
- Add dynamic demand across multiple time periods.
- Study low-precision arithmetic effects on convergence and equilibrium stability.
- Formulate an adversarial optimisation problem that selects a sparse set of links to perturb under a fixed budget.

## Data source

The Sioux Falls network, node and OD demand data used in this project are from the Transportation Networks for Research repository:

Transportation Networks for Research Core Team. *Transportation Networks for Research*. Available at: https://github.com/bstabler/TransportationNetworks. Accessed January 2026.
