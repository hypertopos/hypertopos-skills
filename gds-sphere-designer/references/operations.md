# Build Operations

Build performance, CI verification, and incremental ingest, read from Phases 4 and 5 of the sphere designer skill.

### Performance notes

- **Source caching:** Tier 3 script results cached as parquet in `.cache/`. Second build skips script execution.
- **Parallel:** Sources load in parallel; geometry + temporal build per-pattern in parallel.
- **Batched derived dims:** All simple derived dims sharing the same FK computed in a single `group_by` call.
- **IVF_FLAT indices:** Both geometry and trajectory use IVF_FLAT -- builds in seconds. The trajectory ANN index is skipped automatically when a pattern has too few entities to train it; trajectory search then uses a correct brute-force scan, which is fast at that scale.

### CI / pre-deploy verification (no MCP session needed)

Three `hypertopos sphere` CLI verbs run directly against a built sphere
directory — no live MCP session — so they slot into a CI gate or a
pre-deploy script. All three accept `--json` for machine-readable output.

```bash
# 1. Structural integrity — does every declared line/pattern have its
#    on-disk directory, and (with --strict) is calibration healthy?
hypertopos sphere validate my_sphere/ --strict --json
#    Exits 0 when valid, 1 when invalid. --strict promotes
#    calibration_health "suspect"/"poor" and dim_quality_warnings from
#    warnings to errors, so a degraded build fails the gate.

# 2. Health check — composes sphere_overview + check_alerts into a single
#    status: "ok" (no alerts) / "warning" (MEDIUM) / "critical" (HIGH).
hypertopos sphere health my_sphere/ --exit-code-on-critical
#    With --exit-code-on-critical the process exits 2 when any HIGH-severity
#    alert fires, so `set -e` fails the deploy on a critical sphere.

# 3. Diff against the currently-deployed sphere — pattern inventory
#    (added / removed / common) + per-pattern calibration drift between
#    the latest epoch of each.
hypertopos sphere diff old_sphere/ new_sphere/ --json
#    Patterns whose calibration schema differs are marked not_comparable
#    rather than crashing. Use to catch an unintended pattern drop or a
#    large overall_drift_rms before promoting a rebuild to production.
```

Recommended gate order: `validate --strict` (structure + calibration must
pass) → `health --exit-code-on-critical` (no critical geometric alerts) →
`diff` against the live sphere (review inventory + drift before promote).
The `stale_vector_index` MEDIUM alert surfaced by `health` is handled by the
incremental-ingest reindex path in Phase 5b below.

### Phase 5b — Incremental ingest (add entities without a full rebuild)

When new or changed entities arrive but the pattern design is fixed, you do
not need a full `hypertopos build`. The `GDSBuilder` Python API updates one
pattern's geometry in place against the existing μ/σ/θ calibration, from the
Python API or the `hypertopos sphere ingest` CLI verb.

> **Stage risky ingests on a branch.** Pass `branch="staging"` (CLI:
> `--branch staging`) to apply the update to a sphere branch only; main is not
> touched. Inspect the result with `compare_branches`, then promote it
> (`hypertopos sphere branch promote my_sphere/ staging`, or
> `HyperSphere.promote_branch`) or delete the branch. A full
> `hypertopos build` is refused while the sphere has branches.

> **Unsupported pattern types.** `incremental_update` reconstructs geometry
> against the *global* μ/σ/θ, so it refuses a pattern calibrated per group
> (`group_by_property`), per cluster (`gmm_n_components`), or carrying an FDR
> hierarchy — those need per-group / per-cluster recalibration the incremental
> path cannot reproduce. Rebuild those with `hypertopos build` instead.
> Separately, a single pattern cannot declare `tracked_properties` together
> with `edge_dimensions` / `edge_dim_aggregations` (the build refuses the
> combination); split them across two patterns.

```python
import pyarrow as pa
from hypertopos.builder.builder import GDSBuilder

# Point the builder at the EXISTING built sphere directory.
builder = GDSBuilder(sphere_id="my_sphere", output_path="my_sphere/")

# changed_entities: an Arrow table with the pattern's primary_key column plus
# the columns the pattern's dimensions are derived from. New keys are appended;
# existing keys are updated in place. Every declared edge-dim-aggregation
# ({dim}_{agg}) and event-dimension value column MUST be present — omitting one
# now raises (it would otherwise z-score an absent 0.0 into a spurious delta and
# corrupt the entity's anomaly geometry). Relations / prop columns may be absent.
builder.incremental_update(
    pattern_id="entity_pattern",
    changed_entities=new_rows,        # pa.Table
    deleted_keys=None,                # or a list[str] of keys to remove
    recalibrate="auto",               # "auto" recalibrates only if drift crosses the soft threshold
    reindex=False,                    # see batched-ingest note below
)
```

Key parameters:

- **`recalibrate`** — `"auto"` (default) recalibrates μ/σ/θ only when the
  appended rows push calibration drift past the soft threshold; `"force"`
  always recalibrates; `"never"` keeps the existing coordinate system. Use
  `"never"` for small top-ups where you want appended entities scored against
  the current population, `"auto"` for ongoing ingestion.
- **`reindex`** — when `True`, rebuilds the ANN (IVF) vector index immediately
  so the appended rows are visible to ANN-backed navigation
  (`detect_trajectory_anomaly`, `find_similar_entities`, `find_drifting_similar`).
  When `False` (default), the index is rebuilt only once the unindexed fraction
  crosses ~10% — until then those rows are outside the index and the
  `stale_vector_index` alert (see gds-monitor) fires.

**Batched ingestion (many small appends):** pay the O(N) rank recompute and the
reindex once at the end instead of per append:

```python
for batch in incoming_batches:
    builder.incremental_update(
        pattern_id="entity_pattern",
        changed_entities=batch,
        recompute_ranks=False,        # defer the global delta_rank_pct recompute
    )
builder.finalize_incremental("entity_pattern")  # recompute ranks + reindex once
```

`finalize_incremental(pattern_id)` recomputes the global `delta_rank_pct`
percentile across the whole population (making every row standalone-correct
again) and rebuilds the IVF index so all appended rows are indexed. It is
idempotent and safe to call after `recompute_ranks=True` updates too. After a
session, run `hypertopos sphere health my_sphere/` — a clean `stale_vector_index`
status confirms the reindex took.

**When to full-rebuild instead:** if the pattern's dimension set, relations, or
the source schema changed, incremental update cannot reconstruct the new
geometry — edit the YAML and run `hypertopos build --force`. Incremental ingest
is for new rows under a fixed design, not for design changes.
