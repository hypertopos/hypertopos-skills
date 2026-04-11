---
name: gds-detective
description: Detection recipes for GDS anomaly investigation — event anomaly rates, temporal burst detection, Simpson's paradox, collective drift, composite aggregation, multi-pattern triangulation, passive_scan confirmation. Use when investigating anomalies found by gds-investigator, when sphere_overview shows event_rate_divergence_alerts, when composite patterns need subgroup analysis, or when temporal windowed comparison is needed. Use this skill for ANY detection recipe, even if the user doesn't explicitly name the pattern type.
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.2.2
  mcp-server: hypertopos
---

# GDS Detective

A GDS detective applies proven detection recipes to surface anomaly types
that basic `find_anomalies` misses: joint deviations in event patterns,
Simpson's paradox in composites, temporal bursts, neighbor contamination,
trajectory shapes, collective drift, and cross-pattern discrepancies.

Each recipe below is a tested sequence of tool calls. Adapt parameters to
the sphere at hand — the shapes are universal, the thresholds are starting
points.

For concrete output examples of each recipe, see [references/examples.md](references/examples.md).

---

## Choosing a recipe

| Signal you see | Recipe to reach for |
|---|---|
| `event_rate_divergence_alerts` in sphere_overview | Event anomaly rate |
| Composite patterns in the sphere | Composite subgroup analysis |
| Spike in one time window | Temporal burst detection |
| Anomalous entity with normal neighbors | Neighbor contamination |
| Low displacement but many temporal slices | Trajectory shape analysis |
| High displacement + consistent direction | Collective drift |
| Entity flagged by one pattern but not another | Multi-pattern investigation |
| Need to confirm suspects from another skill | passive_scan confirmation |
| `profiling_alerts` mentions a dimension | Re-ranking by property |
| Need full cohort, not top-N | Exhaustive enumeration |
| Temporal range implausibly wide for the dataset | Temporal artifact check |
| Event patterns with movement dimensions (speed, distance, duration) | Physical bound validation |
| Tip/rate dimensions in event pattern | Payment type segmentation |
| Entity patterns with ID/vendor/source dimension | Unregistered entity check |

---

## Event anomaly rate

Catches multi-dimensional joint deviations where each column individually
looks normal. Run both calls:

```
aggregate(event_pattern, group_by_line=anchor_line,
          geometry_filters={"is_anomaly": true}, limit=50)
aggregate(event_pattern, group_by_line=anchor_line, limit=50)
```

For each entity in the first result, compute `rate = anomalous_count / total_count`.

- Entities with >15% rate are suspect (baseline is typically around 5%).
- `event_rate_divergence_alerts` only covers entities below theta — this
  approach catches ALL high-rate entities.
- List ALL entities above 15% rate in the report with exact rate.
- For entity lines <50K, sampling is unnecessary.

## Composite subgroup analysis (Simpson's paradox)

For each composite pattern, run with dual ranking:

```
aggregate_anomalies(composite_pattern, group_by="parent_key_col", limit=50)
  -> ranked by count of anomalous composites (default)
find_anomalies(composite_pattern, rank_by_property="avg_price_dim", limit=50)
  -> ranked by value metric (catches subgroup price inflation)
```

Parents from either ranking are suspect. Threshold: >=2 anomalous composites
(not >=5 — subgroup inflation typically affects only 2-3 categories).
Run both calls on each composite pattern independently.

Key insight: an entity can have NORMAL aggregate stats but inflate ONLY its
top few subgroup categories. The composite pattern captures per-subgroup
deviation that aggregate analysis misses.

## Temporal burst detection

When `sphere_overview(detail="full")` returns `event_rate_divergence_alerts`,
temporal windowed analysis helps localize the burst.

```
Get the temporal range from get_sphere_info or dive_solid timestamps.
You need min/max dates to pick yearly windows.

Split into yearly windows. Run one aggregate per year:
aggregate(event_pattern, group_by_line=anchor_line, metric="count",
          time_from="YYYY-01-01", time_to="YYYY+1-01-01", limit=50)
At least 3 consecutive years provide a reliable baseline.

Compare per-entity counts across years:
-> 2x+ count spike in one year vs adjacent = burst/splitting
-> Cross-reference with event_rate_divergence_alerts entity keys
-> Entities in BOTH alert list AND spike year = high confidence

If a spike year is found, drill down into quarters:
aggregate(event_pattern, group_by_line=anchor_line, metric="count",
          time_from="YYYY-01-01", time_to="YYYY-04-01", limit=50)
-> pinpoints burst to specific quarter
```

Yearly windows work better than halves — halves dilute single-year bursts.

**Windowed aggregate for temporal burst localization:** use
`aggregate(event_pattern_id, group_by_line, time_from=window_start, time_to=window_end)`
to compare event counts across time windows. Faster than `dive_solid` for
initial burst detection — `dive_solid` is entity-level temporal history while
windowed aggregate gives per-entity counts across the full population in one call.

If edge table available (`edge_stats` returns `has_edge_table: true`), `degree_velocity(key, pattern_id)` corroborates burst timing — accelerating degree alongside event spike = strong behavioral change.

## Neighbor contamination

A normal entity whose geometric neighbors are systematically anomalous.
The entity itself is NOT flagged — the signal is in the neighborhood.

> **Prefer `detect_neighbor_contamination`** if available — it uses inverted search
> (starts from anomalies, finds contaminated normals) which is more effective than
> the manual recipes below. If edge table available, `contagion_score(key, pattern_id)`
> or `contagion_score_batch(keys, pattern_id)` gives the exact anomalous/total neighbor
> ratio directly. Use manual recipes only as fallback when neither tool is available.
>
> **As-of reconstruction:** `contagion_score`, `contagion_score_batch`, and
> `degree_velocity` (as well as the other edge-table graph primitives) accept an
> optional `timestamp_cutoff` parameter (Unix seconds). When set, only edges with
> `timestamp <= cutoff` are considered — use this to reconstruct neighbor
> contamination or burst velocity as they looked on the day of a known incident.

**From anomalous outward** (cheaper, try first):

```
find_anomalies(pattern, top_n=50) -> get top anomalous entities
For 10-15 anomalous entities:
  find_similar_entities(entity_key, pattern, top_n=10)
Check each neighbor: is it NORMAL (is_anomaly=false)?
A normal entity surrounded by anomalous entities = contamination target.
For each normal neighbor found:
  find_similar_entities(normal_key, pattern, top_n=10)
  -> check_anomaly_batch on ITS neighbors
  -> if >50% of its neighbors are anomalous: confirmed target
```

**From population sample** (broader, use if the first approach finds nothing):

```
Sample 50 normal entities (delta_rank_pct 30-70, is_anomaly=false)
For each: find_similar_entities(key, pattern, top_n=10)
check_anomaly_batch(neighbor_keys, pattern)
Normal entity with >50% anomalous neighbors = contamination target
```

This detects entities that are individually normal but positioned in
anomalous geometric neighborhoods — invisible to any single-entity scan.
The target is the NORMAL entity, not the anomalous neighbors.

## Trajectory shape analysis

> **Prefer `detect_trajectory_anomaly`** if available — it performs a full temporal
> scan and ranks by `wasted_motion`, directly targeting non-linear trajectories.
> Use the manual recipe below only as fallback.

When `find_drifting_entities` shows low displacement but `dive_solid`
reveals a non-monotonic temporal shape (arch, V-shape, spike-recovery).

```
find_drifting_entities(pattern, window=365)
-> check BOTH ends: high displacement = linear drift,
   low displacement does not mean no signal
-> entities with many temporal slices but near-zero displacement may have
   non-linear trajectories (arch, V-shape) that cancel out over time

Pick 3-5 entities with moderate displacement (rank 20-40, not top-10):
dive_solid(entity_key, pattern)
-> inspect delta values across temporal slices
-> arch shape (up then down) or V-shape (down then up) = trajectory anomaly

If arch/V-shape found:
find_drifting_similar(entity_key, pattern, top_n=20)
-> finds entities with the SAME trajectory shape
-> returns a cohort sharing the deformation pattern

Report the full cohort with trajectory description.
```

Linear drift has high displacement. Non-linear trajectories (arch, V)
have near-zero all-time displacement but distinctive shape. Only
`dive_solid` + `find_drifting_similar` catches these.

## Population segment shift

When `find_regime_changes` detects a changepoint, identify WHICH
population segment shifted — not just that a shift happened.

```
find_regime_changes(pattern)
-> changepoint at date X with magnitude Y

Run segment analysis — check each property dimension:
find_anomalies(pattern, property_filters={"nation": "<value>"}, limit=50)
-> repeat for each major segment value (nation, category, region)
-> which segment has the most anomalies post-changepoint?

contrast_populations(pattern, pre_window, post_window)
-> which dimensions shifted? Does it match the segment hypothesis?

Report: "segment S shifted by Z% from date X — N entities affected"
```

Individual entities in the shifted segment may each be only mildly
anomalous (below theta). The anomaly is at the GROUP level — no single
entity crosses the threshold but the segment centroid moves significantly.

## Collective drift

```
find_drifting_entities(pricing_pattern, window=365)
-> high displacement + consistent direction across ALL windows = drift

dive_solid(entity_key, pricing_pattern)
-> monotonically increasing delta across years = collective drift

find_regime_changes(pricing_pattern)
-> absent/weak changepoint = gradual shift, not step change

compare_entities(drift_entities, normal_entities, pricing_pattern)
-> which dimension drives the drift?
```

Use `window=365` (not shorter) to amplify gradual signal.
`signal_quality="mixed"` in sphere_overview indicates the pattern is
sensitive to drift.

## Multi-pattern investigation

When an entity is NOT anomalous in the expected pattern:

```
cross_pattern_profile(key, line_id)
-> source_count >= 2? Another pattern catches what the first missed.
-> >30% anomalous events = strong signal even if anchor says "normal"

Check related composite patterns:
Many anomalous composites from one entity = systematic issue.

composite_risk(key, line_id)
-> Fisher's method across independent patterns.
-> combined_p < 0.05 = significant even if no single pattern flags it.
```

## passive_scan for confirmation

Use for **confirmation**, not discovery.

```
find_anomalies per pattern -> top suspects
passive_scan(line_id, threshold=1) -> all single+ source
passive_scan(line_id, threshold=2) -> confirmed multi-source only
```

Cross-line bridging: `passive_scan("accounts")` auto-discovers sibling
lines (same `source_id`). Pre-0.1.x spheres need explicit sources.

## Exhaustive enumeration

**NEVER report "top 3 examples" — enumerate ALL entities that match the
criterion with limit=50.** Incomplete entity lists are the #1 reason for
missed recall in benchmarks.

- `limit=50` (not default 20) on ALL investigation queries
- `having={"gt": threshold}` to extract full cohort above metric value
- `aggregate_anomalies(group_by=...)` for large anomaly populations
- If the tool returns exactly `limit` rows, increase limit or note truncation

"There are ~50 like this, here are 5 examples" is incomplete — list them all.

## Re-ranking by property

`find_anomalies` ranks by delta_norm. But the most geometrically extreme
entity may not be the most business-relevant.

```
find_anomalies(pattern, rank_by_property="<dim_from_profiling_alert>")
```

Run for every `profiling_alert` dimension to surface property-specific extremes.

## Temporal artifact check

At session open, compare `get_sphere_info()` temporal date range against the
dataset's described time window. If the temporal span is implausibly wide
relative to the actual data (e.g., decades for a dataset covering months),
the timestamps are a sphere construction artifact — not data corruption.

```
get_sphere_info() -> check temporal_range min/max
If span > 10x the expected dataset window:
  -> flag as sphere construction artifact in report (INFO, not CRITICAL)
  -> use slice indices and relative ordering for temporal analysis
  -> do NOT report absolute timestamps as evidence of data corruption
```

This prevents misclassifying synthetic temporal axes as critical findings.
Temporal tools (`dive_solid`, `find_drifting_entities`, `find_regime_changes`)
still work correctly on the delta structure — the relative ordering of slices
is reliable even when absolute timestamps are synthetic.

## Physical bound validation

For event patterns with movement-related dimensions (speed, distance,
duration, velocity, or similar), validate extremes against physical
plausibility before interpreting them as behavioral anomalies.

```
rank_by_property(event_pattern, "<speed_dim>", direction="desc", limit=50)
rank_by_property(event_pattern, "<distance_dim>", direction="desc", limit=50)
rank_by_property(event_pattern, "<duration_dim>", direction="asc", limit=50)
```

Check the extremes against domain-appropriate physical bounds:
- Speed exceeding plausible limits for the transport mode
- Distance exceeding plausible range for the service area
- Duration near zero with non-zero distance (teleportation artifact)
- Duration extremely high with near-zero distance (meter/GPS stuck)

Entities violating physical bounds are data quality findings — GPS
artifacts, meter errors, or system clock issues — not behavioral
anomalies. Report them as data quality with the specific bound violated
and the entity count affected.

## Payment type segmentation

For event patterns with tip, gratuity, or rate dimensions, ALWAYS
segment by payment method before drawing conclusions about anomaly
distribution. Different payment methods have structurally different
tipping behavior that is definitional, not anomalous.

```
aggregate(event_pattern, group_by="<payment_type_dim>",
          geometry_filters={"is_anomaly": true}, limit=50)
aggregate(event_pattern, group_by="<payment_type_dim>", limit=50)
```

Compare anomaly rates per payment type. If one payment type has a
dramatically higher anomaly rate AND the driving dimension is
tip-related, the signal is likely payment-type structural:
- Cash/non-electronic payments typically have zero tips by definition
  (no electronic tip capture mechanism)
- Mixing payment types inflates zero-tip anomaly counts

Report the structural explanation and recommend payment-type-stratified
analysis rather than flagging the entire zero-tip population as anomalous.

## Unregistered entity check

For entity patterns with an ID, vendor, source, or provider dimension,
verify that all distinct values map to known entities in the domain.

```
get_line_profile(line_id, "<id_dim>") -> enumerate distinct values
Cross-reference against known entities in the domain context.
```

Unregistered or unexpected IDs are a geometric signal — they form
isolated clusters because their behavior doesn't match any registered
population. For each unregistered value:

```
search_entities(line_id, "<id_dim>", "<unknown_value>") -> get entity keys
find_anomalies(pattern, property_filters={"<id_dim>": "<unknown_value>"})
```

Characterize: is this a test entity, a data migration artifact, a
decommissioned operator, or an undocumented participant? The geometric
isolation alone is a finding — combine with business context to assess
severity.

---

## When things don't work

- **Tool returns empty results** — try different parameters, wider sample,
  or a different pattern. Some patterns may not have the data shape you expect.
- **Tool errors** — check pattern_id and version, verify sphere is open.
  If the error persists, report it rather than retrying in a loop.
- **No anomalies found** — not every sphere has every anomaly type.
  Report "no signal detected" as a valid finding.
- **Aggregate returns 0 rows** — the event pattern may lack events in the
  requested time window. Widen the window or check the temporal range with
  `get_sphere_info`.
- **find_similar_entities returns only the query entity** — the pattern
  population may be too small or the entity is an extreme outlier with no
  geometric neighbors. Try a different pattern or increase `top_n`.

---

## Skill delegation

| Need | Skill |
|---|---|
| Root cause tracing, entity 360, hypothesis testing | gds-investigator |
| Cross-pattern, neighbor, trajectory, segment scans | gds-scanner |
| Drift interpretation, regime change handling | gds-monitor |
| Orientation, profiling, clustering | gds-explorer |

Full detection examples: [references/examples.md](references/examples.md)
