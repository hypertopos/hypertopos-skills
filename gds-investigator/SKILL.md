---
name: gds-investigator
description: Anomaly investigation, root-cause tracing, entity deep-dive, and incident reconstruction via GDS geometric analysis. Use when user asks to "investigate an anomaly", "trace root cause", "why is this entity anomalous", "deep-dive into entity", "what happened to entity X", "reconstruct incident", "compare entities", or "check if this is a false positive". Requires hypertopos MCP server.
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.9.0
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
`extract_chains`, `find_chains_for_entity`) and note
"no edge data available" in the report.

### Sphere health gates

**Pattern-level auditors in `dim_quality_warnings`** — beyond `dead_dim` and `sparse_dim` (per-dim signals), `sphere_overview` also surfaces `dominant_dim_mass` (pattern-level: one dim accounts for ≥70% of tail variance — sphere is effectively a one-dim detector on this pattern), `negative_space` (gaussian-declared dims that empirically sit at zero — the gaussian z-score is wrong for them), `non_normal_dim` (per-dim: a `kind='gaussian'` dim has Shapiro-Wilk / KS `p < 0.01` at build time — the z-score percentile semantics are unstable; recommend `log1p` / `sqrt` / `rank` transform or kind re-declaration), and `heteroscedasticity` (pattern-level: Brown-Forsythe Levene `p < 0.01` on `delta_norm` partitioned by `group_by_property` — the global θ assumption is statistically violated, so a single global threshold produces unequal false-positive rates per group). When `dominant_dim_mass` fires, cross-check by inspecting `reliability_flags.single_dim_driven` incidence on `find_anomalies` top-N for that pattern — pattern-level dominance ⟹ high per-polygon incidence agreement. When `negative_space` fires, treat the affected dim's contribution to `delta_norm` as suspect and prefer `kind='bernoulli'` / `kind='poisson'` re-declaration over per-entity drill-down. When `non_normal_dim` fires (and `negative_space` does not), the dim's z-score is a poor anomaly scorer because the empirical distribution is heavy-tailed; raise a calibration ticket — variance-stabilizing transform in the column source script + rebuild fixes it. When `heteroscedasticity` fires, the pattern already carries per-group calibration via `group_by_property` (the warning is confirmation it was statistically warranted), but cross-group anomaly score comparison is suspect; use `aggregate(event_pattern_id, group_by_line=<related_line>, group_by_property="<line_id>:<prop_col>")` to inspect per-group anomaly rates before drawing conclusions across the whole pattern.

**Label-aware dim audit (`audit_pattern_dims`)** — when the sphere carries a `label_audit:` block in `sphere.yaml` (`label_aware_available: true` on `get_sphere_info`), call `audit_pattern_dims(pattern_id, top_k=10)` to surface label-discriminating dims. The tool returns per-dim raw mu/sigma plus class-conditioned `mu_pos` / `sigma_pos` / `mu_neg` / `sigma_neg`, `cohens_d_pos_neg`, and the per-dim component of the Fisher LDA direction vector. Each dim carries `recommended_action` ∈ {`keep`, `split`, `drop_low_separation`, `investigate_drift`, `kind_mismatch_review`}. Use to: (a) confirm which dims are actually driving the positive class — high `|direction_component|` ⟹ informative axis; (b) flag dims with high `cohens_d_pos_neg` but low `|direction_component|` as candidates for `split` (compound signal — the raw separation is real but the dim conflates two regimes); (c) flag `drop_low_separation` candidates pre-rebuild (`|cohens_d| < 0.1` ⟹ dim contributes noise to delta vector). Spheres without `label_audit:` return a fallback shape with raw stats and `recommended_action: keep` — still useful for surfacing per-dim mu/sigma ranking without label semantics.

---

## Investigation checklist

Copy this list and tick it off per investigation:

- [ ] `edge_stats` checked; edge-dependent tools skipped when it returns null
- [ ] Suspects chosen (ground-truth recall table, or FDR-controlled `find_anomalies`)
- [ ] Each suspect gated (`reliability_flags`, `assess_anomaly_certainty`) before a case opens
- [ ] Root cause traced and the repair set from `explain_anomaly` reported
- [ ] At least 3 hypotheses tested; rejected ones kept in the report
- [ ] Every finding classified CONFIRMED / SUSPECTED / FALSE POSITIVE; FP rate stated
- [ ] `investigation_coverage` run before closing

---

## Investigating with ground truth

If the entity line has a labeled outcome column (e.g., status, outcome, label),
this is the highest priority approach:

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

**Lower-level p-value composition** across detectors (`combine_anomaly_pvalues`,
`classify_detector_consensus`) is in [references/population-analyses.md](references/population-analyses.md).

## Investigating without ground truth

```
anomaly_summary on each pattern -> which has strongest signal?
find_anomalies(best_pattern, top_n=10) -> top suspects
  Optional: find_anomalies(best_pattern, top_n=10, min_confidence=0.7)
    -> filter to entities where bootstrap confidence >= 0.7
    -> reduces the FP investigation burden when population <= 50K
Check BOTH ends: if top anomalies all have negative deltas (below mean),
  use attract_boundary(alias, pattern, direction="in") to find entities
  at the positive extreme. Anomaly = extreme in EITHER direction.
For top 3 suspects: entity 360 (below)
Form hypotheses about what drives anomalies, test them
```

---

## FDR control and diverse selection

Start the suspect list with `fdr_alpha=0.05` on `find_anomalies` and use `select="diverse"` for K>10.
Storey FDR, drift-direction priority and `decompose_drift` are in
[references/tuning-and-troubleshooting.md](references/tuning-and-troubleshooting.md).

---

## Root cause chain

**One-call root-cause tracing.** `trace_root_cause(primary_key, pattern_id)` returns a bounded DAG of evidence in one shot — root witness dimensions from `explain_anomaly`, an edge-counterparty branch (sorted by anomaly, not by transaction volume), a neighbour-contamination branch (with explicit `anomalous_cp_keys` and `revisits_root` clique flag), and a hub branch — replacing the manual `explain_anomaly → find_counterparties → contagion_score → π7 hub` chain below. Use it as the default first step when the user asks "why is this entity anomalous"; fall back to the manual chain only when `truncated=true` signals the bounded tree missed context you actually need.

Read [references/root-cause-tools.md](references/root-cause-tools.md) when tuning or reading a trace:
parameter cheatsheet, clique signals, session cache, a sample tree, and the edge-level
(`edge_potential`) and structural-motif (`score_motif`) evidence on `edge_counterparty` branches.

**Read order for pattern-matching:** `root.severity` first → `revisits_root` (direct clique) → `previously_seen_as_cp_of` (inter-call clique) → `anomalous_cp_keys` (queue for deep-dive) → `truncated` (more evidence dropped — rerun with larger caps if needed).

**Triage playbook:**
- `revisits_root` present on any contagion node → confirmed geometric clique; escalate immediately.
- `previously_seen_as_cp_of` non-empty → shared-counterparty signal across multiple suspects; high-value graph-centre node.
- `contagion_score ≥ 0.5` + `revisits_root` empty + root severity extreme → structural anomaly NOT contagion-driven; investigate root's own properties, not network.
- All child severities `"low"` + root `"extreme"` → isolated primary-source anomaly, not network mule.

`explain_anomaly` tells you WHICH dimension is anomalous: each dim's share
of `delta_norm²` (`pct_of_total`) with its kind tag. That is an
observation, not a finding.

Before opening a case on any anomaly, check `reliability_flags` on the
returned polygon (surfaced by `find_anomalies`, `explain_anomaly`,
`composite_risk`, `combine_anomaly_pvalues`, and `investigate_entity`).
Two flags fire independently:

- `single_dim_driven=true` — one dim contributes >70 % of total anomaly
  attribution. Likely a data-quality artefact (saturated counter,
  outlier on one property) rather than a multi-dim fraud signal. Sanity-
  check the `dominant_dim` value before escalating.
- `low_confidence_bucket=true` — bootstrap-derived `anomaly_confidence`
  is below 0.5. The anomaly flag is fragile to population resampling —
  the entity is on the borderline. Treat as a soft hit; corroborate
  with one more detector (a chain pattern, a witness-cohort overlap, a
  counterparty signal) before opening a case.

`reliability_flags.dominant_dim` always agrees with the top entry of
`explain_anomaly.top_dimensions` for the same polygon — both surfaces
route through the same per-dim contribution primitive. If they disagree
on a real call, that's a bug to report.

**Pre-case certainty gate — `assess_anomaly_certainty`.** Where
`reliability_flags` reports the two soft-hit flags individually, the
`assess_anomaly_certainty(primary_key, pattern_id)` composer rolls them up with
FDR-alpha stability, calibration staleness, and cross-pattern consistency into a
single `certainty_verdict` — the one-call gate to run before opening an
investigation on a single-detector hit.

```
assess_anomaly_certainty(primary_key, pattern_id)
-> certainty_verdict: "high" | "moderate" | "low" | "contested"
-> certainty_score, conformal_p, signed_confidence,
   stability_across_alphas, reliability_flags
   (single_dim_driven, near_data_boundary, calibration_stale),
   cross_pattern_consistency, recommended_next_steps
```

`certainty_verdict` is certainty about the CLASSIFICATION, not about anomaly
status — a confidently-normal entity scores `"high"` too. Gate routing:

- **`high`** — stable across the swept FDR alphas, not single-dim-driven, not
  near the data boundary, calibration fresh. Open the case and run the full
  root-cause chain.
- **`moderate`** — partial stability; investigate but keep the FP possibility
  open. Corroborate with one more detector before classifying CONFIRMED.
- **`low`** — fragile (one-alpha hit or boundary-adjacent). Treat as a soft hit;
  do not open a formal case on this alone.
- **`contested`** — the calibrated `conformal_p` disagrees with the stored
  anomaly flag, or the entity is both single-dim-driven and near-boundary.
  Highest FP risk: resolve the conflict (re-read `explain_anomaly`, inspect
  `near_data_boundary`, check `cross_pattern_consistency`) before deciding.

Use this as the gate that feeds the CONFIRMED / SUSPECTED / FALSE POSITIVE
classification later in the report: `high` + multi-source ⟹ CONFIRMED candidate;
`contested` ⟹ FALSE POSITIVE candidate until the conflict resolves. Pair with
`find_diverse_explanations` when the verdict is `contested` because
`single_dim_driven` fired.

When `single_dim_driven` fires, or the sphere declares `conformance_rules:`, see
[references/root-cause-tools.md](references/root-cause-tools.md) for `find_diverse_explanations`
and the `find_conformance_violations` cross-check.

The root cause chain goes deeper:

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
find_novel_entities(pattern_id) -> WHO deviates most from neighborhood expectation?
  High novelty_score = entity doesn't behave like its neighbors. Requires edge table.
```

Tension, topology and counterfactual tools (`find_graph_geometry_tension`, `find_topological_anomalies`,
`simulate_edge_removal`, `simulate_counterparty_removal`, `select_minimal_joint_edge_removal`,
`simulate_dimension_change`) answer WHICH edges, counterparties or dim values made the entity
anomalous — see [references/root-cause-tools.md](references/root-cause-tools.md).

```
find_witness_cohort(key, pattern_id) -> peers with similar anomaly profile
```

**Graph confirmation chain** (when edge table exists):
After `explain_anomaly`, check graph support to distinguish isolated anomalies from patterns:
1. `contagion_score(key, pattern_id)` — what fraction of neighbors are anomalous?
2. `find_witness_cohort(key, pattern_id)` — which peers share the anomaly profile?
3. `find_novel_entities(pattern_id, top_n=10)` — who deviates most from neighborhood?

High contagion + large witness cohort = confirmed pattern (not isolated outlier).

**As-of graph reconstruction:** for incident forensics, all six edge-table graph primitives (`contagion_score`, `contagion_score_batch`, `entity_flow`, `degree_velocity`, `propagate_influence`, `find_counterparties`) accept an optional `timestamp_cutoff` parameter (Unix seconds). When set, only edges with `timestamp <= cutoff` are considered. Use this to answer *"what did the neighborhood look like at time T?"* — e.g. pass the incident timestamp to contagion_score to see contamination state on that day, not today.

Always report the repair set from `explain_anomaly`. Format:
"Repair: zero {repair_size} dims ({labels}) -> residual {residual_norm} (below theta={theta_norm})."

---

## Entity 360 (deep-dive on one entity)

List tools answer in the compact format by default: an `omitted` map means numeric property columns or positional `delta` vectors were left out on purpose. For one entity's full numbers use `get_polygon` / `explain_anomaly` below; for a whole list repeat the call with `summary=false`.

```
cross_pattern_profile(key, line_id)     -> multi-source risk overview
goto(key, line_id) -> get_polygon(pattern_id) -> anomaly dimensions
explain_anomaly(key, pattern_id)        -> top_dimensions [{dim, kind, delta, pct_of_total}]
  pct_of_total is the dim's share of delta_norm² (the metric behind is_anomaly);
  top_dimensions[0] is the dim whose removal drops delta_norm most.
  Each dim shows kind (gaussian/poisson/bernoulli) and signed delta.
  Interpret by kind: poisson dim unusual = count structure anomaly;
    gaussian dim extreme = magnitude anomaly; bernoulli dim = binary flag fired.
anomaly_confidence (from get_polygon or find_anomalies result):
  >= 0.8  -> stable anomaly: high-confidence finding, proceed with full investigation
  0.5-0.8 -> borderline: investigate but maintain FP possibility in assessment
  < 0.5   -> likely false positive: verify with additional sources before escalating
  (confidence is absent for populations > 50K, group_by_property, or use_mahalanobis patterns)
dive_solid(key, pattern_id)             -> temporal history
find_similar_entities(key, pattern_id)  -> geometric neighbors
  dim_mask=[<dims from anomaly_dimensions>] -> focus on driving dims
  metric="cosine" -> shape similarity ignoring magnitude
find_counterparties(key, event_line, from_col, to_col, pattern_id) -> network + amount aggregates
entity_flow(key, pattern_id)           -> net flow per counterparty (source/sink/mule)
anomalous_edges(key, counterparty, pattern_id) -> event-level scoring of specific transactions
extract_chains(event_pattern_id, seed_nodes=[key])  -> exact chain extraction from this entity (no pre-built chains needed)
  Uses every event for the seeded entity (no subsampling). Use when chain lines
  are unavailable or you want chains from a specific entity.
  Tune min_hops (default 2) and bidirectional=True for undirected search.
```

When the sphere has a chain anchor pattern (built from `chain_lines:`), the chain-coherent
loop tools are listed in [references/root-cause-tools.md](references/root-cause-tools.md).

The one-call chain orchestrators (`investigate_chain`, `chain_full_loop_summary`) grade a chain `strong` / `moderate` / `weak`, or `inconclusive` when steps failed and their missing evidence could change the grade (listed in `summary.failed_steps`). Treat `inconclusive` as "not yet decided", never as a cleared chain — run the failed steps one by one with the granular chain tools before writing a verdict.

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

## Cross-pattern lead-lag

For drift across two anchor patterns over the SAME entity line, use `find_lead_lag` — the
default call and how to read it are in [references/population-analyses.md](references/population-analyses.md).

## False positive assessment

For every anomaly finding, consider whether it is a true positive or false positive:

- High `delta_norm` can mean "large legitimate entity" — check business properties
- `find_similar_entities(key, pattern, filter_expr="is_anomaly = false", top_n=20)`
  — if 10+ normal entities have same shape, it is likely a false positive
- Use `metric="cosine"` when comparing anomaly profile shape regardless of severity — "same type of anomaly, different scale"
- Use `dim_mask` to focus similarity on dimensions from `anomaly_dimensions` output — finds entities similar only in the dimensions that drive the anomaly, ignoring irrelevant ones
- Use `find_anomalies(metric="Linf")` to catch single-dimension spikes that L2 norm dilutes — entities with one extreme dimension but normal on others
- Read `bregman_divergence` on `find_anomalies` results (distribution-aware) next to `delta_norm` — the better signal on mixed-type patterns (counts + amounts + binary flags), particularly when `dimension_kinds` shows a mix of poisson/gaussian/bernoulli
- Dormant/inactive entities being anomalous is expected, not a finding

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

## Investigation Memory

Maintain three lists throughout the investigation session:

- **`checked[]`** — entities where full investigation is complete (goto + get_polygon + explain_anomaly + counterparties done). Never re-investigate.
- **`leads[]`** — entities flagged by tools but not yet investigated. Each lead carries a `lead_score` (see Decision Scoring). Sources: `find_anomalies`, `passive_scan`, `find_witness_cohort`, `propagate_influence`, `investigation_coverage.unexplored_anomalous`.
- **`dead_ends[]`** — entities investigated and found uninteresting (`delta_rank_pct < 70`, no contagion, no temporal signal). Never revisit.

**Protocol:**
1. Before investigating any entity: check `checked[]` and `dead_ends[]`. Skip if present.
2. After each entity investigation: move from `leads[]` to `checked[]`.
3. After each tool call that returns entity lists: score new entities, add to `leads[]` (deduplicating against all three lists).
4. Call `investigation_coverage(pk, pattern_id, explored_keys=checked)` after every deep-dive. If `coverage_pct < 0.5` and `unexplored_anomalous` is non-empty, add those to `leads[]`.
5. When delegating to another skill, pass `checked[]` as context.

---

## Failure Guards

Proactive limits to prevent runaway investigations:

| Guard | Threshold | Action |
|-------|-----------|--------|
| **Depth limit** | 3 hops from seed entity | Stop expanding, summarize findings |
| **Strength gate** | `delta_rank_pct < 70` | Skip entity UNLESS `contagion_score > 0.3` or in witness cohort |
| **Contagion gate** | `contagion_score < 0.2` | Do NOT proceed to network expansion — entity is isolated |
| **Consecutive call limit** | 3 calls to same tool on same entity | Move to next lead |
| **Stale lead expiry** | Lead untouched for 10+ tool calls | Demote below fresh leads |
| **Force-switch** | 5 consecutive calls with no new anomalous entities | STOP current thread, switch to highest-scoring lead |

---

## Decision Scoring

Rank leads by composite score to decide what to investigate next:

```
lead_score = 0.35 × anomaly_strength
           + 0.25 × graph_support
           + 0.25 × temporal_signal
           + 0.15 × novelty_bonus
```

| Component | Source | Value |
|-----------|--------|-------|
| `anomaly_strength` | `delta_rank_pct / 100` | 0.0–1.0 |
| `graph_support` | `contagion_score` | 0.0–1.0 (0 if unchecked) |
| `temporal_signal` | appears in `find_drifting_entities` or `detect_trajectory_anomaly` | 0.0 or 1.0 |
| `novelty_bonus` | appears in `find_novel_entities` or `find_witness_cohort` | 0.0 or 1.0 |

**Protocol:**
1. Always investigate the highest-scoring lead next.
2. After each investigation, update scores of remaining leads (new contagion info may change `graph_support`).
3. Report queue state: `"Next: <entity> (score X.XX) | Queue: N leads remaining"`

---

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| `is_anomaly` tells you THAT, not WHY | Check `top_dimensions` via explain_anomaly: `pct_of_total` (share of `delta_norm²`) ranks the drivers, `kind` says what type of signal each one is |
| `find_similar_entities` finds shape twins, not transaction partners | Use `find_counterparties` for network relationships |
| Using `find_chains_for_entity` when chain lines do not exist | Use `extract_chains(event_pattern_id, seed_nodes=[key])` — works without pre-built chains, uses every event for the seeded entity |
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

Empty results, tool errors, no anomalies, or missing temporal data: see
[references/tuning-and-troubleshooting.md](references/tuning-and-troubleshooting.md).

---

## Skill delegation

| Need | Skill |
|---|---|
| Event rates, Simpson's, temporal bursts, drift recipes | gds-detective |
| Cross-pattern, neighbor, trajectory, segment scans | gds-scanner |
| Drift interpretation, regime change handling | gds-monitor |
| Orientation, profiling, clustering | gds-explorer |

Full investigation examples: [references/examples.md](references/examples.md)

## Hidden influencers and anomaly by absence

`find_calibration_influencers(classify="hidden")` finds entities that shape what "normal"
means; `find_density_gaps` finds dim combinations that should be populated but are not.
Both are in [references/population-analyses.md](references/population-analyses.md).
