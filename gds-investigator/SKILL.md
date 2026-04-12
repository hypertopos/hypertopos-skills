---
name: gds-investigator
description: Anomaly investigation, root-cause tracing, entity deep-dive, and incident reconstruction via GDS geometric analysis. Use when user asks to "investigate an anomaly", "trace root cause", "why is this entity anomalous", "deep-dive into entity", "what happened to entity X", "reconstruct incident", "compare entities", or "check if this is a false positive". Requires hypertopos MCP server.
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.2.3
  mcp-server: hypertopos
---

# GDS Investigator

A GDS investigator traces anomalies to their root cause — determining not
just THAT an entity is anomalous, but WHICH dimension drives it, WHEN
the deviation started, WHAT caused it, and HOW it differs from normal peers.

The goal is findings with evidence chains, not observations. Every finding
should connect detection to root cause to recommended action. A report with
20 detections and 0 root causes is a failure.

Detection recipes (event rates, Simpson's paradox, temporal bursts, drift)
are in the companion skill **gds-detective**. Read it alongside this one.

For concrete tool output examples, see [references/examples.md](references/examples.md).

---

## Core principle

Find the best pattern. Go deep. Validate against ground truth if available.
20 calls on the best pattern beats 5 calls on 4 patterns.

Your report should include a recall/precision table when ground truth exists.
Without it, the investigation is incomplete.

### Prerequisites check

Before network-dependent investigation steps (counterparties, paths, chains),
verify the pattern has edge data:

```
edge_stats(pattern_id) -> row_count, unique_from, unique_to, avg_degree
```

If `edge_stats` returns `null`, the pattern has no edge table — skip all
edge-dependent tools (`find_counterparties`, `find_geometric_path`,
`discover_chains`, `extract_chains`, `find_chains_for_entity`) and note
"no edge data available" in the report.

---

## Investigating with ground truth

If the entity line has labeled outcomes (loan_status, is_default, churn_flag,
fraud_flag, risk_tier), this is the highest priority approach:

```
get_line_profile(line_id, label_property)  -> distribution of labels
search_entities(line_id, label_property, "bad_value") -> get "bad" entity keys

Batch recall check (preferred — 1 call instead of 2N):
  check_anomaly_batch(bad_keys, pattern_id)
  -> returns is_anomaly + delta_rank_pct per key + recall_if_all_bad

Run this for each pattern that covers these entities — including patterns
on OTHER entity lines that share the same primary keys.
Check get_sphere_info for all lines with matching key columns.

Fallback (if check_anomaly_batch not available):
  For each "bad" key: goto(key, line_id) -> get_polygon(pattern_id) -> check is_anomaly

Focus on the pattern with highest recall.

contrast_populations(best_pattern, {"keys": bad_keys})
-> look at MAX single-dimension |d|, not average

get_line_profile for TOP 2-3 numeric properties only (from contrast step),
with group_by=label. Do not profile every column — pick the ones with highest |d|.

Report per-pattern recall table:
| Pattern | Anomalies | "Bad" caught | Recall | Precision |
```

**Checking for missed entities:**

If any "bad" keys are missed by the best single pattern:

```
composite_risk_batch(missed_keys, line_id)
```

Threshold: `combined_p < 0.10` (relaxed vs typical 0.05).

## Investigating without ground truth

```
anomaly_summary on each pattern -> which has strongest signal?
find_anomalies(best_pattern, top_n=10) -> top suspects
Check BOTH ends: if top anomalies all have negative deltas (below mean),
  use attract_boundary(alias, pattern, direction="in") to find entities
  at the positive extreme. Anomaly = extreme in EITHER direction.
For top 3 suspects: entity 360 (below)
Form hypotheses about what drives anomalies, test them
```

---

## FDR control and diverse selection

When tracing root causes, use `fdr_alpha=0.05` on `find_anomalies` to ensure the initial suspect list has controlled false discovery rate — chasing false positives through the full root-cause chain wastes the entire investigation budget. Use `select="diverse"` when requesting K>10 results to surface anomalies driven by different dimensions rather than clustering on one dominant failure mode; this ensures the investigation covers distinct root-cause categories. Both parameters also apply to `attract_boundary`, `find_hubs`, and `find_drifting_entities`.

---

## Root cause chain

`explain_anomaly` tells you WHICH dimension is anomalous. That is an
observation, not a finding. The root cause chain goes deeper:

```
explain_anomaly -> dominant dimension identified
dive_solid -> WHEN did this dimension change?
  If dive_solid returns no data, skip — report temporal analysis unavailable.
  If edge table available: degree_velocity(key, pattern_id) -> did connection rate also change?
get_event_polygons or find_counterparties -> WHAT happened in that period?
  entity_flow(key, pattern_id) -> net flow summary per counterparty (source/sink/mule role)
  contagion_score(key, pattern_id) -> what fraction of neighbors are anomalous?
compare_entities with a normal peer -> HOW does this entity differ?
find_geometric_path(from_key, to_key, pattern_id) -> HOW are two anomalous entities connected?
  Use when two entities share anomaly dimensions — traces the geometric path between them.
  scoring="geometric" (default) ranks by delta coherence; "anomaly" ranks by anomaly density along path.
```

**As-of graph reconstruction:** for incident forensics, all six edge-table graph primitives (`contagion_score`, `contagion_score_batch`, `entity_flow`, `degree_velocity`, `propagate_influence`, `find_counterparties`) accept an optional `timestamp_cutoff` parameter (Unix seconds). When set, only edges with `timestamp <= cutoff` are considered. Use this to answer *"what did the neighborhood look like at time T?"* — e.g. pass the incident timestamp to contagion_score to see contamination state on that day, not today.

Always report the repair set from `explain_anomaly`. Format:
"Repair: zero {repair_size} dims ({labels}) -> residual {residual_norm} (below theta={theta_norm})."

---

## Entity 360 (deep-dive on one entity)

```
cross_pattern_profile(key, line_id)     -> multi-source risk overview
goto(key, line_id) -> get_polygon(pattern_id) -> anomaly dimensions
dive_solid(key, pattern_id)             -> temporal history
find_similar_entities(key, pattern_id)  -> geometric neighbors
find_counterparties(key, event_line, from_col, to_col, pattern_id) -> network + amount aggregates
entity_flow(key, pattern_id)           -> net flow per counterparty (source/sink/mule)
anomalous_edges(key, counterparty, pattern_id) -> event-level scoring of specific transactions
discover_chains(key, pattern_id)        -> runtime chain discovery (no pre-built chains needed)
  Works directly on the edge table via temporal BFS. Use when chain lines
  are unavailable or you want chains from a specific entity without full extract.
  Tune min_hops (default 2) and direction ("forward"/"backward"/"both").
```

`source_count >= 2` = investigate thoroughly. `source_count == 1` = likely FP.

---

## Hypothesis testing

For every major finding, form an explicit hypothesis and test it:

```
### Hypothesis: [statement]
**Test:** [tool call + what you're checking]
**Result:** [what the tool returned]
**Verdict:** Confirmed / Rejected / Partially confirmed
```

Example: "Entity X drives network-wide anomaly spread."
Test: `propagate_influence([X], pattern_id, max_depth=3)` — if 50+ affected entities with decaying influence scores, confirmed.

Minimum 3 cycles per investigation. Rejected hypotheses are valuable.

---

## Population contamination

If top-K anomalies are dominated by entities that LACK the ground truth property:

1. The pattern population is too broad
2. Check: what % of anomalies have the GT property? If <30%, ranking is contaminated
3. Filter recall analysis to only entities that HAVE the property
4. Recommend: "Build a filtered pattern on the relevant subpopulation"

## False positive assessment

For every anomaly finding, consider whether it is a true positive or false positive:

- High `delta_norm` can mean "large legitimate entity" — check business properties
- `find_similar_entities(key, pattern, filter_expr="is_anomaly = false", top_n=20)`
  — if 10+ normal entities have same shape, it is likely a false positive
- Dormant/inactive accounts being anomalous is expected, not a finding

Classify every finding as:
- **CONFIRMED** — 3+ tools confirm, multi-source, root cause identified
- **SUSPECTED** — 1-2 tools confirm, plausible but not proven
- **FALSE POSITIVE** — explained by artifact, structure, or domain expectation

At investigation end, compute and state the overall FP rate:
`FP rate = (# FALSE POSITIVE findings) / (# total findings)`.
Include this in the report summary.

## Reading contrast_populations

- Look at the **TOP dimension** by |d|, not the average
- |d| > 0.8 = large effect, |d| > 1.5 = very large
- Positive d = group_a higher, negative = group_b higher

---

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| `is_anomaly` tells you THAT, not WHY | Check `anomaly_dimensions` via explain_anomaly |
| `find_similar_entities` finds shape twins, not transaction partners | Use `find_counterparties` for network relationships |
| Using `find_chains_for_entity` when chain lines do not exist | Use `discover_chains` — works on edge table directly, no pre-built chains needed |
| Calling edge-dependent tools without checking edge availability | Run `edge_stats(pattern_id)` first — null means no edges |
| `goto()` then `get_polygon()` in parallel | Always sequential — goto sets position, get_polygon reads it |
| Binary geometry has identical deltas for all anomalies | Use aggregate counts instead |
| "High burst" without root cause | `dive_solid` + `find_counterparties` to trace the why |
| Closing investigation without checking network coverage | `investigation_coverage(key, pattern_id, explored_keys)` — confirms explored vs unexplored counterparties |

---

## Output format

```
## FINDING: [one sentence]
## EVIDENCE: [tool -> result -> interpretation]
## CONFIDENCE: HIGH / MEDIUM / LOW
## FP RISK: None / Low / Medium / High
## REMEDIATION: [concrete next step or repair recommendation]
```

HIGH = 3+ tools confirm + multi-source + root cause identified.
MEDIUM = 2 tools confirm + single-source but strong signal.
LOW = 1 tool confirms + no multi-source + root cause unclear.

Every finding must include a concrete remediation step — not just
"investigate further" but a specific action the data owner can take
(e.g., "correct record X", "filter dimension Y above threshold Z",
"segment population by property W before analysis").

---

## When things don't work

- **Tool returns empty results** — try different parameters, wider sample,
  or a different pattern. Adjust `top_n`, `limit`, or filters.
- **Tool errors** — check pattern_id and version, verify sphere is open.
  If the error persists, report it rather than retrying in a loop.
- **No anomalies found** — not every sphere has every anomaly type.
  Report "no signal detected" as a valid finding.
- **check_anomaly_batch returns all normal for "bad" keys** — the pattern
  may not capture the relevant signal. Try other patterns or composite_risk.
- **dive_solid returns no temporal slices** — temporal data may not exist
  for this pattern. Skip temporal root cause and note it in the report.
- **contrast_populations shows small effect sizes** — the anomaly may be
  spread across many dimensions rather than concentrated in one. Check
  individual dimensions rather than relying on the top-1.

---

## Skill delegation

| Need | Skill |
|---|---|
| Event rates, Simpson's, temporal bursts, drift recipes | gds-detective |
| Cross-pattern, neighbor, trajectory, segment scans | gds-scanner |
| Drift interpretation, regime change handling | gds-monitor |
| Orientation, profiling, clustering | gds-explorer |

Full investigation examples: [references/examples.md](references/examples.md)
