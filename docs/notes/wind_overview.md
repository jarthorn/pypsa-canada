# Wind Variability Overview

Wind variability enters as an **exogenous hourly capacity-factor series**, not as something the optimizer invents. The model then uses that series in two different ways: compressed for planning, full-resolution for dispatch.

## 1. Input: hourly wind availability

Wind plants have time-varying availability in `generators-p_max_pu.csv`. Each cell is a capacity factor in \([0, 1]\): fraction of nameplate that can produce in that hour.

In the minimal model that file is **8760 hours × wind/solar columns**. Wind columns (`BUS_2 Wind`, `BUS_* OPT Wind`) have mean CF ≈ 0.50, ranging from near 0 to 1 — so calm hours and full-output hours are both present.

`load_network` imports the PyPSA CSV folder, so those series become `network.generators_t.p_max_pu`.

In the solve, PyPSA enforces:

\[
0 \le p_{g,t} \le p\_nom_g \times p\_max\_pu_{g,t}
\]

So when the wind CF is low, that generator simply cannot dispatch much, no matter how cheap it is.

## 2. Loss adjustment and multi-year tiling

In `add_components`, wind CFs are scaled by generator `efficiency` (e.g. 0.86 for wind in the generic data) via `apply_wind_loss_factors` — a simple loss/derating on top of the meteorological CF.

In `add_snapshots`, the **reference-year** hourly `p_max_pu` is tiled across investment periods (2021, 2035, 2050). Weather shape repeats; capacities and policies change by period. This is not climate-change-adjusted wind; it is the same annual wind pattern reused.

## 3. Planning: compress wind into representative days

With `snapshots.method: "CARPE_DIEM"` and `cluster: 5`, planning does **not** optimize all 8760 hours. `add_representative_days` selects a few days that preserve the joint shape of:

- **net load** (load − run-of-river hydro)
- **wind** (provincial average CF)
- **solar** (provincial average CF)

Per province and period, those series are reshaped to 365×24, min–max normalized, concatenated, and clustered (`NextGridClusterer`). Selected days get `snapshot_weightings` so they stand in for many similar days.

CARPE_DIEM then **rescales** wind/solar `p_max_pu` on the representative days so the **weighted annual mean CF** still matches the full-year mean (capped at 1). That keeps annual wind energy roughly right even though only a handful of days are kept.

`solve_planning` drops snapshots with zero objective weight, so the MILP only sees those weighted days. Capacity expansion therefore “sees” wind variability in a reduced but wind-aware sample — including periods of low wind concurrent with high net load, to the extent the clusterer retained them.

Other methods (`KMEDOID_VRE_HYDRO`, `OPT_VRE_HYDRO`) also fold wind into clustering; `ALL_DAYS` would keep every hour (expensive).

## 4. Planning reserve: firm capacity credit

Separately from hourly CF, the planning reserve margin uses a **static capacity credit** for wind (e.g. 0.21 in `capacity_values_placeholder.csv`). That is a firm-capacity approximation for reliability constraints, not the hourly weather series.

## 5. Dispatch: full hourly wind again

After planning, `create_dispatch_network` takes the **unfiltered / full-timeline** network (with capacities fixed from planning) and resets snapshot weights to 1. `solve_dispatch` then operates each year on ~8760 hours, so wind ramps and lulls bind hour by hour against load, storage, UC, and transmission.

That is why planning `generators-p.csv` has hundreds of rows, while dispatch `2050/generators-p.csv` has ~8760.

## End-to-end picture

```text
Hourly wind CF CSV
    → load into generators_t.p_max_pu
    → × efficiency (losses)
    → tile across investment years
    → CARPE_DIEM: cluster days using load + wind + solar;
         weight & rescale so annual wind energy is preserved
    → solve_planning: build fleet under p ≤ p_nom × CF (rep days)
    → create_dispatch_network: fix capacities, restore full hours
    → solve_dispatch: operate fleet under full hourly wind CF
```

## What this is / isn’t

**Is:** deterministic chronological wind availability from input profiles; planning approximates the year with wind-aware representative days; dispatch re-simulates full hourly variability on the built fleet.

**Isn’t:** stochastic scenarios, forecast-error reserves, or endogenous wind resource modeling. Variability is only as good as the input CF series and the representative-day compression.

## Related files

- Input: `example/model/minimal_model/generators-p_max_pu.csv`
- Load: `pypsa_canada/workflow/scripts/load_network.py`
- Loss factor: `apply_wind_loss_factors` in `pypsa_canada/workflow/scripts/add_components.py`
- Multi-year tiling: `pypsa_canada/workflow/scripts/add_snapshots.py`
- Clustering: `pypsa_canada/workflow/scripts/add_representative_days.py`, `representative_days/carpe_diem.py`
- Capacity credit: `example/data/constraints/capacity_values_placeholder.csv`
- Config: `snapshots` section in `example/config/minimal_model.yaml`
