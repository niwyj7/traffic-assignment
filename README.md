# Iterative User Equilibrium on Sioux Falls

This repo is a small numerical-analysis project for traffic assignment.
It is not a traffic simulator. The goal is to study an iterative equilibrium
method: convergence, step-size stability, and demand perturbation sensitivity.

## Problem

For each origin-destination demand pair, drivers repeatedly choose the current
shortest route. Road travel time depends on road flow, so path choice and road
cost are coupled.

At user equilibrium, no driver can unilaterally switch to a cheaper path.
For each OD pair, all used paths have equal minimal cost.

## Cost Model

The script uses the BPR travel-time function:

```text
t_e(x_e) = t_e^0 * (1 + alpha * (x_e / c_e)^beta)
```

where:

- `t_e^0` is free-flow travel time
- `c_e` is road capacity
- `x_e` is current road flow
- default `alpha = 0.15`
- default `beta = 4`

The provided `SiouxFalls_net.csv` has columns:

```text
LINK,A,B,a0,a1,a2,a3,a4
```

For this format, `a0` is treated as free-flow time. Capacity is inferred from:

```text
a4 = alpha * a0 / capacity^beta
```

## Algorithm

1. Initialize all road flows as zero.
2. Compute road costs from the current flows using the BPR function.
3. For every OD pair, find the shortest path under current costs.
4. Assign all demand to those shortest paths. This gives auxiliary flow `y`.
5. Update flows using MSA:

```text
x^{k+1} = x^k + gamma_k * (y^k - x^k)
gamma_k = 1 / k
```

6. Repeat until the relative flow change is small.

## Run

The script has no third-party Python dependencies.

```bash
python3 traffic_equilibrium.py
```

By default it reads:

```text
/Users/yujiewu/Desktop/ESDA dissertation/traffic/SiouxFalls_net.csv
/Users/yujiewu/Desktop/ESDA dissertation/traffic/SiouxFalls_od.csv
```

Outputs are written to `results/`:

- `convergence.csv`
- `equilibrium_flows.csv`
- `convergence.svg`

## Experiments

Base MSA convergence:

```bash
python3 traffic_equilibrium.py --experiment base --max-iter 200
```

Step-size stability comparison:

```bash
python3 traffic_equilibrium.py --experiment stepsize --max-iter 200 --output-dir results_stepsize
```

Demand perturbation sensitivity:

```bash
python3 traffic_equilibrium.py --experiment perturbation --demand-factor 1.05 --output-dir results_perturbation
```

## Main Outputs

Use `convergence.svg` to discuss convergence:

```text
relative ||x^{k+1} - x^k||_2
```

Use `equilibrium_flows.csv` as the computed approximate equilibrium flow.

For a dissertation-style numerical analysis discussion, focus on:

- whether MSA converges smoothly
- how different step sizes affect oscillation or stability
- how much equilibrium flow changes after demand perturbation
