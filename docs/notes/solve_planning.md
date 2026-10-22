# solve_planning Overview

`solve_planning` is a **cost-minimization optimization**, specifically a **mixed-integer linear program (MILP)**. It finds the cheapest feasible set of capacities and dispatch, subject to physical and policy constraints.

## Solver stack

Three layers:

1. **PyPSA** builds the network optimization problem (`network.optimize.create_model` / `solve_model`).
2. **Linopy** is the modeling layer (variables, constraints, objective as a Linopy MILP).
3. **HiGHS** is the actual solver in the default/example configs.

From `example/config/minimal_model.yaml`:

```yaml
solving:
  solver:
    name: &solver_name highs
    options: highs-fast
```

That selects the `highs-fast` preset from `default_config.yaml` (IPM algorithm, looser tolerances, MIP gap 0.1). A typical run logs something like:

```
Solve problem using Highs solver
Status: ok / Termination condition: optimal
Objective: 1.95e+09
Solver: highs / Runtime: 5.61s
```

Other solvers are wired in config (`gurobi`, `cplex`, `cbc`, `glpk`, `xpress`, `copt`) — switch with `solving.solver.name` and a matching `solver_options` key.

## Problem type

Minimize total system cost over multi-investment periods (capital + O&M + fuel/carbon, discounted), subject to:

- Power balance, line limits, Kirchhoff voltage law
- Generator/storage operating limits, ramps, unit commitment (when enabled)
- Extendable capacity bounds (`p_nom`, `s_nom`)
- Custom policy constraints via `extra_functionality=add_all_planning_constraints`

The model can include binary UC variables (`Generator-status`, start-up/shut-down) plus continuous capacity/dispatch variables — hence MILP, not pure LP.

## Inputs

| Source | What |
|--------|------|
| **Rule input** | `resources/{run}/networks/planning_base_network.nc` — unsolved multi-period PyPSA network from `modify_components` |
| **Config** | `solving.options.planning`, `solving.solver`, `planning.constraints`, `planning.custom_constraints` |
| **Env** | `PYPSA_TEST_MODE=1` optionally truncates to 6 snapshots/period |

Before solve, the script:

1. Applies generator preprocess toggles (committable, `p_min_pu`, ramp rates)
2. Disables UC for *extendable* assets (`disable_committable_for_OPT`)
3. Drops zero-weight snapshots (or truncates in test mode)
4. Optionally adds load shedding
5. Creates the Linopy model with `multi_investment_periods=True`
6. Passes `add_all_planning_constraints` as `extra_functionality` (CER, emissions, reserve margin, stop-production, bidirectional links, custom capacity caps)

## Outputs

| Path | What |
|------|------|
| **Rule output** | `results/{run}/run_{timestamp}/planning_solved_network/` — PyPSA CSV export of the **solved** network |
| **Log** | `logs/{run}/solve_planning_{timestamp}.log` |
| **Benchmark** | timing/memory row in `results_benchmarks.csv` |

The CSV folder is a full solved network dump: static tables (`generators.csv`, `buses.csv`, …) plus time series (`generators-p.csv`, `storage_units-state_of_charge.csv`, `lines-p0.csv`, …). Optimized capacities land in columns like `p_nom_opt` / `s_nom_opt`; dispatch fills the `*-p.csv` series. Downstream rules (`create_dispatch_network`, `post_process_planning`) consume this directory.

## Core call

```python
network.optimize.create_model(
    multi_investment_periods=True,
    linearized_unit_commitment=linearized_uc_ena,
    include_objective_constant=include_objective_constant,
)

solve_status, solve_condition = network.optimize.solve_model(
    solver_name=solver_name,
    solver_options=solver_options,
    extra_functionality=add_all_planning_constraints,
)
```

If the model is infeasible and the solver is Gurobi/Xpress, it can compute an IIS; with HiGHS it just raises.

## Related files

- Rule: `pypsa_canada/workflow/rules/solve_planning.smk`
- Script: `pypsa_canada/workflow/scripts/solve_planning.py`
- Constraints: `pypsa_canada/workflow/scripts/constraints/planning_constraints.py`
- Generic constraints: `pypsa_canada/workflow/scripts/constraints/generic_constraints.py`
