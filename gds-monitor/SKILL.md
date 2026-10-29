---
name: gds-monitor
description: GDS sphere health monitoring — drift detection, regime change tracking, alert response, and periodic health assessment. Use when user asks to "check sphere health", "monitor drift", "are there any alerts", "what changed since last time", "detect regime changes", "find emerging anomalies", "temporal health check", or "is the sphere healthy". Requires hypertopos MCP server with temporal data.
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.10.1
  mcp-server: hypertopos
---

# GDS Monitor

A GDS monitor detects behavioral changes over time — drift at the entity
level, regime shifts at the population level, emerging anomalies not yet
crossed the threshold, and calibration degradation. The approach is
top-down: check population health first, drill to entities only when
something is flagged.

**Prerequisite:** This skill requires patterns with temporal data (`has_temporal: true`
in sphere_overview). If no patterns have temporal data, report that temporal monitoring
is unavailable and hand off to gds-explorer for static analysis instead.

For concrete tool output examples, see [references/examples.md](references/examples.md).

---

## Monitoring checklist

- [ ] On a long-lived session, `snapshot.stale` checked and the sphere reopened when stale
- [ ] `sphere_overview` and `check_alerts` run; every alert answered
- [ ] `find_regime_changes` and `find_drifting_entities` run even when alerts are clean
- [ ] Dim-quality warnings reviewed
- [ ] Findings handed off to the right skill

## Quick health check

Two calls tell you whether anything needs attention:

```
sphere_overview(detail="summary")  -> calibration, anomaly rates, profiling_alerts, has_temporal
check_alerts()                     -> alerts sorted by severity
edge_stats(pattern_id)             -> if event patterns exist, graph monitoring available
```

If alerts are present, respond to them (see table below). **If `check_alerts`
returns nothing, proceed to `find_regime_changes` and `find_drifting_entities`
anyway** — absence of alerts does NOT mean absence of temporal patterns.
Alerts check geometric health (calibration staleness, threshold violations);
regime changes and drift detect behavioral shifts that are structurally valid
but operationally significant. Skipping temporal tools when alerts are clean
is the most common monitoring gap.

**Re-checks in a long-lived session read a pinned snapshot.** A session reads
the sphere as it was when `open_sphere` ran; ingests, rebuilds and
recalibrations made by other sessions or processes afterwards stay invisible until you
reopen. Before a periodic re-check, call `get_session_stats()` and read its
`snapshot` block: `snapshot.stale == true` means a newer sphere state exists —
call `open_sphere(path)` to read it, then run the health check. The session's
own `recalibrate` is visible at once, with no reopen. (`snapshot.stale` is about
sphere state; `vector_index_health.is_stale` below is about ANN index coverage.)

## Responding to alerts

| Alert | What it means and what to do |
|---|---|
| `anomaly_rate_spike` | `anomaly_summary(pattern_id)` then `contrast_populations({anomaly: true})` — what spiked? If edge table available: `degree_velocity` on top spikers to check connection rate acceleration. |
| `population_size_shock` | `get_sphere_info()` — compare entity counts — data ingestion issue? |
| `theta_miscalibration` | `recalibrate(pattern_id)` then re-check `sphere_overview()`. The recalibration records a new calibration epoch, so `compare_calibrations(pattern_id)` shows what moved; to try it without touching main, use the branch workflow below |
| `regime_changepoint` | `find_regime_changes(pattern_id)` then `compare_time_windows()` for flagged period |
| `calibration_drift` | Note for next rebuild. Current results still usable. |
| `calibration_age` | The pattern's calibration reference is older than 60 days; its `conformal_p` (and so the alarm budget behind `is_anomaly`) drifts as the population moves. `recalibrate(pattern_id)` — on a branch first if the change needs review — then `compare_calibrations(pattern_id)` to see what moved. |
| `alarm_rate_drift` | Among rows written since the last calibration, the flagged count exceeds the alarm budget (one-sided beta-binomial p < 0.01, which allows for the calibration's own noise): the population has moved from its reference. Recalibrate — ideally on a branch first, compare alarms and rankings with `compare_calibrations(pattern_id)`, then promote. `check_alerts` `details` carries `n_new`, `flagged`, `expected`, `p_value`. Not raised on per-group calibrated patterns or on spheres built before new rows were calibrated out of fold (rebuild those). |
| `stale_vector_index` | Incrementally-appended geometry rows sit outside the ANN (IVF) index, so trajectory / similarity tools (`detect_trajectory_anomaly`, `find_similar_entities`, `find_drifting_similar`) silently miss them. MEDIUM severity. Drill down with the `vector_index_health` block (see below) to read the unindexed fraction, then trigger a reindex on the builder side. |

### `stale_vector_index` drilldown — reading `vector_index_health`

The alert is metadata-only and fires when the unindexed fraction of a pattern's geometry exceeds the stale threshold (default 0.1). There is no standalone health tool — the per-pattern detail is surfaced as a `vector_index_health` block on every `audit_pattern_dims` response:

```
audit_pattern_dims(pattern_id) -> vector_index_health block:
  { index_present, index_type, num_indexed_rows, num_unindexed_rows,
    total_rows, indexed_fraction, num_partitions, is_stale,
    stale_threshold, recommendation }
```

Triage:

- **`is_stale == false`** — index covers the population; ANN-backed tools are complete. No action.
- **`is_stale == true`** — the `recommendation` string names the fix; an unindexed run accumulated from incremental ingest. ANN tools under-report on the unindexed tail until a reindex runs. Note it as a monitoring finding and hand the reindex to the sphere owner (it is a build-side operation — see the incremental-ingest path in **gds-sphere-designer**, which reindexes via `incremental_update(reindex=True)` or `finalize_incremental()`).
- **`index_present == false` with a small population** — expected. The builder skips the IVF index below the build-time row floor; ANN tools fall back to a full scan and lose nothing. Not an actionable alert.

Interpret `is_stale` as "ANN results are incomplete", NOT "the geometry is wrong" — the rows exist and exact-scan tools (`find_anomalies`, `explain_anomaly`) see them; only the approximate-neighbour tools are affected.

---

## Event rate divergence

Compare event anomaly rate per entity vs static anchor anomaly status:

```
aggregate(event_pattern, group_by_line=anchor_line,
          geometry_filters={"is_anomaly": true})
```

Entities with >15% anomalous events but NOT flagged as anchor anomalies
have concentrated temporal anomalies — investigate with drift detection.

**Shortcut:** `sphere_overview(detail="full")` returns `event_rate_divergence_alerts`
pre-computed for all anchor patterns (top 20 by rate).
**Caution:** `detail="full"` can take **minutes** on large spheres (>100K entities) —
use only for targeted diagnostics, never in interactive loops.

## Windowed volume comparison

Compare event counts per entity across two time periods to detect bursts or drops.
Use `time_from`/`time_to` on `aggregate` — ISO-8601 half-open interval `[from, to)`.
Pick periods from sphere context (recent batch window, pre/post incident, quarter).

```
aggregate(event_pattern, group_by_line=anchor_line, metric="count",
          time_from=period_a_start, time_to=period_a_end, top_n=20)
aggregate(event_pattern, group_by_line=anchor_line, metric="count",
          time_from=period_b_start, time_to=period_b_end, top_n=20)
```

Entities with 2x+ count increase in period B are burst candidates (order splitting,
seasonal spikes, data ingestion anomalies). Requires `timestamp_col` on the event
pattern (set via `temporal` config in sphere.yaml).

---

## Drift detection

```
find_drifting_entities(pattern_id, top_n=10, forecast_horizon=3)
```

### Interpreting drift results

- Single high-displacement drifter = entity-level behavioral change
- Cluster of drifters in same direction = systemic population shift
- `forecast_anomaly=true` + `reliability=high` = early warning

**Drift direction triage.** `find_drifting_entities` now returns `drift_direction` per entity. On a monitoring sweep, triage thus: `deteriorating` (highest priority, entities moving away from the population centre — trending toward anomaly), `neutral` (lateral motion, monitor), `normalizing` (lowest priority, already self-correcting). Filter the top-N list by `drift_direction == "deteriorating"` before escalating, otherwise alerts include entities that are actively recovering. Windows without events are left out of the trajectory (`num_inactive_slices`), so a busy-then-silent entity has no drift — look for it with `find_anomalies`, not here.

For top drifters, `dive_solid(key, pattern_id)` reveals whether it is
gradual drift or a sudden jump. If edge table available: `degree_velocity(key, pattern_id)` — accelerating degree alongside geometric displacement = strong behavioral change confirmation.

### Dimension-specific drift

Displacement is a single number — it hides WHICH dimension changed. For
top drifters:

```
dive_solid(key, pattern_id) -> compare first vs last slice deltas
-> Which dimension changed most? (e.g., counterparty count spike = new relationships,
   burst metric spike = activity change, value metric spike = monetary shift)
```

If top drifters ALL show the same dominant dimension (e.g. burst_monthly),
other dimensions' real changes are invisible in the ranking. In that case:

- For each NON-dominant dimension: check if ANY drifter has significant
  change on THAT specific dimension
- `dive_solid` on 3 random high-displacement entities, compare first vs
  last slice PER DIMENSION separately — not just total displacement
- A drifter with small total displacement but large change on ONE dim
  is more interesting than a drifter with large displacement on a
  high-variance dimension

**Tool shortcut — `rank_by_dimension`:**

```
find_drifting_entities(pattern_id, rank_by_dimension="suppliers")
-> entities ranked by drift on the "suppliers" dimension specifically,
   not total displacement
```

Use this when you suspect a specific dimension is the signal but total
displacement is dominated by a different high-variance dimension.

---

## Compare calibration epochs to inspect drift

After a builder rebuild or a `recalibrate` (each writes a new epoch), `compare_calibrations(pattern_id)` (default args)
returns the drift between the two most recent epochs. Use `top_n=N` to
limit the ranked dimension list, `verbose=True` to also get the full
per-dimension breakdown, or pass explicit `v_from` / `v_to` to compare any
two epochs the sphere has on disk (epoch integers resolved automatically;
omit both to default to the most recent pair).

Example query: "Has the population shifted since the last rebuild?" →
`compare_calibrations("account_pattern")` → check `overall_drift_rms`
(RMS in σ units, comparable across patterns) and `top_drifted` (which
dimensions moved the most).

When the anchor pattern declares `edge_dim_aggregations:`, the report also
carries `edge_dim_threshold_drift` — a per-source-dim `{from, to, delta}`
map of the `_count_above_threshold` cutoff. A large delta means the
population p95 of that source dim shifted between epochs, so the same
agent narrative ("X anomalous on `_count_above_threshold` = +Nσ") does
NOT mean the same edge regime as in the previous epoch. Worth flagging
when it shows up in monitoring alongside a non-zero `overall_drift_rms`.

### Verdict composer — `calibration_drift_report`

`compare_calibrations` returns the raw per-dimension μ/σ/θ drift table but leaves
the "is this drift large enough to distrust cross-epoch reasoning?" call to the
agent. For a monitoring sweep, prefer `calibration_drift_report(pattern_id)` — it
wraps `compare_calibrations` and adds a `drift_verdict` over `overall_drift_rms`
plus a routing recommendation:

```
calibration_drift_report(pattern_id)  # default args = latest vs previous epoch
-> drift_verdict: "stable" | "moderate" | "significant"
-> top_drifted, interpretation, recommended_next_steps
```

Monitoring routing by verdict:

- **`stable`** — the coordinate system held; a "still anomalous" / "no longer
  anomalous" comparison across these epochs reflects the entity, not
  recalibration. Cross-epoch entity verdicts are safe to report.
- **`moderate`** — some dimensions moved materially; treat cross-epoch verdicts
  on the `top_drifted` dimensions with caution before escalating an entity.
- **`significant`** — the coordinate system shifted substantially; a cross-epoch
  "no longer anomalous" claim is dominated by recalibration. Do NOT compare
  entity verdicts as if θ were fixed. Use `decompose_drift(entity, pattern)` to
  split per-entity change into intrinsic (entity moved) vs extrinsic
  (population recalibrated around it), and pair with
  `find_calibration_influencers` (see end of this skill) to find which entities
  drove the shift.

Drop to raw `compare_calibrations` only when you need the full per-dimension
table without the interpreted verdict, or to compare two specific non-adjacent
epochs (`calibration_a` / `calibration_b`).

---

## Safe recalibration via branch

When proposing a recalibration — especially after excluding calibration influencers — run it on an isolated branch first. This leaves main untouched until you confirm the result is the intended calibration.

```
# 1. Recalibrate on a branch (optionally excluding known influencers)
recalibrate(pattern_id, branch="my_branch", exclude_keys=["KEY1", "KEY2"])

# 2. Compare the branch to main (from the main session)
compare_branches(pattern_id, branch="my_branch")
  -> anomalies_from/to, flipped_in/flipped_out, flipped_in_top/flipped_out_top,
     jaccard, top_movers, calibration_drift
#    To drill into the branch, open_sphere(path, branch="my_branch"); reopen
#    main with open_sphere(path) before promoting or deleting it.

# 3a. Calibration looks correct — promote to main (replays recipes, verifies, deletes branch)
manage_branch(action="promote", name="my_branch")

# 3b. Calibration looks wrong — discard the branch
manage_branch(action="delete", name="my_branch")
```

Use `compare_branches` to answer: which entities flipped anomaly status (`flipped_in_top` / `flipped_out_top` name the ones furthest past the threshold), how large is the Jaccard overlap between anomaly sets, and how far the fit moved (`calibration_drift.overall_drift_rms`, in σ units — the scale `calibration_drift_report` grades as stable / moderate / significant). Use `manage_branch(action="list")` to see all open branches before starting a new one.

`promote` refuses when main has moved since the branch was created, and aborts (restoring main) if the replayed fit does not match the branch exactly — so it is safe to call without manual verification.

---

## Threshold sensitivity at a glance

`sphere_overview()` carries a `theta_sensitivity_summary` block per
pattern (when populated by a recent builder rebuild). The block tells
the agent — without a per-pattern drilldown — whether the chosen
`anomaly_percentile` is in a smooth region of the population's
`delta_norm` distribution or perched near a heavy-tail jump.

Fields per pattern:

- `stable_band_from` / `stable_band_to` / `stable_band_length` — the
  longest contiguous run of percentiles where adjacent-pair
  `theta_mean` ratio stays below 1.30. Within this band, recalibrating
  to a different percentile shifts the threshold by less than 30 % per
  step.
- `n_cliffs` — number of percentile boundaries (in `p90 .. p99`)
  where the `theta_mean` ratio is at or above 1.50. A single move
  across a cliff jumps the threshold by 50 % or more — heavy-tail
  region of the distribution.
- `theta_at_p95` — the production-default p95 threshold value for
  quick visual reference.

Quick triage rules:

- **`stable_band_length >= 8` AND `n_cliffs == 0`**: pattern is
  light-tail and the chosen percentile sits comfortably. Recalibration
  moves anywhere in the band are safe.
- **`stable_band_length <= 4` OR `n_cliffs >= 2`**: heavy-tail
  pattern. Production p95 may already sit on a cliff edge. Call
  `theta_sensitivity(pattern_id)` for the full per-percentile sweep
  and inspect the `cliffs[]` list before any recalibration proposal.
- **p95 inside a cliff pair** (e.g. `cliffs[0]['from'] == 'p95'`):
  do not recalibrate up — moving to p96 jumps theta materially.
  Consider tightening the percentile down (`p93` or below) instead.
- **Empty `theta_sensitivity_summary`**: the sphere predates the
  diagnostic. Trigger a rebuild before relying on this surface.

When the `theta_sensitivity_summary` block flags a non-trivial structure
(cliffs > 0 or stable_band_length < 6) and you are about to propose a
recalibration move, prefer the verdict composer over the raw sweep:

```
theta_sensitivity_report(pattern_id)
-> recalibration_safety: "safe" | "caution" | "unsafe"
-> n_cliffs, stable_band, cliffs, theta_sensitivity (per-percentile sweep)
-> interpretation, recommended_next_steps
```

`theta_sensitivity_report` wraps `theta_sensitivity` and reads the band / cliff
structure into a single `recalibration_safety` verdict, so you do not have to
interpret the raw band/cliff lists yourself:

- **`safe`** — a contiguous stable band exists and no cliffs; moving
  `anomaly_percentile` shifts the threshold smoothly. Recalibration anywhere in
  the swept range is safe.
- **`caution`** — a stable band exists but cliffs sit outside it; keep
  `anomaly_percentile` inside `stable_band` and avoid the boundaries in
  `cliffs[]`.
- **`unsafe`** — no stable band; every percentile step shifts θ materially
  (heavy-tail region). Do not recalibrate by percentile without re-measuring
  the flagged population.

Use raw `theta_sensitivity(pattern_id)` directly only when you want the
per-percentile ratios without the interpreted verdict.

---

## Dim-quality warnings

`sphere_overview()` carries an optional `dim_quality_warnings[]` block per pattern surfacing silent build-time failure modes that break z-score / `delta_norm` semantics. These were previously invisible at agent runtime — the dim sat in the delta vector contributing nothing or contributing wrong signal, and the investigator had no way to know without scrolling the calibration log.

| `type` | Trigger | Why it breaks `delta_norm` |
|---|---|---|
| `dead_dim` | `sigma_diag[i] < 1e-10` (zero variance across population) | z-score `(x - mu) / sigma` is undefined / explodes; the dim contributes nothing meaningful and silently dilutes other dims' signal |
| `sparse_dim` | `dim_percentiles[d]['p50'] == 0` AND `p99 > 0` (mostly-zero with rare nonzero) | gaussian z-score assumption is wrong — empirical distribution is point-mass-at-zero plus a tail; Bregman divergence with poisson / bernoulli kind tag is the correct distance |
| `dominant_dim_mass` | pattern-level: one dim's share of `((p99 − μ_d) / σ_d)²` across dims ≥ 0.70 | pattern's `delta_norm` ranking is effectively single-dim; the sphere is a one-dim detector on this pattern, multi-dim composition adds little |
| `negative_space` | per-dim: `dim_percentiles[d]['p50'] == 0` AND `dimension_kinds[i] == "gaussian"` | gaussian z-score is wrong — empirical distribution is point-mass-at-zero, not centered on the mode; gaussian semantics treat zero as the typical value when it is actually the absence value |
Each warning carries `type`, `dim_label` and `reason` (the offending value); the concrete remediation for each `type` is in the response's top-level `advice_by_warning_type[type]`. `dominant_dim_mass` additionally carries `evidence_value` (the firing metric — share) + `threshold` (the cutoff) so the agent can rank patterns by severity. Triage rules:

**Pattern-level calibration auditors** — `dominant_dim_mass` and `negative_space` are sphere-design-quality signals; if they fire, raise a calibration ticket rather than chasing the entities flagged by `find_anomalies` on the affected pattern — those entities are likely noise.

- **Any `dead_dim` warning**: drop the dim from the pattern, OR investigate the data source for missing values / constant column. Builder-time issue — won't fix itself on rebuild without a YAML edit.
- **`sparse_dim` warning on a binary-like signal** (e.g. structuring flag, fraud label): make the dimension `bernoulli` (a 0/1 column declared under `precomputed_dimensions` with `edge_max: 1` or `kind: bernoulli`) and read its Bregman divergence. The warning is not a bug — it's the gaussian assumption being wrong for the data shape.
- **`sparse_dim` warning on a sparse counter** (e.g. tx-count where most accounts have zero): split into `is_active` (bernoulli) + `tx_count_when_active` (poisson) dims. The single-dim representation conflates two regimes.
- **`dominant_dim_mass` warning**: cross-check per-polygon `reliability_flags.single_dim_driven` incidence on `find_anomalies` top-N for the same pattern — pattern-level dominance ⟹ high per-polygon incidence agreement. Raise a calibration ticket; consider splitting the dominant dim into its own pattern or re-weighting via `dimension_weights` so the other dims actually contribute.
- **`negative_space` warning**: re-declare the dim with `kind='bernoulli'` (presence/absence) or `kind='poisson'` (count), or split into a binary `is_active` dim plus a continuous `value_when_active` dim. Same remediation family as `sparse_dim` but the trigger is gaussian-declared, not empirical-distribution-shaped.
- **No `dim_quality_warnings` block**: pattern passed all four checks. Don't infer "perfectly healthy" — the warnings only catch the four specific silent-failure modes; other quality issues (correlated dims, stale data, drifted calibration) need separate diagnostics.

---

## Regime changes

```
find_regime_changes(pattern_id, n_regimes=5)
```

### Interpreting regime change results

- 1 unidirectional shift = regime change (external event, policy change)
- 2 shifts in opposite directions = oscillation (seasonal, cyclical)
- ALL entities dropping simultaneously = **data boundary artifact**, not real change
- Multiple small shifts = gradual evolution

Use `compare_time_windows(pattern_id, pre_start, pre_end, post_start, post_end)`
to identify which dimensions shifted and by how much.

---

## Emerging anomalies

```
find_anomalies(pattern_id, top_n=10, include_emerging=True)
```

Emerging entities are trending toward the anomaly boundary but haven't
crossed yet. Only act on `reliability=high` forecasts. On a calibrated
pattern the list is empty (`emerging_note` says why): the forecast tracks
`delta_norm` against θ, and θ does not decide the flag there. Watch the
p-value instead: re-run `find_anomalies` each tick and diff the keys.

If edge table available: `contagion_score_batch(emerging_keys, pattern_id)` — emerging entities with high neighborhood contamination are higher-priority early warnings than those with clean neighborhoods. Pass `timestamp_cutoff` (Unix seconds) to compare contagion at the previous monitor tick vs now — a jump between the two is a hard alert signal.

---

## Handing off to other skills

Monitor flags, investigator digs. When you find:

- Specific entity with `source_count >= 2` — hand to gds-investigator
- Sudden trajectory change on one entity — hand to gds-investigator
- Population-level shift — report as finding, no entity drill needed

---

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Using `detail="full"` by default | Start with `detail="summary"` — "full" adds I/O per pattern and can take **minutes** on >100K entities |
| Treating drift as anomaly | Drifters may still be within normal range — check is_anomaly |
| Binary geometry drift analysis | Binary geometry produces degenerate drift results — needs continuous geometry |
| Investigating low-magnitude drifters | Focus on top-N by displacement — low-magnitude drift is noise |

---

## When things don't work

- **Tool returns empty results** — try different parameters, wider sample,
  or a different pattern. Check that the pattern has temporal data.
- **Tool errors** — check pattern_id and version, verify sphere is open.
  An unknown parameter name is rejected; the error suggests the closest
  declared name when there is one ("did you mean 'top_n'?") — fix the
  name instead of dropping the argument.
  If the error persists, report it rather than retrying in a loop.
- **No anomalies found** — not every sphere has every anomaly type.
  Report "no signal detected" as a valid finding.
- **find_drifting_entities returns no drifters** — the population may be
  stable, or the pattern lacks sufficient temporal slices. Both are valid
  findings worth reporting.
- **find_regime_changes returns no changepoints** — the population geometry
  has been stable over the observed period. Report stability as a finding.
- **check_alerts returns no alerts** — geometric health is clean, but still
  run `find_regime_changes` and `find_drifting_entities` to check for
  behavioral shifts. No alerts + no drift + no regime changes = truly healthy.

---

## Skill delegation

| Need | Skill |
|---|---|
| Root cause tracing, entity 360, hypothesis testing | gds-investigator |
| Event rates, Simpson's, temporal bursts, drift recipes | gds-detective |
| Cross-pattern, neighbor, trajectory, segment scans | gds-scanner |
| Orientation, profiling, clustering | gds-explorer |

Full monitoring examples: [references/examples.md](references/examples.md)

## When alerts say population shifted unexpectedly

If `compare_calibrations` between two epochs shows large RMS drift on `mu` or `sigma` and you don't have an obvious data-source change to explain it, run `find_calibration_influencers` to see WHICH entities had highest impact on the new calibration. The hidden-influencer cell often catches data-quality regressions (duplicated records, miscategorised entities) that warped μ/σ across the population. Note: `find_calibration_influencers` applies only to patterns scored by `delta_norm` (no `score_dims`); it raises on calibrated patterns — use `explain_anomaly` there.

```
mcp__hypertopos__find_calibration_influencers(
    pattern_id="<pattern>",
    classify="all",
    top_n=10,
)
```

Inspect the `cell_counts` distribution — a healthy population is dominated by `"normal"`. A spike in `"hidden"` or `"distorter"` per epoch is a signal that recalibration was driven by a small subset, not population-wide drift.
