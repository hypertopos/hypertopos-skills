# FDR Controls

Per-dimension FDR, per-dim ranking and multi-resolution (spatial × temporal) FDR for detection recipes.

## FDR axis and per-dim ranking

**FDR axis selection** — `fdr_axis` selects what FDR is controlled over. The default `"entity"` controls the false-discovery rate over per-entity anomaly tests on `||delta||²` (one p-value per entity, df = dimensionality) — use it when the question is "which entities are jointly anomalous across the whole shape vector". The `"per_dim"` axis runs an independent BH/Storey correction per dimension on chi²(1) univariate p-values for each cell of the delta matrix and keeps an entity iff *any* dim's q-value clears alpha — use it when the question is "which dimension is anomalous for this entity", because a single dim driving many discoveries no longer inflates the threshold for unrelated dims (so a rare-but-real single-dim signal that would be lost in a joint norm test surfaces). The `"both"` axis requires the entity to survive *both* tests — strictest, lowest false-positive rate, smallest candidate list. Per-dim mode attaches `q_values_per_dim` (a `{dim: q}` map of the dims with q < 1), `min_q_per_dim` (the smallest), and `dominant_q_dim` (the name of that smallest dim) to every returned polygon — investigator drilldown can read which dim drove each survivor straight from the result.

**Ranking by per-dim significance** — `rank_by="min_q_per_dim"` re-ranks the post-FDR survivors by smallest per-dim q-value ascending (most extreme single-dim signal first), instead of the default `rank_by="delta_norm"` which sorts by joint norm descending. Use when an entity with one knife-edge dimension matters more to the investigation than an entity with diffuse moderate elevation across many dims. Requires `fdr_alpha` set and `fdr_axis ∈ {"per_dim", "both"}` (no q-values otherwise); incompatible with `select="diverse"` (which has its own ordering). Note: chi²(1) is direction-agnostic — both extreme-positive and extreme-negative deviations produce small p-values — so on patterns where some dims are *anti-signal* (high `|delta|` correlated with non-target class, e.g. legitimate-traffic outliers in fraud detection), per-dim FDR will flag both wings of those dims. Mitigation when labels exist: use `engine.dim_audit.compute_per_dim_label_auroc` to identify anti-signal dims and silence them via `dimension_weights={dim: 0.0}` before ranking.

## Multi-resolution FDR: spatial × temporal hierarchies

When the pattern declares `fdr_hierarchy:` (spatial) and/or `fdr_temporal_hierarchy:` (temporal) in sphere.yaml, `find_anomalies` accepts two extra params:

- `fdr_resolution: "<spatial_level_name>"` — gates returned anomalies to entities whose cell at this level cleared per-level BH/Storey FDR at `fdr_alpha`.
- `fdr_temporal_resolution: "<temporal_level_name>"` — same, for the temporal axis.

Both can be combined: intersection-FDR — an entity survives iff its cell cleared every named level on BOTH axes.

### Cell test stat

Per-cell evidence is a Fisher exact 2×2 upper-tail p-value comparing the cell's anomaly rate against the rest of the population. Reads `is_anomaly` boolean column on geometry. Non-parametric, exact, copes with low-count cells (cells with few entities behave correctly — a z-test would break here).

### Per-level Tippett aggregation

A coarse-level cell's evidence is the *minimum* p-value among its child cells (Tippett combination). This is the standard hierarchical-test composition rule and is conservative: coarse levels reject only when at least one child level fires very strongly. Per-level BH/Storey then applies to the coarse-cell p-value set.

### When to use

- `fdr_resolution` — population-level anomaly rate jumped in a sub-group (per-bank, per-region, per-community). Surfaces "this bank is suspicious" rather than "these accounts are suspicious".
- `fdr_temporal_resolution` — epoch-level shifts the entity-axis test cannot see (a time window has elevated anomaly density even though individual entities pass entity-FDR).
- Both — strictest filter, useful for finding the cells where risk concentrates at the latest tag.

### What this filter does and does NOT do

Multi-resolution FDR is a *per-cell* filter on the geographic / temporal axis. It localises anomalies to a hot-spot region and provides `cell_q_spatial` / `cell_q_temporal` / `cell_path` as pre-computed statistical evidence for SAR rationale. It does NOT tighten the entity-level ranking inside surviving cells. Survivor count and per-K precision depend on how the cells partition the population, not directly on `alpha`.

Two failure modes to watch for:
- **Single-cell levels collapse to no contrast.** A hierarchy level that resolves to one unique value across the whole sphere has p=1.0 by construction (single cell == whole population), so it rejects nothing. Check `temporal_bucket` granularity against the data's actual time range: a 90-day bucket on a 50-day sphere is one cell.
- **Intersection returns zero survivors when one axis is degenerate.** Drop to single-axis mode (`fdr_resolution` only, or `fdr_temporal_resolution` only) before raising `alpha`.

### Entity-axis FDR auto-upgrades when `fdr_resolution` is set

When `fdr_resolution` or `fdr_temporal_resolution` is set on the entity axis and `fdr_method` is left unset, it resolves to `"storey"` instead of `"bh"` (entity-level p-values are always `chi2`). An explicit `fdr_method` passes through unchanged; `fdr_axis="per_dim"` / `"both"` use `chi²(1)` per-dim, so no upgrade happens there.

The user-visible surface is therefore: setting `fdr_resolution=<level>` is enough to get both the cell-level gate AND the workload-correct entity ranking. No manual combo needed.

### Anti-signal direction

Like `fdr_axis="per_dim"`, this surface is direction-agnostic: an anomaly-rate DROP in a cell will not flag (upper-tail test), but an anomaly-rate SPIKE will. This is the intended direction for fraud / outlier investigation.

### Example invocation

```python
# Spatial gate — entity-axis FDR auto-upgrades to chi2+Storey under
# the hood when fdr_resolution is set
find_anomalies(pattern_id="account_pattern", top_n=20, fdr_alpha=0.05,
               fdr_resolution="bank")

# Intersection — both axes named; same auto-upgrade applies. Only
# meaningful when both hierarchies are non-degenerate.
find_anomalies(pattern_id="account_pattern", top_n=20, fdr_alpha=0.05,
               fdr_resolution="bank", fdr_temporal_resolution="quarter")

# Power-user override — pass fdr_method explicitly to bypass the Storey default
find_anomalies(pattern_id="account_pattern", top_n=20, fdr_alpha=0.05,
               fdr_resolution="bank", fdr_method="bh")
```
