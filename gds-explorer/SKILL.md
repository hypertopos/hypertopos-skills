---
name: gds-explorer
description: GDS sphere exploration — orientation, population analysis, segmentation, and aggregate analysis. Use when user asks to "explore a sphere", "orient in a sphere", "show me what's in this sphere", "find archetypes", "cluster entities", "segment the population", "compare groups", "what patterns exist", or "give me an overview". Requires hypertopos MCP server.
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.4.1
  mcp-server: hypertopos
---

# GDS Explorer

A GDS explorer orients in unfamiliar spheres — understanding what entity
lines exist, which patterns capture meaningful geometry, how the population
is distributed, and where to focus deeper investigation.

The goal is efficient orientation: 3 calls to understand the landscape,
then targeted profiling to find the signal, then hand off to the right
specialist skill. Depth beats breadth — 20 calls on one promising pattern
outperform 5 calls across 4 patterns.

For concrete tool output examples, see [references/examples.md](references/examples.md).

---

## Orientation

Every exploration starts here, regardless of the question:

```
sphere_overview()        -> patterns, anomaly rates, calibration health, profiling_alerts
get_sphere_info()        -> lines, columns, aliases, FTS availability
anomaly_summary(top_pattern) -> what drives anomalies
```

> **Performance:** Always use `detail="summary"` (default) for interactive work.
> `detail="full"` runs event-rate-divergence scans that can take **minutes** on
> large spheres (>100K entities).  Reserve "full" for deep-dive diagnostics only.

After these 3 calls, check for signals that guide next steps:

- **`dimension_kinds`** in sphere_overview — shows distribution-kind tags per dimension
  (e.g. "bernoulli x4, poisson x2, gaussian x8"). Use this as a quick orientation to
  understand which patterns have count-heavy vs magnitude-heavy vs binary geometries.
  Patterns with many bernoulli dims perform best with `metric="Linf"` in find_anomalies.
  Patterns with mixed kinds (poisson + gaussian + bernoulli) benefit from `metric="bregman"`.
- **`confidence_distribution`** in sphere_overview — when present, shows the spread of
  bootstrap anomaly confidence across the flagged population. A distribution skewed toward
  low confidence suggests the anomaly rate is inflated by unstable detections. In that case,
  filter downstream calls: `find_anomalies(pattern_id, min_confidence=0.7)` to focus on
  high-confidence detections before investing in full investigation.
- **`profiling_alerts`** — run each alert's suggested call
  (`find_anomalies(rank_by_property=<dim>)`) on every pattern including composites.
- **`has_temporal: true`** — for each temporal pattern, consider running
  `find_drifting_entities(pattern_id, top_n=3)` to detect behavioral changes.
- **Event patterns with edge tables** — run `edge_stats(pattern_id)` for each
  event pattern. If `has_edge_table: true`, note that graph traversal tools
  (`find_geometric_path`, `discover_chains`) are available for that pattern.
  The stats (row_count, unique_from/to, avg_degree) help estimate graph density
  before committing to path-finding or chain discovery calls.

**MANDATORY: always run `find_clusters` on every anchor pattern during
orientation.** Population archetypes reveal structural segments (e.g. airport
zones vs residential vs dead zones, premium vs budget customers) that
contextualize all subsequent findings. Skipping archetypes is a common
investigation gap — anomaly rates and drift only make sense relative to the
population structure that clustering reveals.

Then profile raw dimension values before diving into anomalies.

## Profiling raw dimensions

For each anchor pattern with derived dimensions:

```
get_line_profile(entity_line, dim_property)
```

Check 2-3 key dimensions per pattern. What to look for:
- **MAX >> P99** — an extreme cluster exists that find_anomalies may rank low
  (e.g. max <some_dimension>=400 when p99=50 means a hidden cluster of
  extreme entities that delta_norm ranks below the majority)
- **MIN << P01** — same in the other direction
- **Bimodal split** — two populations mixed in one pattern

This catches signals that delta_norm misses. Normalization absorbs
outlier clusters into population mu/sigma — raw profiles reveal them.

---

## Matching signals to next steps

| Signal | What to do |
|---|---|
| Entity line has status/default/churn/risk property | Go to gds-investigator for ground truth validation — highest priority |
| `calibration_health = poor` | `recalibrate(pattern_id)` before trusting anomaly results |
| `geometry_mode = binary` | Skip anomaly tools. Use `find_hubs` + `find_neighborhood(entity, pattern, max_hops=2)` + `get_centroid_map(pattern, group_by_line=X)`. If pattern has edge table (`edge_stats` returns `has_edge_table: true`), prefer `find_geometric_path(from, to, pattern)` for entity-to-entity reachability and `discover_chains(entity, pattern)` for temporal chain discovery — both are faster than polygon-edge BFS and score paths by geometric coherence |
| `inactive_ratio > 0.30` | `is_anomaly = true` means "active entity", not "problem" — verify with `contrast_populations` |
| Multiple patterns on same entity line | Signal dilution risk. Check `anomaly_summary` on each. If recall is low, consider entity line isolation |
| Composite patterns present | Priority check. Composites surface subgroup-level anomalies invisible at anchor level. Rank by per-transaction dims (avg, max) not just count |
| Patterns on isolated entity lines (e.g. `<entity_line_variant>`) | `passive_scan` and `composite_risk` auto-discover sibling lines (same `source_id`). If sphere lacks `source_id` (pre-0.1.x), use explicit sources or per-entity `goto` + `get_polygon` for cross-line triangulation |
| Aliases with `has_cutting_plane = true` | For each alias, run `attract_boundary(alias_id, pattern_id, direction="in")` AND `attract_boundary(alias_id, pattern_id, direction="out")` — both directions give the most complete picture |

---

## Tools by question

| Question | Tool |
|---|---|
| "What types of entities exist?" | `find_clusters(pattern_id, n_clusters=5, sample_size=5000)` |
| "How do subgroups differ?" | `get_centroid_map(pattern_id, group_by_line=X)` |
| "What drives anomalies?" | `contrast_populations(pattern_id, {"anomaly": true})` |
| "How are anomalies distributed?" | `aggregate_anomalies(pattern_id, group_by="property")` — when total_found >> top_n |
| "Is the sphere geometry intact?" | `check_alerts()` + `detect_data_quality_issues(pattern_id)` — structural/geometric issues only, not domain outliers |
| "Any domain outliers or bad values?" | `find_anomalies(pattern_id, rank_by_property="<dim>")` — unusual values, extreme outliers, null-heavy dimensions |
| "Did population shift over time?" | `find_regime_changes(pattern_id, n_regimes=5)` |
| "What values does this property have?" | `get_line_profile(line_id, property)` — instant, ~50ms |
| "Event volume per group?" | `aggregate(event_pattern, group_by_line=X)` |
| "Which groups have anomalous events?" | `aggregate(event_pattern, group_by_line=X, geometry_filters={"is_anomaly": true})` |
| "Graph structure for binary FK?" | `find_neighborhood(entity, pattern, max_hops=2)` |
| "Edge table density / graph stats?" | `edge_stats(pattern_id)` — row_count, unique_from/to, avg_degree |
| "How are two entities connected?" | `find_geometric_path(from_key, to_key, pattern_id)` — bidirectional BFS scored by geometric coherence |
| "Transaction chains from this entity?" | `discover_chains(primary_key, pattern_id)` — temporal BFS on edge table, no pre-built chains needed |
| "What is this entity's net flow?" | `entity_flow(key, pattern_id)` — outgoing/incoming/net per counterparty |
| "How contaminated is this neighborhood?" | `contagion_score(key, pattern_id)` — anomalous neighbor ratio (0-1) |
| "Is connection rate changing?" | `degree_velocity(key, pattern_id)` — accelerating/decelerating degree over time |
| "Have I explored enough?" | `investigation_coverage(key, pattern_id, explored_keys)` — explored vs unexplored counterparties |
| "Which entities bridge clusters?" | `cluster_bridges(pattern_id, n_clusters=5)` — cross-cluster intermediaries |
| "Are specific transactions anomalous?" | `anomalous_edges(from_key, to_key, pattern_id)` — event-level scoring per edge |

---

## Parameter guidance

- `sample_size=5000` for `find_clusters` on populations > 10K
- `sample_size=50000` for heavy `aggregate` on populations > 500K
- `goto()` then `get_polygon()` — always sequential, never parallel

## Selection modes for anomaly results

`find_anomalies`, `attract_boundary`, `find_hubs`, and `find_drifting_entities` support two selection modes: the default `select="top_norm"` returns the most extreme entities by delta norm, while `select="diverse"` applies submodular facility location to return a geometrically spread set covering different anomaly types. During exploration, `top_norm` answers "show me the most extreme" and `diverse` answers "show me all the kinds of anomaly present." An optional `fdr_alpha` parameter applies Benjamini-Hochberg FDR control to filter statistically insignificant results — useful when handing results to specialist skills, but exploratory orientation can leave it unset to see the full landscape.

`find_anomalies` also accepts `min_confidence` (0–1) to filter by bootstrap anomaly confidence. In exploration, this is useful when the population has many borderline detections: `min_confidence=0.7` returns only entities where the anomaly is stable across bootstrap resamples. Omit for initial orientation to see the full landscape, then narrow when handing suspects to investigation skills. Not applicable for populations > 50K, `group_by_property`, or `use_mahalanobis` patterns (confidence is absent).

---

## Avoiding common exploration traps

Exploring ALL patterns, ALL aliases, ALL tools equally (100+ calls, 20% recall)
is tempting but inefficient. Orient in 3 calls, find what matters, go deep on
that (60 calls, 95% recall).

## Minimal spheres

If sphere_overview returns 0 entities or no patterns, the sphere may be empty
or misconfigured. Report this to the user — do not attempt clustering or
profiling on empty populations. If population is very small (<100 entities),
skip sampling and use full population for all operations.

---

## Report structure

Reference skill phase names in report section headers so reviewers can trace
which workflow drove each finding. Example: "## Orientation (gds-explorer)",
"## Ground Truth Validation (gds-investigator)", "## Temporal Health (gds-monitor)".

---

## When things don't work

- **Tool returns empty results** — try different parameters, wider sample,
  or a different pattern. The sphere may lack data for the requested analysis.
- **Tool errors** — check pattern_id and version, verify sphere is open.
  If the error persists, report it rather than retrying in a loop.
- **No anomalies found** — not every sphere has every anomaly type.
  Report "no signal detected" as a valid finding.
- **sphere_overview returns 0 patterns** — the sphere may be empty or
  misconfigured. Report to the user and stop.
- **find_clusters produces a single cluster** — the population is too
  homogeneous for geometric separation. Try increasing `n_clusters` or
  check if the pattern has enough derived dimensions.
- **get_line_profile returns all zeros** — the property may not be populated
  for this entity line. Try a different property or check `get_line_schema`
  for available columns.

---

## Skill delegation

| Need | Skill |
|---|---|
| Ground truth validation, root cause tracing | gds-investigator |
| Event rates, Simpson's, temporal bursts, drift recipes | gds-detective |
| Cross-pattern, neighbor, trajectory, segment scans | gds-scanner |
| Drift interpretation, regime change handling | gds-monitor |

Full exploration examples: [references/examples.md](references/examples.md)
