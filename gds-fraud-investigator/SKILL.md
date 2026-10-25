---
name: gds-fraud-investigator
description: AML/fraud detection and financial anomaly investigation via GDS geometric analysis — 3-phase process with 25 typology recipes. Use when user asks to "detect fraud", "screen for money laundering", "find suspicious accounts", "run AML scan", "investigate suspicious transactions", "typology detection", "passive scan for fraud", "detect insider trading", "find anomalous billing", or "financial crime detection". Use this skill for ANY financial anomaly investigation, not just AML. Requires hypertopos MCP server with a financial transaction sphere (account + pair + chain patterns).
license: Apache-2.0
compatibility: Requires hypertopos MCP server with a financial transaction sphere (account, pair, chain patterns).
metadata:
  author: Karol Kędzia
  version: 0.9.0
  mcp-server: hypertopos
---

# GDS Fraud Investigator Skill

Specialized AML/fraud investigation using GDS geometric navigation. Requires an active
hypertopos MCP session with a financial transaction sphere.

## Core Principle

GDS fraud investigation is a **3-phase process**: build right, scan passively, verify actively.
Active navigation does NOT increase recall — it only helps verify suspects and eliminate
false positives. The real detection power comes from multi-source passive scanning across
all geometry layers.

## Investigation checklist

Copy this list and tick it off per screening run:

- [ ] Phase 1: `passive_scan` run; suspects ordered by `cross_pattern_profile` risk
- [ ] Phase 2: each suspect through Entity 360 (`investigate_entity` or the manual steps)
- [ ] Pre-SAR gate: `assess_anomaly_certainty` and `consensus_classification` before escalation
- [ ] FP elimination run on every escalated suspect
- [ ] Phase 3: typology or R1–R11 recipe matched when the user asked for one
- [ ] Failure Guards respected; `investigation_coverage` checked before closing

## Prerequisites — Edge Table Check

Before using graph traversal tools (`find_geometric_path`, `extract_chains`), verify the
sphere has an edge table for the relevant event pattern:

```
edge_stats(pattern_id="<event_pattern>")
```

If this returns edge counts and degree stats, graph tools are available. If it errors or
returns empty, the pattern lacks from/to FK structure — fall back to `find_counterparties`
and `extract_chains` instead.

## Threshold Convention

All typology recipes use **relative thresholds** based on population geometry, not
hardcoded amounts. This makes them dataset-agnostic and currency-independent.

- **Amount thresholds** → use `delta_rank_pct > N` or `conformal_p < X` instead of fixed amounts
- **Count thresholds** → use population percentiles (e.g. "top 5% by tx_count" = `delta_rank_pct > 95`)
- **"High"/"significant"** → means above the pattern's anomaly boundary (`is_anomaly = true`)
- **"Many"/"few"** → relative to entity's own segment (per-group theta handles this automatically)

When a recipe says "total_amount in top percentile for population", interpret as: "total_amount in top percentile
for this population". Use `get_polygon` → `anomaly_dimensions` to see which dims drive
the anomaly — the geometry already normalizes for scale.


## Phase 1 — Screening (start here)

### Recipe: Multi-Source Passive Scan

One call screens the entire population:

```
passive_scan("<anchor_line>", threshold=2)
```

This reads all geometry layers (account, pair, chain patterns) and returns entities
flagged by 2+ independent sources. `threshold=2` = multi-source confirmed.

**passive_scan source types:**
- `geometry` (default) — anomaly flag from pattern geometry
- `borderline` — near-threshold entities (rank >= threshold, not flagged)
- `points` — entity column rules without geometry (e.g., multi-currency filter)
- `compound` — geometry expansion intersected with column rules

Example — multi-source scan with mixed types:
```
passive_scan(home_line_id="<anchor_line>", sources='[
  {"type": "geometry", "pattern_id": "<anchor_pattern>"},
  {"type": "borderline", "pattern_id": "<anchor_pattern>", "rank_threshold": 80},
  {"type": "points", "line_id": "<anchor_line>", "rules": {"<suspicious_property>": [">=", 2]}, "combine": "AND"}
]')
```

Auto-discover with borderline (also auto-detects graph contagion source if edge table exists):
```
passive_scan(home_line_id="<anchor_line>", include_borderline=true, borderline_rank_threshold=80)
```

After passive_scan, batch-score the top suspects by neighborhood contamination:
```
contagion_score_batch(suspect_keys, pattern_id)
```
Entities with high contagion ratio are network hubs, not isolated actors — prioritize them.

**As-of reconstruction:** `contagion_score`, `contagion_score_batch`, `entity_flow`, `degree_velocity`, `propagate_influence`, and `find_counterparties` all accept an optional `timestamp_cutoff` parameter (Unix seconds). When set, only edges with `timestamp <= cutoff` are considered — use this to reconstruct what an entity's neighborhood looked like at the time of a known incident, or to validate a detection recipe retroactively against a prior snapshot of the graph.

If `passive_scan` is not available, manually combine:
```
1. find_anomalies("<anchor_pattern>", top_n=50)     → anomalous entities
2. For pair patterns: execute EVERY profiling_alert call (rank_by_property on each alerted dim)
   If total_found >> 50: aggregate_anomalies("<pair_pattern>", group_by="<anchor_key>")
3. find_anomalies("<chain_pattern>", top_n=50)       → anomalous chains → expand chain_keys
4. Intersect: entities in 2+ sources = high confidence
```

### FDR control and diverse selection

Screen with `fdr_alpha=0.05` and `select="diverse"` for K>10; when to switch to Storey FDR is in
[references/phase2-signals.md](references/phase2-signals.md).

### Recipe: Risk Triage

For each suspect from Phase 1:

```
cross_pattern_profile(suspect_key, line_id="<anchor_line>")
```

Interpret the result:
- `source_count >= 3` → investigate immediately (anomalous in all layers)
- `source_count == 2` → high priority
- `source_count == 1` → likely false positive, deprioritize
- `risk_score > 0.5` → high anomaly density across patterns
- `connected_risk > 80` → counterparties are also anomalous (network signal)

Sort suspects by `risk_score` descending for investigation order.

## Phase 2 — Investigation (per suspect)

> **Note:** Recipes below use angle-bracket placeholders (e.g. `<anchor_line>`, `<event_pattern>`) for sphere-specific names. Replace them with the actual line, pattern, and column names from your sphere before calling the tools.

### Recipe: Entity 360

Full investigation of a single suspect:

```
1. cross_pattern_profile(pk)              → multi-source risk overview
2. goto(pk, "<anchor_line>") then
   get_polygon("<anchor_pattern>")        → anomaly_dimensions (WHY anomalous)
3. find_counterparties(pk, "<event_line>",
     "<from_col>", "<to_col>",
     pattern_id="<anchor_pattern>")       → WHO they transact with
4. contagion_score(pk, "<event_pattern>") → what fraction of neighbors are anomalous
5. find_witness_cohort(pk, "<anchor_pattern>") → peers with similar anomaly profile
6. find_novel_entities("<event_pattern>",
     top_n=10, sample_size=1000)          → neighborhood deviation screen
6b. find_graph_geometry_tension(pk, "<anchor_pattern>",
      line_id="<event_pattern>")           → behavioural-vs-edge 2x2 cross-tab
      (hidden_cluster = lookalikes never seen together;
       suspicious_links = out-of-peer-group counterparties)
6c. find_topological_anomalies("<anchor_pattern>",
      top_n=20, k_neighbors=100)           → local H_1 cycle persistence
      (population risk screener, NOT a top-N drill-down — mid-rank lift only)
7. dive_solid(pk, "<anchor_pattern>")     → WHEN behavior changed
8. investigation_coverage(pk, "<event_pattern>",
     explored_keys=checked)               → coverage check, add unexplored to leads
```

**Graph confirmation chain (steps 4→5→6):**
- `contagion_score > 0.3` → neighborhood is infected, not an isolated outlier
- `witness_cohort_size > 3` → anomaly signature is shared by non-connected peers
- `find_novel_entities` surfaces entities whose geometry deviates from what their neighbors predict — catches entities that contagion and witness miss
- `find_graph_geometry_tension` cross-tabs behavioural k-NN × graph adjacency. On AML-class data the discriminative signal is concentrated in `n_suspicious_total` (out-of-peer-group counterparties); the `hidden_cluster` cell saturates at `k_geometric` because behavioural similarity in delta space and direct counterparties are nearly disjoint sets — treat hidden_cluster as architectural completeness, not a fraud-rank lever
- `find_topological_anomalies` ranks by local H_1 cycle persistence (raw `h1_max_persistence`, NOT the normalised `topo_score`). Its lift over the `delta_norm` baseline is mid-rank rather than top-N — use as composition input for HMP / passive_scan, not as a top-N drill-down replacement

**One-call root-cause tracing.** Instead of running steps 1–8 above manually, `trace_root_cause(suspect_pk, "<anchor_pattern>")` returns a bounded DAG of evidence in one shot — root witness dimensions, edge-counterparty branch (sorted by anomaly, not transaction volume — catches structural cps that high-volume sort would miss), neighbour-contamination branch with `anomalous_cp_keys` + `revisits_root` clique flag, hub branch when population ≤ `hub_pop_limit`. Use it as the default first pass during Phase 2 confirmation; drop into the manual chain only when `truncated=true` signals the tree missed context you need or when you explicitly want per-step control.

**One-call full entity 360.** `investigate_entity(suspect_pk, "<anchor_pattern>", line_id="<event_pattern>")` chains polygon shape + explain_anomaly + witness cohort + chains + root_cause + graph_geometry_tension into one MCP call with per-step `steps_status` (partial failures surface without aborting). Entity-side analog of `investigate_chain`. Use as default Phase 2 entry point when you want one structured report rather than chaining 6 manual calls. Pass `chain_pattern_id="<chain_pattern>"` to populate chains; `include_per_edge_counterfactual=true` wires through to `simulate_edge_removal` (covers `relations` and `edge_dim_aggregations` dims; the `count_above_threshold` aggregation is held constant, and `event_dimensions` / `prop_columns` carry no per-edge contribution by design).

**Counterfactual suite (4 tools)** — edge, counterparty, minimal-edge-set and dimension what-ifs:
see [references/phase2-signals.md](references/phase2-signals.md).

**Multi-hypothesis explainer**: `find_diverse_explanations(primary_key, pattern_id, n_hypotheses=3)` returns K diverse dim sets, each a separate "why this account is anomalous" hypothesis. Use when triaging cases where `explain_anomaly` flags `single_dim_driven=True` — the primitive's `degraded_reason="insufficient_diverse_mass"` confirms the single-dim case; multiple returned hypotheses unlock parallel SAR narratives (one per dim cluster).

### Pre-SAR triage gate — certainty + detector consensus before escalation

Before escalating a suspect to a SAR or opening a formal case, run two
agent-correctness composers as a gate. They answer "how much should I trust
this classification?" and "do my detectors agree?" — the two questions that
most often turn an escalation into a false positive.

```
assess_anomaly_certainty(primary_key, pattern_id)
-> certainty_verdict: "high" | "moderate" | "low" | "contested"
-> certainty_score, conformal_p, signed_confidence,
   stability_across_alphas, reliability_flags
   (single_dim_driven, near_data_boundary, calibration_stale),
   cross_pattern_consistency, recommended_next_steps
```

`assess_anomaly_certainty` composes `explain_anomaly`, the entity's stored
geometry meta, `find_anomalies` (FDR-stability across alphas), `sphere_overview`
(calibration staleness), and `cross_pattern_profile` into one verdict on how
confident you should be in the classification — note `certainty_verdict` is
certainty about the classification, not about anomaly status (a
confidently-normal entity also scores `"high"`). SAR-escalation routing:

- **`high`** — stable across FDR alphas, not single-dim-driven, not near the
  data boundary, calibration fresh. Solid foundation for escalation; proceed to
  the SAR narrative composers.
- **`moderate`** — partial stability. Corroborate with one more detector
  (a chain pattern, a witness-cohort overlap) before escalating.
- **`low`** — fragile: flagged at only one alpha or sitting near the data
  boundary. Treat as a soft hit; do not escalate on this alone.
- **`contested`** — the entity's calibrated conformal p-value disagrees with
  its stored anomaly flag, or it is both single-dim-driven AND near-boundary.
  Investigate the conflict (re-read `explain_anomaly`, check
  `near_data_boundary`) before any escalation decision — this is the highest
  false-positive risk class.

When the entity line has ≥2 patterns, also run the single-entity detector
consensus:

```
consensus_classification(primary_key, pattern_id)
-> classification: "anomalous_consensus" | "mixed_signal" |
   "single_detector_signal" | "normal_consensus" | "insufficient_data"
-> anomalous_detectors / normal_detectors / borderline_detectors,
   n_detectors_fired, hmp, population_rank, interpretation
```

`consensus_classification` focuses `classify_detector_consensus` on one entity
(routing the focal row without the agent re-scanning a ranked list). Read the
classification:

- **`anomalous_consensus`** — 2+ detectors agree anomalous, none dissent. A
  corroborated escalation target; combine with `assess_anomaly_certainty="high"`
  for the strongest pre-SAR posture.
- **`mixed_signal`** — detectors genuinely disagree. The hidden-mule /
  legitimate-but-extreme surface; inspect which detector dissents via
  `cross_pattern_profile` before escalating — the disagreeing lens often names
  the concealment mechanism.
- **`single_detector_signal`** — one lone detector fired; the weakest evidence
  class. Corroborate with `passive_scan` or a second pattern before opening a
  case.
- **`normal_consensus`** — deprioritise.
- **`insufficient_data`** — every detector sits in its borderline band; no
  detector resolves the entity. Widen the detector set or recalibrate.

The focal entity is always scored, even when the random sample missed it; `found=false`
means it does not exist in the pattern. Gate rule of thumb: escalate to SAR only when
`assess_anomaly_certainty` is `high`/`moderate` AND `consensus_classification`
is `anomalous_consensus` (or a `moderate`/`high` certainty corroborated by a
second pattern). A `contested` certainty or a `single_detector_signal`
consensus is a hold-and-corroborate signal, not an escalation.

Fraud-specific `trace_root_cause` tuning and the clique signals to read on every trace
(`revisits_root`, `previously_seen_as_cp_of`, `anomalous_cp_keys`) are in
[references/phase2-signals.md](references/phase2-signals.md).

High contagion (>0.3) + large witness cohort + high novelty = confirmed network pattern. Low contagion (<0.2) + empty cohort = isolated anomaly, deprioritize. Borderline contagion (0.2–0.3) = expand cautiously to one hop only before deciding.

Edge-level rare-pair detection (`find_high_potential_edges`, the dual singleton / recurring
query, `is_high_potential`) and structural motifs (`score_motif`, `find_high_potential_motifs`)
are in [references/phase2-signals.md](references/phase2-signals.md); the full motif vocabulary, defaults and
typology mapping are in [references/motif-detection.md](references/motif-detection.md).

Key signals:
- `anomaly_dimensions` / `explain_anomaly.top_dimensions` (each dim's share of `delta_norm²` + its `kind`) show WHICH behavioral features drive the detection
  - `kind: poisson` dim anomalous = count/frequency structure deviation (structuring, burst)
  - `kind: bernoulli` dim anomalous = binary flag triggered (cross-border, FX flag, unusual channel)
  - `kind: gaussian` dim extreme = magnitude anomaly (large amount, high velocity)
  - `pct_of_total` is the dim's share of `delta_norm²`; `top_dimensions[0]` is the dim whose removal drops `delta_norm` most
- Counterparties with `is_anomaly=true` = network confirmation
- Temporal burst → silence pattern = classic placement/layering

### Recipe: Network Expansion

From a confirmed suspect, expand the investigation network:

```
1. find_counterparties(suspect)           → list of transaction partners
2. Filter anomalous counterparties        → subset with is_anomaly=true
3. For each anomalous counterparty:
     cross_pattern_profile(cp_key)        → are THEY multi-source flagged?
4. extract_chains(event_pattern_id="<event_pattern>", seed_nodes=[suspect],
     time_window_hours=72, max_hops=5)    → exact chain discovery from suspect
5. Filter: chains with cyclic structure   → round-trip chains
```

**Seeded `extract_chains`:** passing `seed_nodes=[suspect]` causes every event to be used (no subsampling) — chains from that entity are exact. Use `min_hops` and `max_hops` to focus on the chain lengths relevant to the typology; use `bidirectional=True` for undirected search (e.g. round-trip detection).

**Tracing connections between two suspects:**
```
find_geometric_path(from_key=suspect_A, to_key=suspect_B,
  pattern_id="<event_pattern>", scoring="anomaly")
```
This traces how two suspects are connected through the transaction graph. `scoring="anomaly"`
prioritizes paths through anomalous intermediaries — the most suspicious route between them.
Use `scoring="geometric"` to find paths through geometrically unusual entities, or
`scoring="shortest"` for the most direct connection.

### Recipe: FP Elimination

Before closing an alert, check for exculpatory evidence:

```
1. find_similar_entities(suspect, "<anchor_pattern>",
     filter_expr="is_anomaly = false", top_n=20)
2. If 10+ normal entities have identical shape → suspect is likely
   a legitimate high-activity entity, not fraudulent
3. Check: are counterparties ALL normal? → further FP evidence
```

**Metric and dimension focus for FP elimination:**
- `metric="cosine"` — compare anomaly profile shape ignoring magnitude. Entities with same pattern but different scale will be cosine-close. Use when "same type of activity" matters more than "same scale."
- `dim_mask=[<dims from anomaly_dimensions>]` — focus similarity on the dimensions that drive the anomaly, ignoring irrelevant ones. Read the target entity's `anomaly_dimensions` first, then pass those labels as the mask.
- `find_anomalies(metric="Linf")` — rank by max single-dimension spike. Catches entities with one extreme dimension that L2 norm dilutes. Use for single-behavior typologies.
- `bregman_divergence` on every `find_anomalies` result — distribution-aware score, more telling than L2 `delta_norm` on patterns mixing counts (poisson), amounts (gaussian), and flags (bernoulli). Check `dimension_kinds` in sphere_overview — if mixed kinds, compare the two scores.

**Declarative compliance rules** — when the sphere is built with `conformance_rules:` declared on a pattern in `sphere.yaml`, `find_conformance_violations(pattern_id, severity_min="medium")` surfaces entities that broke human-authored expectations. Conformance violations are independent from `delta_norm` anomalies; an entity can be one, the other, or both. High-value workflow: `find_conformance_violations` → pick top violators → `investigate_entity(primary_key)` on each to drill into whether the rule break is also accompanied by geometric anomaly. AML adoption surface for SAR-narrative composition: cite the broken rule by `rule_id`, attach the entity's geometric explanation from `investigate_entity` as supporting evidence.

## Phase 3 — Typology Detection

25 AML typology recipes are documented in [references/typologies.md](references/typologies.md).
Load it when the user requests a specific typology or asks for a comprehensive AML scan.

Quick reference:
- Typology 1: Structured Layering — multi-hop structuring with rounded amounts
- Typology 2: Flash-Burst Round-Trip — short intense burst with return flow <24h
- Typology 3: Round-Tripping 3-Party — A->B->C->A cycle
- Typology 4: Bidirectional Burst — ping-pong between two accounts
- Typology 5: Multi-Stage Layering — 4+ hop cross-jurisdiction
- Typologies 6-25 in references/typologies.md

## Edge Table Investigation Recipes

Use these when the sphere has event patterns with edge tables (`edge_stats` returns `has_edge_table: true`).

| Recipe | Signal | When to use |
|--------|--------|-------------|
| R1 Mirror Transaction | A→B and B→A same amount same day | Circular flow detection |
| R2 Pass-Through | receive + send within 2 hours | Rapid pass-through / layering |
| R3 Burst Detection | many tx to same target in 24h | Structuring / smurfing |
| R4 Weighted Reciprocity | balanced in/out with same counterparty | Round-tripping / wash trading |
| R5 Financial Profile | entity total in/out/net flow | Risk profiling / mule detection |
| R6 Concentration Risk | single counterparty dominates flow | Over-reliance / control |
| R7 Benford's Law | first-digit distribution of amounts | Fabricated transactions |
| R8 Witness Cohort | high witness overlap + trajectory convergence, no existing edge | Fraud cohort expansion — surface peers sharing the target's anomaly signature |
| R9 Chain-Coherent Cascade | ≥ N consecutive entities along a stored chain individually anomalous on the same dominant dim | Layering paths whose chain-level shape looks normal but composition is uniformly suspicious |
| R10 Trade-Based ML | multi-currency cross-bank flows with amount mismatches | Trade-misinvoicing schemes |
| R11 Mule Network | density-based clusters of low-volume coordinated accounts | Mule-network discovery without a seed |

Which recipe runs on what input, which need the edge table, and which have a one-shot
orchestrator: see the capability matrix and per-pattern guide in
[references/recipe-index.md](references/recipe-index.md).

### Per-recipe detail

Tool sequences, interpretation rules, false-positive guards, and uniqueness rationale for every recipe live in [`references/recipes.md`](references/recipes.md) (anchors `#r1` … `#r11`). Read on demand when actually executing a recipe — the summary table above and [references/recipe-index.md](references/recipe-index.md) are the index. R1–R7 are described there in full; R8–R11 keep a one-line pattern below.

### R8 — Witness Cohort Discovery (Fraud Cohort Expansion)

**Pattern:** A confirmed launderer X has a ring of accomplices. Some are already in X's counterparty network. Others share X's anomaly signature — same witness dimensions, drifting in the same geometric direction — but are NOT yet connected to X via transactions. `find_witness_cohort` surfaces these geometric peers, with already-connected entities filtered out. **Honest scope:** investigative cohort expansion, NOT edge forecasting. Full detail in [`references/recipes.md#r8`](references/recipes.md).

### R9 — Chain-Coherent Cascade (Composition-based Layering)

**Pattern:** A stored chain (anchor pattern built from `chain_lines:` BFS extraction OR ingested from an external system per the [external-chains convention](https://github.com/hypertopos/hypertopos-py/blob/main/docs/external-chains-as-anchor-line.md)) hops through ≥ N consecutive entity-anchor positions that are individually anomalous in the entity-anchor pattern AND share the same dominant delta dimension. The chain-level shape (hop count, time span, amount decay) may look unremarkable, but the *composition* of the path — every consecutive entity flagged for the same structural reason — is the signal.

Full tool sequence (triage → flag → trace → label → extend, plus the `investigate_chain` and
`generate_sar_rationale` shortcuts) in [`references/recipes.md#r9`](references/recipes.md).

### R10 — Trade-Based Money Laundering (TBM)

**Pattern:** Multi-currency cross-bank flows with amount mismatches typical for over/under-invoicing. Discriminator: `currency_diversity >= 2` AND `cross_bank_count >= 2` along a fan-out path. **Status:** Exploratory — treat output as a candidate set, not a verdict. Full detail in [`references/recipes.md#r10`](references/recipes.md).

### R11 — Mule Network Discovery

**Pattern:** A *group* of low-volume accounts forming a dense subgraph with coordinated reciprocal flows. Detected as a cluster (not from a confirmed seed — that's R8). **Status:** Exploratory — treat output as a candidate set, not a verdict. Full detail in [`references/recipes.md#r11`](references/recipes.md).

## Investigation Memory

Maintain three lists throughout the investigation session:

- **`checked[]`** — entities where Entity 360 is complete (polygon + explain + counterparties done). Never re-investigate.
- **`leads[]`** — entities flagged by tools but not yet investigated. Each lead carries a `lead_score` (see Decision Scoring). Sources: `find_anomalies`, `passive_scan`, `find_witness_cohort`, `propagate_influence`, `investigation_coverage.unexplored_anomalous`.
- **`dead_ends[]`** — entities investigated and found uninteresting for this thread (`delta_rank_pct < 70`, no contagion, no temporal signal). Never revisit.

**Protocol:**
1. Before investigating any entity: check `checked[]` and `dead_ends[]`. Skip if present.
2. After each Entity 360: move entity from `leads[]` to `checked[]`.
3. After each tool call that returns entity lists: score new entities, add to `leads[]` (deduplicating against all three lists).
4. Call `investigation_coverage(pk, pattern_id, explored_keys=checked)` after every deep-dive. If `coverage_pct < 0.5` and `unexplored_anomalous` is non-empty, add those to `leads[]`.
5. When delegating to another skill, pass `checked[]` as context.

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

## Decision Scoring

Phase 1 Risk Triage sorts by `risk_score` for initial suspect selection. Once investigation begins and graph/temporal data becomes available, `lead_score` supersedes `risk_score` as the authoritative ordering.

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

**Triage levels** (anomaly_confidence available for populations <= 50K):
- `>= 0.7` AND `anomaly_confidence >= 0.9` — CRITICAL: investigate immediately
- `>= 0.7` AND `anomaly_confidence >= 0.7` — HIGH: investigate in current session
- `>= 0.4` AND `anomaly_confidence >= 0.5` — MEDIUM: investigate if time permits
- `< 0.4` OR `anomaly_confidence < 0.5` — LOW: skip unless explicitly asked
- (When anomaly_confidence absent, triage by lead_score alone using prior thresholds)

**Protocol:**
1. Always investigate the highest-scoring lead next.
2. After each investigation, update scores of remaining leads (new contagion info may change `graph_support`).
3. Report queue state: `"Next: <entity> (score X.XX) | Queue: N leads remaining"`

## Common Pitfalls

- `is_anomaly` alone misses most fraud — check each pattern separately and combine signals
- `find_similar_entities` returns shape twins, not new suspects — use `find_counterparties` for network expansion
- High `delta_norm` can mean legitimate high-activity entity — cross-reference with business context
- Pair pattern captures relationship anomalies invisible at account level — always check it
- `extract_chains` without `seed_nodes` causes hub monopolization — always pass `seed_nodes=[suspect]` for entity-focused chain investigation
- `find_geometric_path` with `scoring="shortest"` finds direct connections but misses suspicious intermediaries — use `scoring="anomaly"` for fraud investigation
- `extract_chains` `time_window_hours` defaults broadly — narrow it (e.g. 24-72h) to focus on rapid layering patterns
- Typologies are agent-level rules on GDS primitives, not core engine logic
- Single-source flags have high FP rate — multi-source confirmation (2+ patterns) is stronger

## Examples + Troubleshooting

Worked end-to-end examples (full AML screening / typology detection / false-positive elimination) and the canonical troubleshooting tree (`passive_scan` not available / `extract_chains` empty-or-times-out / source_count=1 collapse / `find_counterparties` hub explosion) live in [`references/examples-and-troubleshooting.md`](references/examples-and-troubleshooting.md). Read on demand when an investigation hits one of those modes — the workflow above already covers the happy path.

## Advanced operational workflows

Beyond the per-suspect Phase 2 playbook above, four canonical scenarios get full per-step recipes in [`references/advanced-workflows.md`](references/advanced-workflows.md). One-line entry points:

- **Detect coordinated population-shift attack** — group-level `find_group_influence` with `reinforcing_factor > 1.5` test; catches collusion rings / mule networks / duplicate-record injections that single-account anomaly detection misses by design.
- **AML hidden-influencer triage → SAR candidate workflow** — `find_calibration_influencers(classify="hidden")` + AML adversarial-typology atom matching + `delta_norm ≥ 0.7 × theta_norm` near-threshold filter + group-coordination cross-check.
- **Branch-isolated exclusion calibration** — when hidden influencers are identified, run `recalibrate(pattern_id, branch="excl_branch", exclude_keys=[...hidden_keys...])` to fit without them while still scoring every entity. Then call `compare_branches(pattern_id, branch="excl_branch")` to see how many entities flip anomaly status (`flipped_in` / `flipped_out`; `flipped_in_top` / `flipped_out_top` name them) and whether the Jaccard overlap justifies the change. If the result is the intended calibration, `manage_branch(action="promote", name="excl_branch")` replays the recipes on main and removes the branch. If not, `manage_branch(action="delete", name="excl_branch")` discards it without touching main.
- **Cross-pattern lead-lag for AML rapid-escalation** — `find_lead_lag` between two anchor patterns over the SAME entity line (structural prerequisite); detects whether behaviour shifts precede stress signatures. A typical AML config has one anchor per entity space, so this needs a sphere rebuild before it's usable.
- **Edge-derived dimensions for AML detection** — five build-time per-edge dim functions (`pair_edge_count`, `position_in_chain`, `time_since_pair_last_edge`, `pair_amount_zscore`, `find_motif_structuring`) declared via `edge_dimensions:` on event patterns; lift per-edge geometry signal for layering / structuring detection.

## Aggregated edge dims (account-level + chain-level recall)

The `edge_dim_aggregations:` YAML block on an anchor pattern (or inside `chain_lines:` for chain-anchor patterns) bakes per-edge sidecar dim aggregates (`_mean`, `_max`, `_std`, `_p95`, `_count_above_threshold`) into the polygon. Lifts recall on workflows where per-tx geometry signal is faint but accumulates at the entity / chain level. Full schema, per-aggregate semantics, and how to read them on a suspect in [`references/edge-aggregations.md`](references/edge-aggregations.md). Anchor regimes covered: `single`, `pair`, `chain` (BFS-extracted OR external-table-ingested), k>2 composite.

## Feature curation via stratified correlation gates

Before relying on a feature in a SAR rationale, sanity-check it carries signal independent of the obvious confounders (chain length, account transaction volume). Three-lens approach: all-population gate, stratified per-bucket gate (ROBUST / DIRECTION-INCONSISTENT / LENGTH-MEDIATED / VOLUME-MEDIATED / NOISE verdicts), partial point-biserial correlation. Wire verdicts into `find_anomalies(dimension_weights={...})` runtime ranking: NOISE → `0.0`, DIRECTION-INCONSISTENT / VOLUME-MEDIATED → `0.5`, ROBUST → leave at `1.0`. Full machinery + verdict interpretation + heterogeneous-label hypothesis in [`references/correlation-gates.md`](references/correlation-gates.md).

## Declarative motif cheatsheets

`find_motif_by_hops` opens up custom motif queries beyond the closed vocab (`find_motif_structuring`, `cycle_2`, `fan_out`, etc.) — declarative structuring chains with `amount_ratio_to_prev`, long-chain layering with global `time_window_hours`, and per-hop anomalous-intermediary filtering with `require_anomalous_entity`. Full per-pattern recipes + YAML/Python templates + rules in [`references/declarative-motifs.md`](references/declarative-motifs.md).
