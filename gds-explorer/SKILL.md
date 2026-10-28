---
name: gds-explorer
description: GDS sphere exploration — orientation, population analysis, segmentation, and aggregate analysis. Use when user asks to "explore a sphere", "orient in a sphere", "show me what's in this sphere", "find archetypes", "cluster entities", "segment the population", "compare groups", "what patterns exist", or "give me an overview". Requires hypertopos MCP server.
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.10.0
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

## Exploration checklist

- [ ] Orientation calls run (`sphere_overview`, `get_sphere_info`, `anomaly_summary`)
- [ ] Every `profiling_alerts` dimension looked at
- [ ] Signals matched to next steps (see "Matching signals to next steps")
- [ ] Hand-off to the right specialist skill named in the report

## Orientation

Every exploration starts here, regardless of the question:

```
sphere_overview()        -> patterns, anomaly rates, calibration health, profiling_alerts
get_sphere_info()        -> lines, columns, aliases, FTS availability
anomaly_summary(top_pattern) -> what drives anomalies
```

To explore a branch instead of main, pass `branch=` to `open_sphere`: `open_sphere(path, branch="my_branch")`. Use `manage_branch(action="list")` to see all available branches before opening one.

> **Performance:** Always use `detail="summary"` (default) for interactive work.
> `detail="full"` runs event-rate-divergence scans that can take **minutes** on
> large spheres (>100K entities).  Reserve "full" for deep-dive diagnostics only.

After these 3 calls, check for signals that guide next steps:

- **`dimension_kinds`** in sphere_overview — shows distribution-kind tags per dimension
  (e.g. "bernoulli x4, poisson x2, gaussian x8"). Use this as a quick orientation to
  understand which patterns have count-heavy vs magnitude-heavy vs binary geometries.
  Patterns with many bernoulli dims (and no calibrated score) perform best with `metric="Linf"` in find_anomalies.
  Patterns with mixed kinds (poisson + gaussian + bernoulli): compare `bregman_divergence` with `delta_norm` in `find_anomalies` results.
- **`confidence_distribution`** in sphere_overview — when present, shows the spread of
  bootstrap anomaly confidence across the flagged population. A distribution skewed toward
  low confidence suggests the anomaly rate is inflated by unstable detections. In that case,
  filter downstream calls: `find_anomalies(pattern_id, min_confidence=0.7)` to focus on
  high-confidence detections before investing in full investigation (patterns without the
  calibrated score; on calibrated ones keep the smallest `conformal_p` instead).
- **`profiling_alerts`** — run each alert's suggested call
  (`find_anomalies(rank_by_property=<dim>)`) on every pattern including composites.
- **`dim_quality_warnings`** — four silent-failure auditors fire per pattern:
  `dead_dim` (zero-variance dim; z-score undefined), `sparse_dim` (mostly-zero
  with rare nonzero tail; gaussian assumption wrong), `dominant_dim_mass`
  (pattern-level: one dim ≥ 70 % of tail variance — sphere is a one-dim detector
  on this pattern, multi-dim composition adds little), and `negative_space`
  (gaussian-declared dim sitting at point-mass-at-zero — z-score treats zero
  as typical when it is the absence value). If `dominant_dim_mass` or
  `negative_space` fires, raise a calibration ticket rather than investigating
  the flagged entities — they are likely noise. Full triage rules in `gds-monitor`.
- **`has_temporal: true`** — for each temporal pattern, consider running
  `find_drifting_entities(pattern_id, top_n=3)` to detect behavioral changes.
- **Event patterns with edge tables** — run `edge_stats(pattern_id)` for each
  event pattern. If `has_edge_table: true`, note that graph traversal tools
  (`find_geometric_path`, `extract_chains`) and behavioural-vs-graph cross-tab
  (`find_graph_geometry_tension(primary_key, anchor_pattern, line_id=event_pattern)`)
  are available for that pattern. The stats (row_count, unique_from/to, avg_degree)
  help estimate graph density before committing to path-finding or chain discovery calls.

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
| `geometry_mode = binary` | Skip anomaly tools. Use `find_hubs` + `get_polygon(pattern)` / `jump_polygon(...)` for an entity's relation neighbours + `get_centroid_map(pattern, group_by_line=X)`. If pattern has edge table (`edge_stats` returns `has_edge_table: true`), prefer `find_geometric_path(from, to, pattern)` for entity-to-entity reachability (paths scored by geometric coherence) and `extract_chains(pattern, seed_nodes=[entity])` for the time-ordered chains from that entity |
| `inactive_ratio > 0.30` | `is_anomaly = true` means "active entity", not "problem" — verify with `contrast_populations` |
| Multiple patterns on same entity line | Signal dilution risk. Check `anomaly_summary` on each. If recall is low, consider entity line isolation |
| Composite patterns present | Priority check. Composites surface subgroup-level anomalies invisible at anchor level. Rank by per-transaction dims (avg, max) not just count |
| Patterns on isolated entity lines (e.g. `<entity_line_variant>`) | `passive_scan` and `composite_risk` auto-discover sibling lines (same `source_id`). If sphere lacks `source_id`, use explicit sources or per-entity `goto` + `get_polygon` for cross-line triangulation |
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
| "Any domain outliers or bad values?" | `find_anomalies(pattern_id, rank_by_property="<dim>")` — unusual values, extreme outliers, null-heavy dimensions; add `rank_order="asc"` for the lowest values |
| "Did population shift over time?" | `find_regime_changes(pattern_id, n_regimes=5)` |
| "What values does this property have?" | `get_line_profile(line_id, property)` — instant, ~50ms |
| "Event volume per group?" | `aggregate(event_pattern, group_by_line=X)` |
| "Which groups have anomalous events?" | `aggregate(event_pattern, group_by_line=X, geometry_filters={"is_anomaly": true})` |
| "Volume per sender, when events relate to the same line twice (sender / receiver)?" | `aggregate(event_pattern, group_by_line=X, group_by_relation="sender")` — the "ambiguous" error lists the relation names |
| "Graph structure for binary FK?" | `get_polygon(pattern)` (alive relation edges) then `jump_polygon` hop by hop; `find_common_relations(key_a, key_b, pattern)` for shared targets |
| "Edge table density / graph stats?" | `edge_stats(pattern_id)` — row_count, unique_from/to, avg_degree |
| "How are two entities connected?" | `find_geometric_path(from_key, to_key, pattern_id)` — bidirectional BFS scored by geometric coherence |
| "Transaction chains from this entity?" | `extract_chains(event_pattern_id, seed_nodes=[primary_key])` — exact chain extraction from that entity, no pre-built chains needed |
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
- Responses default to the compact format: when a response carries `omitted`, the named subtrees (positional `delta`, numeric property columns) were left out, not missing — follow its `full_response` hint, pass `summary=false`, or open the sphere with `response_format="full"` when you need them

## Selection modes for anomaly results

`find_anomalies`, `attract_boundary`, `find_hubs`, and `find_drifting_entities` support two selection modes: the default `select="top_norm"` returns the most extreme entities by delta norm, while `select="diverse"` applies submodular facility location to return a geometrically spread set covering different anomaly types. During exploration, `top_norm` answers "show me the most extreme" and `diverse` answers "show me all the kinds of anomaly present." On calibrated patterns (non-empty `score_dims`) `is_anomaly` is the alarm budget — `conformal_p ≤ 1 − anomaly_percentile/100`, about 5 % of a clean population at the 95th percentile — so a flagged share (`find_anomalies.total_found` / population) well above that budget is real excess. When the pattern declares a spatial / temporal hierarchy, `fdr_alpha` with `fdr_resolution` shows which regions concentrate the anomalies — useful when handing results to specialist skills.

On patterns without the calibrated score, `find_anomalies` also accepts `min_confidence` (0–1) to filter by bootstrap anomaly confidence. In exploration, this is useful when the population has many borderline detections: `min_confidence=0.7` returns only entities where the anomaly is stable across bootstrap resamples. Omit for initial orientation to see the full landscape, then narrow when handing suspects to investigation skills. Not applicable for populations > 50K, `group_by_property` or calibrated patterns (confidence is absent).

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
  An unknown parameter name is rejected; the error suggests the closest
  declared name when there is one ("did you mean 'top_n'?") — fix the
  name instead of dropping the argument.
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
