# Error Controls: the Alarm Budget and the Cell Gate

How much to trust a flag, and where flags concentrate. Two controls, for two different questions.

## The alarm budget (every calibrated pattern)

A calibrated pattern (non-empty `score_dims` in `get_sphere_info`) flags an entity when its `conformal_p` is at most `alpha = 1 − anomaly_percentile/100`. `conformal_p` is the conformal p-value against the fitted population: the fraction of the calibration reference scoring at least as high as the entity, so the flag carries a calibrated statement: **at the 95th percentile, at most 5 % of the population is flagged.** That is the false-alarm budget of the whole list — read it straight off `is_anomaly`; there is nothing to switch on.

What follows for an investigation:

- **Expected false alarms = budget × population.** On a population of N entities at the 95th percentile, about `0.05 × N` flags are what a clean population would produce. If `anomaly_summary` shows a rate close to the budget, most flags may be noise; a rate well above it means real excess.
- **Rank inside the list by evidence.** `find_anomalies` orders by `conformal_p` ascending, then `anomaly_score` descending; `explain_anomaly` gives `bits = −log2 conformal_p` and splits them over axes and dimensions. A smaller `conformal_p` is stronger evidence, but it is a rank, not a per-entity error rate.
- **There is no entity-level FDR.** The conformal p-values of the fitted population are its own ranks, so a BH / Storey cut over them cannot reject anything beyond the budget. `find_anomalies` has no `fdr_axis` or entity q-values; `fdr_alpha` alone raises an error.
- **A tighter list** = a higher `anomaly_percentile` in the pattern YAML (rebuild), or keep the entities whose `conformal_p` is below a smaller alpha — `assess_anomaly_certainty` sweeps `perturbation_alphas` for one entity.
- **Old references drift.** `check_alerts` raises `calibration_age` when a pattern's calibration is older than 60 days, and `alarm_rate_drift` when rows written since the calibration already exceed the budget; `recalibrate(pattern_id)` refits it before you lean on the budget.

Event, GMM and discrete patterns keep `delta_norm` against θ as the decision; their `conformal_p` is on `delta_norm`.

## The cell gate: where anomalies concentrate

When the pattern declares `fdr_hierarchy:` (spatial) and/or `fdr_temporal_hierarchy:` (temporal) in sphere.yaml, `find_anomalies` gates its result to the cells that hold more anomalies than their size predicts:

- `fdr_resolution: "<spatial_level_name>"` — keep entities whose cell at this level cleared per-level FDR at `fdr_alpha`.
- `fdr_temporal_resolution: "<temporal_level_name>"` — same, for the temporal axis.
- `fdr_alpha` is the level of this gate and requires one of the two; `fdr_method` is `"storey"` by default (`"bh"` for the stricter all-null assumption).

Both can be combined: intersection — an entity survives iff its cell cleared every named level on BOTH axes. Survivors carry `cell_q_spatial` / `cell_q_temporal` / `cell_path`.

### Cell test stat

Per-cell evidence is a hypergeometric (one-sided Fisher exact) upper-tail p-value comparing the cell's `is_anomaly` count with the rest of the population. Non-parametric, exact, copes with low-count cells (a z-test would break there). This tests a real hypothesis — "this region is enriched in flags" — which is why it keeps an FDR level while the entity list does not.

### Per-level Tippett aggregation

A coarse-level cell's evidence is the *minimum* p-value among its child cells (Tippett combination). This is the standard hierarchical-test composition rule and is conservative: coarse levels reject only when at least one child level fires very strongly. Per-level BH/Storey then applies to the coarse-cell p-value set.

### When to use

- `fdr_resolution` — the anomaly rate jumped in a sub-group (per-bank, per-region, per-community). Surfaces "this bank is suspicious" rather than "these accounts are suspicious".
- `fdr_temporal_resolution` — a time window holds elevated anomaly density even though no single entity stands out.
- Both — strictest filter, useful for finding the cells where risk concentrates at the latest tag.

### What this filter does and does NOT do

The cell gate localises anomalies to a hot-spot region and provides `cell_q_spatial` / `cell_q_temporal` / `cell_path` as pre-computed statistical evidence for SAR rationale. It does NOT tighten the ranking inside surviving cells. Survivor count depends on how the cells partition the population, not directly on `alpha`.

Two failure modes to watch for:
- **Single-cell levels collapse to no contrast.** A hierarchy level that resolves to one unique value across the whole sphere has p=1.0 by construction (single cell == whole population), so it rejects nothing. Check `temporal_bucket` granularity against the data's actual time range: a 90-day bucket on a 50-day sphere is one cell.
- **Intersection returns zero survivors when one axis is degenerate.** Drop to single-axis mode (`fdr_resolution` only, or `fdr_temporal_resolution` only) before raising `alpha`.

### Direction

The test is upper-tail: an anomaly-rate SPIKE in a cell flags, a DROP does not. This is the intended direction for fraud / outlier investigation.

### Example invocation

```python
# Spatial gate (Storey over cells by default)
find_anomalies(pattern_id="account_pattern", top_n=20, fdr_alpha=0.05,
               fdr_resolution="bank")

# Intersection — both axes named. Only meaningful when both hierarchies
# are non-degenerate.
find_anomalies(pattern_id="account_pattern", top_n=20, fdr_alpha=0.05,
               fdr_resolution="bank", fdr_temporal_resolution="quarter")

# Benjamini-Hochberg over cells instead of Storey
find_anomalies(pattern_id="account_pattern", top_n=20, fdr_alpha=0.05,
               fdr_resolution="bank", fdr_method="bh")
```
