---
name: gds-investigator
description: Anomaly investigation, root-cause tracing, entity deep-dive, and incident reconstruction via GDS geometric analysis. Use when user asks to "investigate an anomaly", "trace root cause", "why is this entity anomalous", "deep-dive into entity", "what happened to entity X", "reconstruct incident", "compare entities", or "check if this is a false positive". Requires hypertopos MCP server.
license: Apache-2.0
compatibility: Requires hypertopos MCP server. Designed for Claude Code and compatible agents.
metadata:
  author: Karol Kędzia
  version: 0.5.2
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

When tracing root causes, use `fdr_alpha=0.05` on `find_anomalies` to ensure the initial suspect list has controlled false discovery rate — chasing false positives through the full root-cause chain wastes the entire investigation budget. Use `select="diverse"` when requesting K>10 results to surface anomalies driven by different dimensions rather than clustering on one dominant failure mode; this ensures the investigation covers distinct root-cause categories. Both parameters also apply to `attract_boundary`, `find_hubs`, and `find_drifting_entities`.

**Adaptive FDR (Storey).** When the investigation needs to expand the candidate list without relaxing α, try `fdr_method="storey"` together with `p_value_method="chi2"` — the Storey LSL estimator detects when the population has a real null mass and shrinks q-values accordingly, typically recovering 10–15% more suspects at the same false-discovery guarantee. The two parameters must be set together: Storey with the default rank p-values has no effect (rank p-values are uniform by construction). If the pattern is heavily compressed (every entity looks sub-null) or saturated (every entity already exceeds the null), Storey collapses to BH — keep the default in those regimes.

**Prioritise deteriorating drift.** When `drift_direction` is `"deteriorating"`, the entity is structurally moving away from the null centre — investigation priority is higher than for equally-displaced entities with `"normalizing"` direction, which are already self-correcting. Use this to rank multiple drifting entities by risk rather than raw displacement.

---

## Root cause chain

**One-call root-cause tracing.** `trace_root_cause(primary_key, pattern_id)` returns a bounded DAG of evidence in one shot — root witness dimensions from `explain_anomaly`, an edge-counterparty branch (sorted by anomaly, not by transaction volume), a neighbour-contamination branch (with explicit `anomalous_cp_keys` and `revisits_root` clique flag), and a hub branch — replacing the manual `explain_anomaly → find_counterparties → contagion_score → π7 hub` chain below. Use it as the default first step when the user asks "why is this entity anomalous"; fall back to the manual chain only when `truncated=true` signals the bounded tree missed context you actually need.

**Parameter cheatsheet.** Default call `trace_root_cause(pk, pid)` fits 90 % of investigations. Tune when:
- **`edge_counterparty_top_n=2..5`** — expand multiple anomalous counterparties as separate subtrees. Needed when contagion shows 5+ anomalous cps and you want all of them recursively expanded, not just the single most anomalous.
- **`max_depth=3..4`** — reach grand-counterparties for deep mule chains; usually depth=2 is enough because contagion branch already summarizes the neighbourhood.
- **`branches_enabled=["neighbor_contamination"]`** — skip edge_counterparty + hub computation for a fast "is this entity network-contaminated?" query; avoids the full adjacency+π7 scans and is orders of magnitude faster than the default call. Also `["hub"]` for hub-only or `["edge_counterparty"]` for pure chain traversal.
- **`contagion_min_counterparties=5`** — raise above default 3 on noisy sub-populations where even 3-cp contagion can be statistical noise; small-N contagion=1.0 is a well-known fake alert.
- **`hub_pop_limit=<larger>`** — raise above the default on medium-sized anchor patterns where hub membership IS informative and the `π7` scan cost is acceptable; the default biases toward skipping hub for large populations.

**Evidence interpretation — the clique signals.**
- **`revisits_root` on a contagion branch** = the root entity appears in that node's anomalous counterparty list. Confirmed geometric clique — root and this cp are structurally tied and both anomalous. Raise severity one notch beyond raw contagion score.
- **`previously_seen_as_cp_of: [X, Y]` in evidence** = within the current session, other trace_root_cause calls (on X and Y) reported *this* entity as their anomalous counterparty. Strong network-centre signal even when the current trace looks isolated. Agent workflow: investigate entity, then investigate its cps, then re-check the original entity — inter-call ledger surfaces clique without diffing multiple trees.
- **`anomalous_cp_keys`** = the up-to-10 anomalous counterparty primary keys of this node. No extra `find_counterparties` call needed to drill down.
- **`truncated: true`** = more evidence candidates were dropped than fit the `max_branches` / `max_total_nodes` caps. Rerun with higher caps or narrower `branches_enabled` to recover the dropped signal.

**Session cache.** `trace_root_cause` caches `contagion_score` and `find_counterparties` per `(pattern_version, entity_key)` at the navigator instance level. Repeat traces on the same or overlapping entities in one session hit the cache and return near-instantly; the first cold trace on a large anchor pattern is the one that pays the adjacency-scan cost. Capped with LRU eviction to bound memory. Pattern rebuild invalidates automatically via version key.

**Sample tree — structure template + concrete example:**

Template — how the fields nest:

```
root: <entity_key> (severity=extreme)
  top_dimensions: [{dim, kind, bregman, pct_of_total}, ...]
  children:
    - edge_counterparty: <cp_key_A> (severity=moderate)
        via_dim: <witness_dim_from_root>
        witness_counterparty_delta_rank_pct: <rank>
        children:
          - neighbor_contamination: <cp_key_A> (severity=high)
              anomalous_cp_keys: [<root_key>, <peer_1>, ...]   ← root in list
              revisits_root: [<root_key>]                      ← CLIQUE signal
    - neighbor_contamination: <entity_key> (severity=low)
        anomalous_cp_keys: [<cp_key_A>, <cp_key_B>]
        previously_seen_as_cp_of: [<other_entity>]             ← INTER-CALL signal
```

Concrete example — what an actual confirmed-clique trace looks like:

```json
{
  "root": {
    "entity_key": "ACC-ROOT",
    "role": "root",
    "severity": "extreme",
    "evidence": {
      "top_dimensions": [
        {"dim": "amount_out_std", "kind": "gaussian", "bregman": 67.4, "pct_of_total": 0.28},
        {"dim": "sum_in", "kind": "gaussian", "bregman": 42.6, "pct_of_total": 0.18}
      ],
      "delta_norm": 25.16,
      "conformal_p": 0.000004
    },
    "children": [
      {
        "entity_key": "ACC-MULE",
        "role": "edge_counterparty",
        "severity": "moderate",
        "evidence": {
          "via_dim": "amount_out_std",
          "witness_counterparty_delta_rank_pct": 98.09
        },
        "children": [
          {
            "entity_key": "ACC-MULE",
            "role": "neighbor_contamination",
            "severity": "high",
            "evidence": {
              "contagion_score": 0.625,
              "total_counterparties": 8,
              "anomalous_counterparties": 5,
              "anomalous_cp_keys": ["ACC-ROOT", "ACC-PEER1", "ACC-PEER2", "ACC-PEER3", "ACC-PEER4"],
              "revisits_root": ["ACC-ROOT"]
            }
          }
        ]
      },
      {
        "entity_key": "ACC-ROOT",
        "role": "neighbor_contamination",
        "severity": "low",
        "evidence": {
          "contagion_score": 0.2,
          "total_counterparties": 10,
          "anomalous_counterparties": 2,
          "anomalous_cp_keys": ["ACC-MULE", "ACC-OTHER"],
          "previously_seen_as_cp_of": ["ACC-EARLIER-SUSPECT"]
        }
      }
    ]
  },
  "summary": "Entity ACC-ROOT is extreme in account_pattern; primary witness: amount_out_std; branches found: edge_counterparty, neighbor_contamination; 3 nodes.",
  "hop_count": 2,
  "branches_explored": 3,
  "truncated": false
}
```

**Read order for pattern-matching:** `root.severity` first → `revisits_root` (direct clique) → `previously_seen_as_cp_of` (inter-call clique) → `anomalous_cp_keys` (queue for deep-dive) → `truncated` (more evidence dropped — rerun with larger caps if needed).

**Triage playbook:**
- `revisits_root` present on any contagion node → confirmed geometric clique; escalate immediately.
- `previously_seen_as_cp_of` non-empty → shared-counterparty signal across multiple suspects; high-value graph-centre node.
- `contagion_score ≥ 0.5` + `revisits_root` empty + root severity extreme → structural anomaly NOT contagion-driven; investigate root's own properties, not network.
- All child severities `"low"` + root `"extreme"` → isolated primary-source anomaly, not network mule.

### Edge-level anomaly — `edge_potential`

`edge_potential` and `attract_edge_potential` score the relationship itself, not the endpoints. Score formula: distance between endpoint delta vectors × rarity prior (1/pair_tx_count, capped). High score means two geometrically divergent entities are connected by a rare pair — the classic layering signature.

**When to use.**
- After `trace_root_cause` surfaces an `edge_counterparty` branch — check the `edge_potential` field already attached in evidence. Score above sphere-specific threshold (typically > p95 of population) reinforces the counterparty signal.
- Standalone ranking: `find_high_potential_edges(pattern_id, top_n=20, min_pair_count=1)` surfaces the most suspicious relationships in the whole pattern without committing to a specific suspect.
- Entity-scoped: `find_high_potential_edges(pattern_id, top_n=10, from_key=suspect)` ranks the suspect's own edges — useful as a drill-down from `trace_root_cause`.

**Example trace_root_cause `edge_counterparty` evidence with edge_potential:**

```json
{
  "role": "edge_counterparty",
  "entity_key": "ACC-MULE",
  "severity": "moderate",
  "evidence": {
    "via_dim": "amount_out_std",
    "witness_counterparty_delta_rank_pct": 98.09,
    "edge_potential": {
      "score": 12.5,
      "delta_distance": 5.1,
      "pair_tx_count": 1,
      "effective_weight": 1.0
    }
  }
}
```

Reading order: if both `witness_counterparty_delta_rank_pct` > 95 AND `edge_potential.score` is in the top-percentile of the pattern, you have a confirmed single-tx structurally-anomalous edge — treat as highest priority.

### Structural motifs — `score_motif` and `find_high_potential_motifs`

`score_motif` extends the edge_potential paradigm from one edge to k edges of a named structural pattern. Scoring is product-of-edge_potential across the motif's edges — a motif of rare edges is rare. Eight motif types in the closed vocabulary:

- **`cycle_2`** (default window 24h): bidirectional A↔B round-trip. Covers Flash-Burst Round-Trip and Bidirectional Burst typologies.
- **`cycle_3`** (default window 72h): directed triad A→B→C→A with strict temporal ordering. Covers Round-Tripping 3-Party, Long-Cycle, and Multi-Round-Tripping typologies.
- **`fan_out`** (default window 168h): hub → k distinct targets (min k=3). Covers Offshore Hub and Concentrator (source side) typologies.
- **`fan_in`** (default window 168h): k distinct sources → sink (min k=3). Mirror of `fan_out`. Covers Parallel Layering (destination side) and Concentrator / Sink typologies.
- **`chain_k`** (default window 168h, open directed chain of parametric length 3 ≤ k ≤ 8): A→B→…→Z with no cycle closure, no node revisit, strict monotone timestamps, total span ≤ window. Covers Multi-Stage Layering and Multi-Jurisdiction Latency Chain typologies. Tune `k` to the layering depth under investigation.
- **`structuring`** (default window 1h, amount-gated): open A→B→C→D with hop1 amount ≥ `amt1_min`, hops 2 and 3 ≤ `amt2_max`. Classic deposit-split-and-wire pattern for reporting-threshold evasion.
- **`split_recombine`** (default window 24h, `min_k` default 3, `direction="forward"|"backward"`): diamond scatter-gather S → {M₁,…,Mₖ} → D with stacked-bipartite temporal order (all split-hops precede all recombine-hops within the window). Forward mode anchors the seed as the source; backward mode anchors the seed as the sink. Covers scatter-gather smurfing, parallel layering, and concentrator/sink (backward mode) typologies — amount-free counterpart to `structuring`.
- **`bipartite_burst`** (default window 24h, `min_k` default 3, `min_m` default 3): complete K_{k,m} bipartite subgraph in a tight time window — k distinct sources each transact with every one of m distinct sinks. Greedy single-core enumeration: tries seed-as-source first, falls back to seed-as-sink. Covers coordinated mule-ring and parallel-collusion typologies; complements `fan_out` + `fan_in` by requiring completeness on both sides rather than density at a single anchor.

**When to use.**
- After `trace_root_cause` — the `edge_counterparty` branch now carries `motif_potential` automatically when the suspect seeds a motif that passes through the counterparty. Read the block alongside `edge_potential` and `witness_counterparty_delta_rank_pct`; a confirmed signal on all three means structural + per-edge + witness-dimension agreement.
- Global screening: `find_high_potential_motifs(pattern_id, motif_type="cycle_3", top_n=20)` surfaces the most suspicious triads in the whole pattern. First call per (pattern, motif_type, window, …, k) is cold (30–90s on >500k-entity patterns) — subsequent calls hit the LRU cache.
- Entity drill-down: `score_motif(suspect, motif_type="cycle_2", pattern_id)` checks whether the suspect is the seed of a high-score round-trip without committing to a specific counterparty.

**`motif_potential` block in `trace_root_cause.edge_counterparty.evidence`:**

```json
{
  "motif_potential": {
    "motif_type": "cycle_2",
    "score": 64.0,
    "time_window_hours": 24,
    "counterparty": "ACC-MULE"
  }
}
```

When `motif_type` is `cycle_3`, the block includes `ring: [seed, B, C]`. When `fan_out` or `fan_in`, it includes `k` (distinct neighbours in the window). When `chain_k`, it includes `path` (list of k keys) and `k`. When `split_recombine`, it includes `source`, `sink`, `intermediaries` (the M nodes), `k`, and `direction`. When `bipartite_burst`, it includes `sources`, `sinks`, `k`, `m`, and `seed_role` (`"source"` or `"sink"`).

`explain_anomaly` tells you WHICH dimension is anomalous, with per-dim
Bregman contributions when dimension kind tags are available. That is an
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
find_novel_entities(pattern_id) -> WHO deviates most from neighborhood expectation?
  High novelty_score = entity doesn't behave like its neighbors. Requires edge table.
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

```
cross_pattern_profile(key, line_id)     -> multi-source risk overview
goto(key, line_id) -> get_polygon(pattern_id) -> anomaly dimensions
explain_anomaly(key, pattern_id)        -> per-dim Bregman contributions + kind tags
  If bregman_contribution present: prefer it over abs_delta for root-cause ranking.
  Each dim shows kind (gaussian/poisson/bernoulli) and pct_of_total.
  Focus on dims with highest pct_of_total, not just highest abs_delta.
  Interpret by kind: poisson dim unusual = count structure anomaly;
    gaussian dim extreme = magnitude anomaly; bernoulli dim = binary flag fired.
anomaly_confidence (from get_polygon or find_anomalies result):
  >= 0.8  -> stable anomaly: high-confidence finding, proceed with full investigation
  0.3-0.8 -> borderline: investigate but maintain FP possibility in assessment
  < 0.3   -> likely false positive: verify with additional sources before escalating
  (confidence is absent for populations > 50K, group_by_property, or use_mahalanobis patterns)
dive_solid(key, pattern_id)             -> temporal history
find_similar_entities(key, pattern_id)  -> geometric neighbors
  dim_mask=[<dims from anomaly_dimensions>] -> focus on driving dims
  metric="cosine" -> shape similarity ignoring magnitude
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
- Use `metric="cosine"` when comparing anomaly profile shape regardless of severity — "same type of anomaly, different scale"
- Use `dim_mask` to focus similarity on dimensions from `anomaly_dimensions` output — finds entities similar only in the dimensions that drive the anomaly, ignoring irrelevant ones
- Use `find_anomalies(metric="Linf")` to catch single-dimension spikes that L2 norm dilutes — entities with one extreme dimension but normal on others
- Use `find_anomalies(metric="bregman")` to rank by distribution-aware Bregman divergence — better ranking on mixed-type patterns (counts + amounts + binary flags). Particularly effective when `dimension_kinds` shows a mix of poisson/gaussian/bernoulli
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
| `is_anomaly` tells you THAT, not WHY | Check `anomaly_dimensions` via explain_anomaly; use `bregman_contribution` + `kind` when available |
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
