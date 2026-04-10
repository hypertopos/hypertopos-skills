---
name: gds-analyst
description: GDS anomaly investigation — goal-oriented, reads investigation hints and plans approach around what to find. Use for any investigation task with or without specific anomaly categories. Triggers on "investigate", "find anomalies", "run analysis", "what's anomalous here", "sphere health check", or any anomaly investigation task.
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.2.0
  mcp-server: hypertopos
---

# GDS Analyst

A GDS anomaly analyst navigates geometric data spheres to find anomalies
that SQL cannot see: cross-pattern discrepancies, geometric neighborhood
effects, non-linear temporal trajectories, and population segment shifts.

The goal is actionable findings — not observations, not summaries, not
recommendations for future work. Every finding should answer: what is
anomalous, why, and what to do about it.

---

## Starting an investigation

Read the investigation instruction first. Look for:
- Anomaly categories to target (cross-pattern, neighborhood, trajectory, segment shift, etc.)
- Entity lines or patterns of interest
- Time windows, segments, or properties to focus on
- Hypotheses already stated

If the instruction mentions specific categories, those should guide your
investigation. Use the decision table below to find the right approach
for each category. If no categories are given, explore freely starting
with `sphere_overview()` (use `detail="summary"` for interactive work;
`detail="full"` can take **minutes** on >100K-entity spheres).

The skills are your toolbox — use what fits the task:
- **gds-scanner** — recipes for each anomaly category
- **gds-detective** — event rates, composites, drift, passive_scan
- **gds-investigator** — root cause, entity deep-dive, hypothesis testing
- **gds-monitor** — temporal drift, regime changes
- **gds-explorer** — orientation, profiling, clustering

---

## When you know what to look for

### Orient

```
sphere_overview()     → patterns, anomaly rates, calibration, has_temporal
get_sphere_info()     → lines, aliases, columns
edge_stats(pattern_id) → if event patterns exist, check edge table availability
```

15% of budget. Note which patterns cover the hinted entity lines.
If `edge_stats` returns `has_edge_table: true`, graph-aware tools are available (entity_flow, contagion_score, etc.).
Do not run generic find_anomalies yet — hints drive the next step.

**Temporal artifact check:** compare the temporal date range from
`get_sphere_info()` against the dataset's described time window. If the
span is implausibly wide (e.g., decades for a dataset covering months),
flag as a sphere construction artifact — use slice indices for relative
ordering, not absolute timestamps. See gds-detective for the full recipe.

### Match category to approach

For each hint in the instruction, identify the matching scan and run it.
See [references/decision-framework.md](references/decision-framework.md) for the
full mapping. The most common:

| Hint category | Primary scan | Secondary |
|---|---|---|
| Cross-pattern discrepancy | `passive_scan(threshold=1)` + `cross_pattern_profile` | `composite_risk` for borderline |
| Geometric neighborhood | `detect_neighbor_contamination` (inverted search) | `find_similar_entities` + `check_anomaly_batch` fallback |
| Temporal trajectory | `detect_trajectory_anomaly` (full temporal scan) | `dive_solid` + `find_drifting_similar` fallback |
| Population segment shift | `find_regime_changes` → segment `property_filters` | `contrast_populations` |
| Event anomaly rate | `aggregate(geometry_filters=is_anomaly)` + total `aggregate` | `sphere_overview(detail="full")` for alerts |
| Composite subgroup | `aggregate_anomalies` + `find_anomalies(rank_by_property=avg_dim)` | dual ranking |
| Physical impossibility | `rank_by_property` on speed/distance/duration dims | Physical bound validation in gds-detective |
| Unregistered entity / ghost vendor | `get_line_profile` on ID dims → cross-ref domain | Unregistered entity check in gds-detective |

Allocate **70% of budget to matched scans**, 30% to exploration and validation.

Every hint requires a finding in the report — even if the finding is "not detected."

### Root cause the findings

For every anomalous cluster discovered by targeted scans:

```
explain_anomaly(key, pattern)       → WHICH dimension dominates?
dive_solid(key, pattern)            → WHEN did it change?
find_counterparties or get_event_polygons(key)  → WHAT caused it?
  entity_flow(key, pattern_id)      → net flow summary before detailed counterparty drill-down
  contagion_score(key, pattern_id)  → what fraction of neighbors are anomalous?
compare_entities(key, normal_peer)  → HOW does it differ?
```

Minimum: first 3 links. Fourth for top-priority findings only.

### Step 4T — Cross-validate + temporal

For key suspects:
```
cross_pattern_profile(key, line_id)     → multi-source confirmation
passive_scan(line_id, threshold=2)      → confirmed multi-source cohort
```

For patterns with `has_temporal: true`:
```
find_drifting_entities(pattern, top_n=5)
```

---

## When you're exploring freely

### Orient + profile

```
sphere_overview()                  → patterns, rates, profiling_alerts
                                     (use detail="full" only for small spheres
                                      — slow on >100K entities)
# If needed for event_rate_divergence_alerts on small spheres:
# sphere_overview(detail="full")   → adds temporal_quality, divergence alerts
get_sphere_info()                  → lines, aliases
check_alerts()                     → calibration issues
```

For each `profiling_alerts` dimension, execute the suggested call immediately.
For each `event_rate_divergence_alerts` entry, flag for temporal investigation.

**Temporal artifact check:** compare `get_sphere_info()` temporal range
against the dataset's described window. Implausibly wide span = sphere
construction artifact, not data corruption. Use slice indices for relative
analysis. See gds-detective for the full recipe.

### Detect

```
find_anomalies(pattern, top_n=20)              → geometric anomalies
aggregate(event_pattern, group_by_line=anchor, geometry_filters={"is_anomaly": true})
                                               → event anomaly rate per entity
```

For composite patterns:
```
aggregate_anomalies(composite, group_by="parent_key_col")
find_anomalies(composite, rank_by_property="avg_dim", top_n=50)
```

For spheres with aliases:
```
attract_boundary(alias, pattern, direction="in")
attract_boundary(alias, pattern, direction="out")
```

### Advanced scans

Run ALL four scans from gds-scanner regardless of what step 2 found:
1. Cross-pattern: `passive_scan(threshold=1)` → entities with source_count=1 → `cross_pattern_profile`
2. Neighbor: `find_similar_entities` on 5 anomalous + 5 normal entities → `check_anomaly_batch`
3. Trajectory: `dive_solid` at top / mid / low displacement ranges
4. Segment: `find_regime_changes` → per-segment `property_filters`

Skipping a scan means missing an entire anomaly category.

**Mandatory alias coverage:** after processing all base patterns, run the
same anomaly scan (`find_anomalies`, `attract_boundary` in/out) on EVERY
available alias. Aliases expose derived views (e.g., vendor-level
aggregation, high-activity segments) that base patterns cannot surface.
Skipping aliases = incomplete discovery.

### Step 4E — Root cause (40% of total budget)

For EVERY cluster found in steps 2E-3E, run the root cause chain:
```
explain_anomaly → dive_solid → find_counterparties → compare_entities
```

### Step 5E — Temporal validation

For all patterns with `has_temporal: true`:
```
find_drifting_entities(pattern, top_n=5)
find_regime_changes(pattern, n_regimes=5)
```

---

## Budget allocation

| Mode | Orient | Scans | Root cause | Temporal + validation | Report |
|------|--------|-------|------------|-----------------------|--------|
| Targeted | 15% | 70% | (included) | 10% | 5% |
| Exploration | 15% | 20% | 40% | 15% | 10% |

An investigation with 60 detection calls and 0 root cause calls is a failure.

---

## Quality checklist — verify before writing the report

```
□ Every hint in the instruction has a finding (confirmed or "not detected")
□ Root cause chain (explain → when → what) for every major finding
□ Every finding has (a) mechanism, (b) affected entity list, (c) concrete remediation step
□ Entity lists are complete — ALL entities, not "top 5 examples"
□ limit=50 (not default 20) on all investigation queries
□ Minimum 3 hypotheses tested with explicit confirmed/rejected verdicts
□ False positive assessed for every finding
□ Overall estimated FP rate stated at end of report
□ investigation_coverage checked for key suspects (if edge table available)
□ Each finding classified: CONFIRMED / SUSPECTED / FALSE POSITIVE
□ If has_temporal patterns: ran find_drifting_entities on each
□ Cross-validated top findings via passive_scan or cross_pattern_profile
□ If event_rate_divergence_alerts present: windowed aggregate to localize burst
□ Composite patterns: both count ranking AND value ranking executed
□ Every available alias scanned (find_anomalies + attract_boundary in/out)
□ If tip/rate dimensions present: segmented by payment type before conclusions
□ Temporal range checked for sphere construction artifacts
```

---

## Report structure — best practices

This is a best-practice template, not a rigid format. Adapt sections to
the sphere and investigation scope — a 5-finding report doesn't need all
sections. The minimum viable report has: Executive Summary, Findings
(severity-grouped with per-finding fields), FP Summary, and
Recommendations.

### Metadata header

```markdown
# [Sphere Name] GDS Investigation Report
**Sphere:** `sphere_id` | **Engine version:** vX.Y.Z | **Date:** YYYY-MM-DD
**Skills used:** [list of skills applied]
```

### Executive Summary

One dense paragraph: total finding count, critical findings named
explicitly, confirmed data errors vs structural/behavioral signal,
overall investigation character. Helps the reader decide how deep to go.

### Sphere Overview

Per-pattern table for at-a-glance comparison, plus profiling alerts:

```markdown
| Pattern | Type | Entities | Anomaly Rate | Calibration | Temporal | Top Driving Dim |
|---|---|---|---|---|---|---|

**Profiling alerts:**
| Pattern | Dimension | P99 | Max | Ratio | Alert Type |
|---|---|---|---|---|---|
```

### Population Archetypes

If `find_clusters` was run, present the archetype structure before
findings — gives the reader the segmentation context needed to interpret
individual findings.

### Findings — severity-grouped

Group findings under severity headers: CRITICAL → HIGH → MEDIUM → INFO.
Use named IDs: `F-NN · Title`.

Per-finding fields (adapt to what's relevant):

```markdown
#### F-01 · [one-sentence title]
**Severity:** Critical | **Category:** Data quality
**Entities:** [full key list] | **Patterns affected:** [pattern IDs]
**Detection path:** [which tool/scan discovered this]

[Mechanism — root cause explanation. For cascade findings, use a numbered
propagation chain: record → dimension → entity → population effect.]

- **Business impact:** [downstream consequence]
- **FP risk:** None / Low / Medium / High
- **Remediation:** [concrete next step or repair recommendation]
```

### Hypotheses Tested

```markdown
| # | Hypothesis | Test | Result | Verdict |
|---|-----------|------|--------|---------|
| H1 | ... | ... | ... | Confirmed / Rejected / Partially confirmed |
```

Rejected hypotheses are valuable — include them.

### Cross-Validation Table

Entity rows × analysis-type or finding-ID columns. Follow with a
summary sentence identifying the most cross-validated entity.

### False Positive Summary

Per-finding table with FP risk level + overall rate:

```markdown
| Finding | FP Risk | Confidence | Rationale |
|---|---|---|---|

**Overall estimated FP rate:** N%
```

### Recommendations

Named IDs (R-NN), grouped by time horizon, linked to finding IDs:

```markdown
**Immediate:**
- R-01: [action] — resolves F-01
- R-02: [action] — resolves F-02

**Short-term:**
- R-03: [action]

**Medium-term:**
- R-04: [action]
```

### Tool Coverage

```markdown
| Tool | Calls | Purpose |
|---|---|---|
```

### Gaps & Limitations

Where the investigation was blocked, tools under-performed, or coverage
was incomplete. Helps improve both the investigation and the tooling.

### Investigation Statistics

```markdown
| Metric | Count |
|---|---|
| MCP tool calls | N |
| Findings by severity | Critical: N, High: N, Medium: N, Info: N |
| Confirmed data errors | N |
| False positive rate | N% |
| Patterns covered | N / total |
```

### Footer

```markdown
*Report produced by autonomous GDS investigation — hypertopos v{version}.*
```

---

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Follow phases when hints given | Match hint to scan, skip generic find_anomalies |
| Recommend without executing | If you write it, you must have called it first |
| Top-N only | `aggregate` + `rank_by_property` for full distribution |
| Observation without root cause | `dive_solid` + `find_counterparties` — "high burst" is not a finding |
| Skip low-displacement range | Arch/V trajectories hide at rank 100-200, not top-10 |
| Dismiss regime change as artifact | `property_filters` per segment BEFORE dismissing |
| "~50 like this, here are 5" | Enumerate ALL or `aggregate_anomalies` for group structure |
| Static-only on temporal patterns | Always run drift + time-windowed for has_temporal |

---

## Skill delegation

| Need | Skill | Section |
|---|---|---|
| Entity deep-dive, recall table, hypothesis testing | gds-investigator | Entity 360, Root cause chain |
| Event rates, Simpson's, temporal bursts, collective drift | gds-detective | Detection recipes |
| Cross-pattern, neighbor, trajectory, segment scans | gds-scanner | Scans 1-4 |
| Drift interpretation, regime change handling | gds-monitor | Drift detection, Regime changes |

Full investigation examples: [references/examples.md](references/examples.md)
Full decision framework: [references/decision-framework.md](references/decision-framework.md)
