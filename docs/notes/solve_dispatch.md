# solve_dispatch Overview

`solve_dispatch` is a **cost-minimization operational optimization** — typically a **mixed-integer linear program (MILP)** when unit commitment is enabled. Capacities are **fixed** from planning; the solver decides how to operate generators, storage, and transmission hour-by-hour within each investment year.

## Solver stack

Same three layers as planning:

1. **PyPSA** builds each optimization problem (`network.optimize(...)`).
2. **Linopy** is the modeling layer.
3. **HiGHS** is the default/example solver (`solving.solver.name: highs`).

Unlike planning’s single multi-period solve, dispatch runs **many sequential solves** — one rolling unit-commitment (UC) window at a time, for each investment period.

## Planning vs dispatch

Yes — that is the intended split:

| | **solve_planning** | **solve_dispatch** |
|---|---|---|
| **Question** | What should we *build*? | How do we *operate* what was built? |
| **Capacities** | Decision variables (`p_nom_opt`, `s_nom_opt`, …) | Fixed from planning (`fix_optimal_capacities`) |
| **Time resolution** | Representative days / clustered snapshots across all investment periods at once | Full (or near-full) hourly timeline, **one year/period at a time** |
| **Horizon** | Multi-investment-period model in one solve | Rolling UC windows (`horizon` / `overlap`) within each year |
| **Objective** | Capex + O&M + fuel/carbon over the planning horizon (discounted) | Operating cost (fuel, carbon, UC, load shedding, …) for that year’s snapshots |
| **Typical output size** | Hundreds of weighted snapshots (e.g. ~360 rows in `generators-p.csv`) | ~8760 hours per year folder (e.g. `2050/generators-p.csv`) |

Bridge step between them: **`create_dispatch_network`** copies `*_opt` capacities from the planning solution onto the unfiltered planning network, calls `fix_optimal_capacities()`, drops zero-capacity assets, and flattens multi-period snapshots into a continuous DatetimeIndex. That produces `dispatch_planning_unsolved_network.nc`, which `solve_dispatch` consumes.

Nuance: planning *also* co-optimizes dispatch on its reduced snapshot set (so build decisions are consistent with operation). Dispatch then re-solves operation on a much finer timeline with fixed plant and line sizes — closer to “how this grid is deployed in a specific year.”

## Problem type

For each investment period and each UC window, minimize operating cost subject to:

- Nodal power balance, line/link limits, Kirchhoff voltage law
- Generator/storage limits, ramps, min up/down times (when `committable`)
- Fixed capacities (not extendable)
- Dispatch-specific policy constraints via `extra_functionality` (stop-production, bidirectional links, prevent-spill, CER, custom constraints)

Config (`minimal_model.yaml`):

```yaml
dispatch:
  horizon: 336   # hours per UC window (~2 weeks)
  overlap: 48    # hours of look-ahead / handoff between windows
  investment_period: []  # empty → use year_settings.investment_period

solving:
  options:
    dispatch:
      enable_committable: true
      load_shedding: true
      linearized_unit_commitment: false
```

## How the solve works

1. Load `dispatch_planning_unsolved_network.nc`.
2. Apply generator preprocess toggles (`enable_committable`, `p_min_pu`, ramp rates).
3. For each investment period (e.g. 2021, 2035, 2050):
   - Copy the network and keep only that year’s snapshots.
   - Drop inactive assets for the period (`drop_inactive_assets`).
   - Optionally add load shedding.
   - Call `optimize_uc_period(...)`, which splits the year into windows of length `horizon` with `overlap` look-ahead.
   - Between windows, carry forward storage SoC and generator up/down times so UC continuity is preserved.
   - CER budgets (if enabled) are tracked across UC periods.
4. Export each period’s solved network to its own subdirectory.

Core call inside each UC window:

```python
status, condition = network.optimize(
    snapshots=snapshots,
    linearized_unit_commitment=linearized_unit_commitment,
    include_objective_constant=include_objective_constant,
    solver_name=solver_name,
    solver_options=solver_options,
    extra_functionality=_extra_func,  # dispatch + CER + custom constraints
)
```

## Inputs

| Source | What |
|--------|------|
| **Rule input** | `resources/{run}/networks/dispatch_planning_unsolved_network.nc` — capacities fixed from planning |
| **Config** | `dispatch.horizon`, `dispatch.overlap`, `dispatch.investment_period`, `dispatch.constraints`, `dispatch.custom_constraints`, `solving.options.dispatch`, `solving.solver` |
| **Env** | `PYPSA_TEST_MODE=1` limits each period to 6 snapshots (and one UC window of that length) |

## Outputs

| Path | What |
|------|------|
| **Rule output** | `results/{run}/run_{timestamp}/dispatch_solved_network/` |
| **Per-period folders** | `2021/`, `2035/`, `2050/`, … — each a full PyPSA CSV export |
| **Solve stats** | `dispatch_solved_network/solve_stats.csv` — UC window timings and status |
| **Log** | `logs/{run}/solve_dispatch_{timestamp}.log` |
| **Benchmark** | timing/memory row in `results_benchmarks.csv` |

Each year folder contains the same style of CSV dump as planning (`generators.csv`, `generators-p.csv`, `generators-status.csv`, `storage_units-state_of_charge.csv`, `lines-p0.csv`, `loads-p.csv`, …), but with **fixed** `p_nom` / `p_nom_opt` and **hourly** operating series. Downstream rules (`post_process_dispatch`, corridor maps, summary, IDEA export) read these folders.

Example layout from the minimal model run:

```
dispatch_solved_network/
  solve_stats.csv
  2021/
  2035/
  2050/
    generators.csv
    generators-p.csv          # ~8760 rows
    generators-status.csv
    ...
```

## Related files

- Rule: `pypsa_canada/workflow/rules/solve_dispatch.smk`
- Script: `pypsa_canada/workflow/scripts/solve_dispatch.py`
- Network prep: `pypsa_canada/workflow/scripts/create_dispatch_network.py`
- Constraints: `pypsa_canada/workflow/scripts/constraints/dispatch_constraints.py`
- Generic constraints: `pypsa_canada/workflow/scripts/constraints/generic_constraints.py`
