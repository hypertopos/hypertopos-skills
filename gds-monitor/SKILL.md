---
name: gds-monitor
description: GDS sphere health monitoring — drift detection, regime change tracking, alert response, and periodic health assessment. Use when user asks to "check sphere health", "monitor drift", "are there any alerts", "what changed since last time", "detect regime changes", "find emerging anomalies", "temporal health check", or "is the sphere healthy". Requires hypertopos MCP server with temporal data.
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.6.0
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
