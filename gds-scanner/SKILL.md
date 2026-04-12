---
name: gds-scanner
description: Advanced anomaly detection — cross-pattern discrepancy, geometric neighbor contamination, temporal trajectory shape, population segment shift. These scans find anomaly types invisible to standard find_anomalies and passive_scan. Use when the sphere has multiple patterns covering the same entity line, temporal data, or categorical properties (nation, segment).
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.2.3
  mcp-server: hypertopos
---

# GDS Scanner

A GDS scanner looks for anomaly types invisible to standard `find_anomalies`
and `passive_scan`: cross-pattern discrepancies, geometric neighbor
contamination, non-linear temporal trajectories, and population segment
shifts.

Each scan below targets a distinct anomaly category. Standard detection
finds entities that are anomalous — these scans find entities that are
anomalous in ways standard detection cannot see. Covering all four gives
the most complete picture; skipping one leaves that category unchecked.

> **Note:** If the MCP server version supports the `detect_*` tools (verify with
> `sphere_overview`), prefer those over manual recipes below — they encapsulate these
> workflows in a single call. Key algorithm differences from manual recipes:
>
> - `detect_neighbor_contamination` uses **inverted search** — starts from anomalous
>   entities, finds their normal neighbors, checks contamination rate. More effective
>   than the manual "sample normals → check neighbors" approach. The `sample_size`
>   parameter means "anomalous entities to start from", not "entities per group".
> - `detect_trajectory_anomaly` uses **full temporal scan** — streams all temporal data,
>   classifies every entity's trajectory shape, ranks by `wasted_motion`
>   (path_length - displacement). No need to manually call `find_drifting_entities` at
>   multiple rank offsets. The `displacement_ranks` parameter is deprecated/ignored;
>   `top_n_per_range` is now a total results cap.

---

## FDR control and diverse selection

Large-population scans should always set `fdr_alpha` (e.g. `fdr_alpha=0.05`) on `find_anomalies`, `attract_boundary`, `find_hubs`, and `find_drifting_entities` because the multiple-testing problem is worse at scale — without FDR control, a 100K-entity scan produces hundreds of false discoveries that contaminate downstream cross-pattern and contamination analysis. Use `select="diverse"` to surface the distinct types of anomaly present in the population rather than ranking purely by severity; this applies submodular facility location to maximize geometric coverage across the result set, which is critical when the scan feeds into segment shift or trajectory shape analysis where variety matters more than extremity.

---

## Choosing which scans to run

| Sphere characteristic | Relevant scan |
|---|---|
| Entity line covered by 2+ patterns | Cross-pattern discrepancy |
| Any pattern with entities | Geometric neighbor contamination |
| Patterns with `has_temporal: true` | Temporal trajectory shape |
| Categorical properties (nation, segment, region) | Population segment shift |
| Sphere has aliases (`get_sphere_info` shows aliases) | Alias anomaly scan — **mandatory** |
| Pattern has edge table (`edge_stats` returns `has_edge_table: true`) | `contagion_score_batch` for neighbor contamination via transaction graph |

---

## Alias anomaly scan

**Mandatory.** Aliases are derived views with their own geometry — scanning
only base patterns misses alias-specific anomalies. Skipping alias coverage
is the most common cause of incomplete discovery in benchmarks.

For every alias returned by `get_sphere_info()`:

```
find_anomalies(alias_pattern, top_n=20)
attract_boundary(alias, base_pattern, direction="in")
attract_boundary(alias, base_pattern, direction="out")
```

Alias-specific anomalies often surface aggregation-level signals (e.g.,
vendor-level or segment-level deviations) invisible in entity-level base
patterns.

---

## Cross-pattern discrepancy

**What it finds:** entities anomalous in ONE pattern but NORMAL in another.
This is the OPPOSITE of multi-source confirmation (threshold=2).

For each entity line covered by 2+ patterns:

```
passive_scan(line_id, threshold=1)
```

To include near-threshold entities in the discrepancy analysis, add borderline sources:
```
passive_scan(home_line_id=line_id, sources='[
  {"type": "geometry", "pattern_id": "pattern_a"},
  {"type": "geometry", "pattern_id": "pattern_b"},
  {"type": "borderline", "pattern_id": "pattern_a", "rank_threshold": 80}
]')
```

See gds-fraud-investigator skill for full source type reference (`geometry`, `borderline`,
`points`, `compound`).

Filter results to entities with source_count=1. For each:

```
cross_pattern_profile(key, line_id)
```

What you're looking for: one pattern says "anomalous", another says "normal".
Example: supplier anomalous in pricing but normal in activity. This is a DISTINCT
finding type — do not collapse with multi-source findings.

**Common mistake:** using threshold=2 (finds agreement). Here you need threshold=1
(finds discrepancy).

**Complementary:** For spheres with clear cluster structure, `cluster_bridges(pattern_id)` surfaces entities that straddle cluster boundaries — these often manifest as cross-pattern discrepancies (anomalous in one cluster's context, normal in another's).

## Geometric neighbor contamination

**What it finds:** normal entities whose geometric neighbors are systematically
anomalous. The target is NORMAL — invisible to any anomaly scan.

> **Prefer (in order):** (1) `detect_neighbor_contamination` — inverted search, most effective.
> (2) `contagion_score_batch(candidate_keys, pattern_id)` — if edge table available, computes
> exact anomalous/total neighbor ratio via transaction graph. (3) Manual recipe below — fallback.
>
> **As-of retrospective scan:** `contagion_score_batch` (and the other five
> edge-table graph primitives) accepts an optional `timestamp_cutoff` parameter
> (Unix seconds). Use it to re-run the scan as the graph looked on a prior date —
> useful for validating new detection recipes against historical snapshots without
> reopening the sphere at a different manifest version.

**Manual recipe** (fallback): run `find_similar_entities` on at least 10 entities
(5 anomalous + 5 normal).

**From anomalous entities:**

```
For 5 anomalous entities from find_anomalies:
  find_similar_entities(entity_key, pattern, top_n=10)
  check_anomaly_batch([neighbor_keys], pattern)
If a NORMAL neighbor has >50% anomalous neighbors, it is a contamination target.
```

**From normal entities:**

```
For 5 normal entities (delta_rank_pct 30-70):
  find_similar_entities(normal_key, pattern, top_n=10)
  check_anomaly_batch([neighbor_keys], pattern)
```

Contamination targets are by definition normal — you won't find them
by only checking anomalous entities. Check both directions.

## Temporal trajectory shape

**What it finds:** entities with non-linear temporal trajectories (arch,
V-shape, spike-recovery) that have near-zero displacement — invisible
to top-N drift ranking.

> **Prefer `detect_trajectory_anomaly`** — it performs a full temporal scan,
> streaming all temporal data and classifying every entity's trajectory shape.
> It ranks results by `wasted_motion` (path_length - displacement), which directly
> targets non-linear trajectories. No need to manually probe at multiple displacement
> rank offsets. The `displacement_ranks` parameter is deprecated/ignored; use
> `top_n_per_range` as a total results cap. Use the manual recipe only if `detect_*`
> tools are unavailable.

**Manual recipe** (fallback): run `dive_solid` on entities from DIFFERENT
displacement ranges — not just top-10 or mid-rank. Check at least one entity
from each range:
- Top-5 displacement (likely linear drift)
- Rank 20-50 (moderate — may be non-linear)
- Rank 100-200 (low displacement — arch/V candidates hide here)

```
dive_solid(entity_key, pattern)
```

Inspect delta values across slices:
- Monotonic increase = linear drift (already caught by drift detection)
- Up-then-down (arch) or down-then-up (V) = trajectory anomaly
- Flat = no signal (move to next range)

**If flat at rank 20-50, check rank 100-200 before concluding.**
Arch trajectories can have very low displacement.

If arch/V-shape found:

```
find_drifting_similar(entity_key, pattern, top_n=20)
```

This recovers the full cohort sharing the same trajectory shape.

Key insight: linear drift has HIGH displacement. Non-linear trajectories
have NEAR-ZERO displacement — they cancel out over time. Searching lower
displacement ranges is the only way to find them.

## Population segment shift

**What it finds:** a specific segment (nation, category, region) that shifted
collectively — individual entities below theta, group centroid moved significantly.

Run after every `find_regime_changes` detection. Do not dismiss a regime change
as "data boundary" without segment analysis first.

```
get_line_schema(entity_line)  -> find categorical properties
```

For each categorical property with <50 distinct values:

```
find_anomalies(pattern, property_filters={"<prop>": "<value>"}, limit=50)
```

Compare anomaly counts per segment value. The segment with disproportionately
more anomalies = affected group.

**When you find the affected segment, list ALL entity keys in it:**

```
search_entities(entity_line, "<prop>", "<segment_value>")
```

These are ALL entities in the shifted segment — include the full key list
in the report. Do not just say "GERMANY shifted" — list every entity.

**Common mistake:** detect regime change, attribute to data boundary, move on.
Decompose by segment before dismissing.

---

## When things don't work

- **Tool returns empty results** — try different parameters, wider sample,
  or a different pattern. Not all spheres have the data shape these scans expect.
- **Tool errors** — check pattern_id and version, verify sphere is open.
  If the error persists, report it rather than retrying in a loop.
- **No anomalies found** — not every sphere has every anomaly type.
  Report "no signal detected for this scan type" as a valid finding.
- **passive_scan returns 0 results at threshold=1** — the entity line may
  be covered by only one pattern. Cross-pattern discrepancy requires 2+ patterns.
- **find_similar_entities returns few neighbors** — small population or
  extreme outlier. Reduce `top_n` or try a different pattern.
- **dive_solid returns no temporal slices** — the pattern lacks temporal data.
  Skip trajectory shape analysis and note it in the report.

---

## Skill delegation

| Need | Skill |
|---|---|
| Root cause tracing, entity 360, hypothesis testing | gds-investigator |
| Event rates, Simpson's, temporal bursts, drift recipes | gds-detective |
| Drift interpretation, regime change handling | gds-monitor |
| Orientation, profiling, clustering | gds-explorer |
| Single-call detection of all 4 scan types | Use `detect_cross_pattern_discrepancy`, `detect_neighbor_contamination` (inverted search), `detect_trajectory_anomaly` (full temporal scan), `detect_segment_shift` MCP tools directly |
