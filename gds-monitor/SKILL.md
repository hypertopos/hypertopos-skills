---
name: gds-monitor
description: GDS sphere health monitoring — drift detection, regime change tracking, alert response, and periodic health assessment. Use when user asks to "check sphere health", "monitor drift", "are there any alerts", "what changed since last time", "detect regime changes", "find emerging anomalies", "temporal health check", or "is the sphere healthy". Requires hypertopos MCP server with temporal data.
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.7.1
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

## Responding to alerts

| Alert | What it means and what to do |
|---|---|
| `anomaly_rate_spike` | `anomaly_summary(pattern_id)` then `contrast_populations({anomaly: true})` — what spiked? If edge table available: `degree_velocity` on top spikers to check connection rate acceleration. |
| `population_size_shock` | `get_sphere_info()` — compare entity counts — data ingestion issue? |
| `theta_miscalibration` | `recalibrate(pattern_id)` then re-check `sphere_overview()` |
| `regime_changepoint` | `find_regime_changes(pattern_id)` then `compare_time_windows()` for flagged period |
| `calibration_drift` | Note for next rebuild. Current results still usable. |

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
          time_from=period_a_start, time_to=period_a_end, limit=20)
aggregate(event_pattern, group_by_line=anchor_line, metric="count",
          time_from=period_b_start, time_to=period_b_end, limit=20)
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

**Drift direction triage.** `find_drifting_entities` now returns `drift_direction` per entity. On a monitoring sweep, triage thus: `deteriorating` (highest priority, entities moving away from the population centre — trending toward anomaly), `neutral` (lateral motion, monitor), `normalizing` (lowest priority, already self-correcting). Filter the top-N list by `drift_direction == "deteriorating"` before escalating, otherwise alerts include entities that are actively recovering.

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

After a builder rebuild, `compare_calibrations(pattern_id)` (default args)
returns the drift between the two most recent epochs. Use `top_n=N` to
limit the ranked dimension list, `verbose=True` to also get the full
per-dimension breakdown, or pass explicit `v_from` / `v_to` to compare any
two epochs the sphere has on disk (`list_calibration_versions(pid)` returns
what's available).

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

The full per-percentile sweep lives behind `theta_sensitivity(pattern_id)`
— use that when the summary flags a non-trivial structure (cliffs > 0
or stable_band_length < 6) and you need the exact ratios to decide on
a safe recalibration move.

---

## Dim-quality warnings

`sphere_overview()` carries an optional `dim_quality_warnings[]` block per pattern surfacing silent build-time failure modes that break z-score / `delta_norm` semantics. These were previously invisible at agent runtime — the dim sat in the delta vector contributing nothing or contributing wrong signal, and the investigator had no way to know without scrolling the calibration log.

| `type` | Trigger | Why it breaks `delta_norm` |
|---|---|---|
| `dead_dim` | `sigma_diag[i] < 1e-10` (zero variance across population) | z-score `(x - mu) / sigma` is undefined / explodes; the dim contributes nothing meaningful and silently dilutes other dims' signal |
| `sparse_dim` | `dim_percentiles[d]['p50'] == 0` AND `p99 > 0` (mostly-zero with rare nonzero) | gaussian z-score assumption is wrong — empirical distribution is point-mass-at-zero plus a tail; Bregman divergence with poisson / bernoulli kind tag is the correct distance |
| `dominant_dim_mass` | pattern-level: one dim's share of `((p99 − μ_d) / σ_d)²` across dims ≥ 0.70 | pattern's `delta_norm` ranking is effectively single-dim; the sphere is a one-dim detector on this pattern, multi-dim composition adds little |
| `negative_space` | per-dim: `dim_percentiles[d]['p50'] == 0` AND `dimension_kinds[i] == "gaussian"` | gaussian z-score is wrong — empirical distribution is point-mass-at-zero, not centered on the mode; gaussian semantics treat zero as the typical value when it is actually the absence value |
| `heteroscedasticity` | pattern-level: Brown-Forsythe (median-centred Levene) `p < 0.01` on `delta_norm` partitioned by `group_by_property` | global θ assumption violated for this pattern; `delta_norm` variance differs sharply across the levels of the grouping variable, so a single global threshold produces unequal false-positive rates per group |
| `non_normal_dim` | per-dim: Shapiro-Wilk (N ≤ 5000) or Kolmogorov-Smirnov (larger N) `p < 0.01` on a `kind='gaussian'` dim | gaussian z-score is wrong — empirical distribution is heavy-tailed, so `(x − μ) / σ` produces unequal false-positive rates per quantile of the dim; mu/sigma are stable but the percentile semantics are not |

Each warning carries `dim_label`, `reason` (the offending value), and `advice` (concrete remediation); `dominant_dim_mass` and `heteroscedasticity` additionally carry `evidence_value` (the firing metric — share for dominant_dim_mass, p-value for heteroscedasticity) + `threshold` (the cutoff) so the agent can rank patterns by severity. Note: the `heteroscedasticity` warning's `dim_label` is the GROUPING VARIABLE name (a categorical line property), not a δ-dim — unlike the per-dim auditors. Triage rules:

**Pattern-level calibration auditors** — `dominant_dim_mass`, `negative_space`, and `heteroscedasticity` are sphere-design-quality signals; if they fire, raise a calibration ticket rather than chasing the entities flagged by `find_anomalies` on the affected pattern — those entities are likely noise.

- **Any `dead_dim` warning**: drop the dim from the pattern, OR investigate the data source for missing values / constant column. Builder-time issue — won't fix itself on rebuild without a YAML edit.
- **`sparse_dim` warning on a binary-like signal** (e.g. structuring flag, fraud label): switch to Bregman with `kind: bernoulli`. The warning is not a bug — it's the gaussian assumption being wrong for the data shape.
- **`sparse_dim` warning on a sparse counter** (e.g. tx-count where most accounts have zero): split into `is_active` (bernoulli) + `tx_count_when_active` (poisson) dims. The single-dim representation conflates two regimes.
- **`dominant_dim_mass` warning**: cross-check per-polygon `reliability_flags.single_dim_driven` incidence on `find_anomalies` top-N for the same pattern — pattern-level dominance ⟹ high per-polygon incidence agreement. Raise a calibration ticket; consider splitting the dominant dim into its own pattern or re-weighting via `dimension_weights` so the other dims actually contribute.
- **`negative_space` warning**: re-declare the dim with `kind='bernoulli'` (presence/absence) or `kind='poisson'` (count), or split into a binary `is_active` dim plus a continuous `value_when_active` dim. Same remediation family as `sparse_dim` but the trigger is gaussian-declared, not empirical-distribution-shaped.
- **`heteroscedasticity` warning**: per-group θ calibration is statistically warranted on the named grouping — the pattern already carries it via `group_by_property`, so the warning is confirmation, not a prescription for new work. If per-group θ is undesirable downstream (e.g. cross-group comparison of anomaly scores), apply a variance-stabilizing transform — `log1p` on `delta_norm` before thresholding — to make scores comparable across groups. Re-check after rebuild: variance-stabilizing should push the Levene p-value back above 0.01.
- **`non_normal_dim` warning**: the empirical distribution of a gaussian-declared dim is heavy-tailed — z-score percentiles are unstable. Remediation in priority order: (a) re-declare kind as `'poisson'` (counts) or `'bernoulli'` (binary) if the data shape fits; (b) apply variance-stabilizing transform in the column source script (`log1p` for skewed positive, `sqrt` for right-skewed) and rebuild; (c) accept the warning if the dim is rarely top-of-rank — non-normal dims rarely dominate `delta_norm` rankings on their own. Composes with `negative_space`: if `negative_space` already fires on the same dim, `non_normal_dim` is suppressed (the kind itself is the bug, not the empirical departure).
- **No `dim_quality_warnings` block**: pattern passed all five checks. Don't infer "perfectly healthy" — the warnings only catch the five specific silent-failure modes; other quality issues (correlated dims, stale data, drifted calibration) need separate diagnostics.

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
crossed yet. Only act on `reliability=high` forecasts.

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

If `compare_calibrations` between two epochs shows large RMS drift on `mu` or `sigma` and you don't have an obvious data-source change to explain it, run `find_calibration_influencers` to see WHICH entities had highest impact on the new calibration. The hidden-influencer cell often catches data-quality regressions (duplicated records, miscategorised entities) that warped μ/σ across the population.

```
mcp__hypertopos__find_calibration_influencers(
    pattern_id="<pattern>",
    classify="all",
    top_n=10,
)
```

Inspect the `cell_counts` distribution — a healthy population is dominated by `"normal"`. A spike in `"hidden"` or `"distorter"` per epoch is a signal that recalibration was driven by a small subset, not population-wide drift.
