---
name: gds-fraud-investigator
description: AML/fraud detection and financial anomaly investigation via GDS geometric analysis — 3-phase process with 25 typology recipes. Use when user asks to "detect fraud", "screen for money laundering", "find suspicious accounts", "run AML scan", "investigate suspicious transactions", "typology detection", "passive scan for fraud", "detect insider trading", "find anomalous billing", "investigate suspicious transactions", or "financial crime detection". Use this skill for ANY financial anomaly investigation, not just AML. Requires hypertopos MCP server with a financial transaction sphere (account + pair + chain patterns).
license: Apache-2.0
compatibility: Requires hypertopos MCP server with a financial transaction sphere (account, pair, chain patterns).
metadata:
  author: Karol Kędzia
  version: 0.8.0
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

## Prerequisites — Edge Table Check

Before using graph traversal tools (`find_geometric_path`, `discover_chains`), verify the
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

When using `find_anomalies` for fraud screening, set `fdr_alpha=0.05` to apply Benjamini-Hochberg FDR control — false positives waste investigator time and erode trust in the alert pipeline, so controlling the false discovery rate is critical. When requesting K>10 results, use `select="diverse"` to surface different fraud typologies (structuring, layering, round-tripping) instead of 50 variants of the same high-volume pattern; this leverages submodular facility location to maximize typological coverage in the result set. Both parameters also apply to `attract_boundary`, `find_hubs`, and `find_drifting_entities`.

**Adaptive Storey FDR.** On transaction event patterns whose delta distribution has a real null mass (normal accounts far below the alert threshold) plus an anomalous tail (genuine high-risk accounts), `fdr_method="storey"` combined with `p_value_method="chi2"` relaxes the BH threshold to recover 10–15% more candidates at the same α. Use this when the default BH top-K is systematically leaving edge cases that investigators later flag as true positives. Both parameters must be set together — Storey with rank p-values is a no-op. Keep the default `fdr_method="bh"` on account_stress-style patterns where every entity carries some stress signal, because there Storey cannot distinguish null mass from tail.

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
- `find_topological_anomalies` ranks by local H_1 cycle persistence (raw `h1_max_persistence`, NOT the normalised `topo_score`). Empirical AUROC `+0.12–0.16` over `delta_norm` baseline on labelled fraud rings; lift is mid-rank rather than top-N — use as composition input for HMP / passive_scan, not as a top-N drill-down replacement

**One-call root-cause tracing.** Instead of running steps 1–8 above manually, `trace_root_cause(suspect_pk, "<anchor_pattern>")` returns a bounded DAG of evidence in one shot — root witness dimensions, edge-counterparty branch (sorted by anomaly, not transaction volume — catches structural cps that high-volume sort would miss), neighbour-contamination branch with `anomalous_cp_keys` + `revisits_root` clique flag, hub branch when population ≤ `hub_pop_limit`. Use it as the default first pass during Phase 2 confirmation; drop into the manual chain only when `truncated=true` signals the tree missed context you need or when you explicitly want per-step control.

**One-call full entity 360.** `investigate_entity(suspect_pk, "<anchor_pattern>", line_id="<event_pattern>")` chains polygon shape + explain_anomaly + witness cohort + chains + root_cause + graph_geometry_tension into one MCP call with per-step `steps_status` (partial failures surface without aborting). Entity-side analog of `investigate_chain` (0.6.7). Use as default Phase 2 entry point when you want one structured report rather than chaining 6 manual calls. Pass `chain_pattern_id="<chain_pattern>"` to populate chains; `include_per_edge_counterfactual=true` wires through to `simulate_edge_removal` (v0 covers relations + prop_columns dim classes; AML `account_pattern` returns empty under v0 because its edge-derived signal sits in `edge_dim_aggregations` — full coverage in a follow-up patch).

**Counterfactual suite (4 tools).** Different questions, different aggregation level.

- `simulate_edge_removal(suspect_pk, "<anchor_pattern>", line_id="<event_pattern>", top_n=5)` — "which transactions made this entity anomalous?". Ranks the entity's edges by contribution to `delta_norm`. For each edge: `(delta_norm_before, delta_norm_after, drop_pct, dominant_dim_label)`. v0 covers relations + prop_columns dim classes; on AML `account_pattern` returns empty under v0 because edge signal sits in `edge_dim_aggregations` held constant in v0.
- `simulate_counterparty_removal(suspect_pk, "<anchor_pattern>", line_id="<event_pattern>", top_n=5)` — "which counterparty made this entity anomalous?". Aggregates all edges to one counterparty per call; ranks counterparties by aggregate `delta_norm` drop. Use when the edge-level ranking is too granular and you want a counterparty-level SAR narrative ("removing transactions with ACC-X drops delta_norm by 38 %").
- `select_minimal_joint_edge_removal(suspect_pk, "<anchor_pattern>", line_id="<event_pattern>", target="flip", k=8)` — "smallest set of edges whose joint removal flips the anomaly verdict". Greedy edge-set search; returns the minimal `edge_ids[]` + `delta_norm_after` + `flipped` flag. SAR composition: the smallest joint set is the literal "evidence of laundering" that explains why this entity crosses θ.
- `simulate_dimension_change(suspect_pk, "<anchor_pattern>", line_id="<event_pattern>", set_dimension={dim_label: new_value}, top_n=5)` — "would this entity still be anomalous if dim X were value V?". Override one or more raw shape-vector dims, recompute `delta_norm` under the pattern's calibration; reports `delta_norm_before/after`, the anomaly-flag flip, and new top witness dims. Companion to `simulate_edge_removal` for non-edge dimensions. Call `explain_anomaly` first to pick dim_labels and read current raw values.

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

If the entity is not in the scored sample, the response carries `found=false`
with a note — raise `sample_size` (or pass `None` for the full population) and
re-run. Gate rule of thumb: escalate to SAR only when
`assess_anomaly_certainty` is `high`/`moderate` AND `consensus_classification`
is `anomalous_consensus` (or a `moderate`/`high` certainty corroborated by a
second pattern). A `contested` certainty or a `single_detector_signal`
consensus is a hold-and-corroborate signal, not an escalation.

**Fraud-specific tuning.**
- **`edge_counterparty_top_n=3..5`** — mule networks typically involve multiple anomalous counterparties; default 1 shows only the single most anomalous cp. For Phase 3 escalation on high-risk suspects, raise to expand each anomalous cp as a separate subtree.
- **`branches_enabled=["neighbor_contamination"]`** — fast targeted scan: for a batch of candidate suspects, contagion-only trace avoids the edge_counterparty recursion and π7 scan; orders of magnitude cheaper than default once cache is warm. Use to rank-by-contagion before committing to full traces on the top-N.
- **`contagion_min_counterparties=5`** — raise default 3 for high-confidence fraud networks; filters out statistical noise from small-counterparty suspects.

**Clique detection signals (read these on every trace).**
- **`revisits_root: [keys]`** on a contagion branch = confirmed 2+ node clique with the root. Every name in this list transacted with the root AND is itself anomalous. Immediate Phase 3 escalation criterion.
- **`previously_seen_as_cp_of: [X, Y]`** = session ledger signal: other suspects you already traced (X, Y) listed THIS entity as their anomalous cp. The entity is a shared counterparty across multiple confirmed suspects — high-value graph-centre node worth prioritising.
- **`anomalous_cp_keys`** list → all 10+ anomalous neighbours available without a second `find_counterparties` call. Use directly as Phase 3 investigation queue.

High contagion (>0.3) + large witness cohort + high novelty = confirmed network pattern. Low contagion (<0.2) + empty cohort = isolated anomaly, deprioritize. Borderline contagion (0.2–0.3) = expand cautiously to one hop only before deciding.

**Edge-level rare-pair detection — `find_high_potential_edges`.** AML layering often shows as a one-off transaction between two geometrically divergent accounts (high-income sender, dormant/low-history receiver). Node-level delta_norm misses it because each endpoint alone looks only borderline. `find_high_potential_edges(pattern_id, top_n=20, min_pair_count=1)` catches these by scoring the edge itself.

- Use early in Phase 2 — run AFTER `find_anomalies` on the anchor pattern but BEFORE committing a full `trace_root_cause` on each candidate. High-edge-potential edges point directly at suspect pairs without requiring a candidate selection.
- Cross-reference with `trace_root_cause.edge_counterparty.edge_potential` — if the same edge appears in both (top of find_high_potential AND enriched evidence on trace), that's a confirmed rare-pair layering signal worth immediate Phase 3 escalation.
- Raise `min_pair_count=3` on spheres where legitimate frequent-but-large pairs (corporate payroll) dominate the top — shifts focus to truly one-off suspicious edges.

**Dual-query playbook — singletons AND recurring suspects.** The default ranking with `min_pair_count=1` is dominated by one-off singleton pairs (classic placement/layering signature — AUROC 0.92 on AML HI-small). But recurring suspect pairs (a handful of transactions between two anomalous accounts) are a DIFFERENT fraud shape — e.g. structuring below reporting thresholds over several days. A single ranking call catches one shape; run BOTH:

```
# Pass 1: singletons (placement / one-off layering)
find_high_potential_edges(pattern_id, top_n=20, min_pair_count=1)

# Pass 2: recurring suspects (structuring / smurfing)
find_high_potential_edges(pattern_id, top_n=10, min_pair_count=5)
```

The second pass returns pairs with 5+ transactions between the same two anomalous accounts — that pattern rarely appears in legitimate business and is a strong smurfing signal. Use both outputs as two parallel Phase 3 queues. Each `is_high_potential=true` entry is above the pattern's p95 of scores and warrants immediate review.

**Reading the `is_high_potential` flag + `score_rank_pct`.** The scalar `score` is not cross-sphere-comparable (it scales with delta dimensionality), but every result carries `score_rank_pct` (percentile within the pattern) and `is_high_potential` (boolean, true when score ≥ p95). Use these for triage: `is_high_potential=true` is the actionable flag; `score_rank_pct >= 99` marks the extreme tail for immediate escalation.

**Structural motif detection — `score_motif` and `find_high_potential_motifs`.** Where `edge_potential` scores one anomalous edge, `score_motif` scores the whole structural shape of a suspicious k-edge motif. The scoring rule is product-of-edge_potential across motif edges — a motif survives only if ALL its edges are rare and its endpoints are geometrically distant. One regular high-volume edge in a triad collapses the score to near-zero (correct: a triad with a payroll edge is not laundering).

Eight closed motif types (`cycle_2`, `cycle_3`, `fan_out`, `fan_in`, `chain_k`, `structuring`, `split_recombine`, `bipartite_burst`) map to the 25 typologies in [`references/typologies.md`](references/typologies.md). On AML-shaped spheres start with `structuring` (highest empirical recall on IBM AML — closed-triad `cycle_3` is effectively absent there). `cycle_2` for flash-burst priors. `fan_out` for concentrator hubs.

Full motif vocabulary specs (defaults, temporal-ordering rules, structural-atom mappings to typologies T1–T19) + AML Phase 2 workflow + jurisdictional structuring thresholds (US CTR / UK STR / EU CTR / crypto) + cold-call cost / warm-up order / scale threshold (>10M edges → seed-first pattern with `score_motif`) + `motif_potential` enrichment on `trace_root_cause` are documented in [`references/motif-detection.md`](references/motif-detection.md).

Key signals:
- `anomaly_dimensions` + `bregman_contribution` shows WHICH behavioral features drive the detection
  - `kind: poisson` dim anomalous = count/frequency structure deviation (structuring, burst)
  - `kind: bernoulli` dim anomalous = binary flag triggered (cross-border, FX flag, unusual channel)
  - `kind: gaussian` dim extreme = magnitude anomaly (large amount, high velocity)
  - Focus on dims with highest `pct_of_total` in the Bregman breakdown, not just highest abs_delta
- Counterparties with `is_anomaly=true` = network confirmation
- Temporal burst → silence pattern = classic placement/layering

### Recipe: Network Expansion

From a confirmed suspect, expand the investigation network:

```
1. find_counterparties(suspect)           → list of transaction partners
2. Filter anomalous counterparties        → subset with is_anomaly=true
3. For each anomalous counterparty:
     cross_pattern_profile(cp_key)        → are THEY multi-source flagged?
4. discover_chains(suspect, pattern_id="<event_pattern>",
     time_window_hours=72, max_hops=5)    → runtime chain discovery (preferred)
5. Filter: chains with cyclic structure   → round-trip chains
```

**`discover_chains` vs `extract_chains`:** `discover_chains` runs temporal BFS at query
time on the edge table — no pre-built chain_lines required. Use it as the primary chain
discovery tool. Fall back to `extract_chains(seed_nodes=[suspect])` when you need
population-level chain statistics or when the sphere has pre-built chain patterns.

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
- `find_anomalies(metric="bregman")` — rank by Bregman divergence. Better than L2 on patterns mixing counts (poisson), amounts (gaussian), and flags (bernoulli). Check `dimension_kinds` in sphere_overview — if mixed kinds, try bregman first.

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

## Per-Pattern Investigation Guide

| Pattern type | Key signal | Investigation tool |
|---|---|---|
| FAN-OUT | high out_degree + many dest_banks | find_counterparties → check all targets; `edge_stats` confirms degree distribution |
| FAN-IN | high in_sources | find_counterparties → who feeds this account?; `edge_stats` for inbound concentration |
| CYCLE | pair anomalies + chain is_cyclic | `discover_chains(direction="outgoing")` → filter cyclic; `find_geometric_path(scoring="anomaly")` traces the ring |
| STACK | delta_rank_pct 90-95 (borderline) | `discover_chains(min_hops=3)` → check chain membership at runtime |
| BIPARTITE | pair anomalies + community | aggregate(group_by_property="community_id") |
| RANDOM | chain anomalies | `discover_chains(max_hops=6)` → longest reachable chains |
| BRIDGE | entity straddles two communities | `cluster_bridges(pattern_id)` → bridges with anomalous status on both sides are high-risk connectors |

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

### Recipe input + capability matrix

Quick-reference for "what does this recipe operate on?" and "how does it compose with the chain investigative loop?". Use to triage which recipes apply when only certain primitives are available, OR when picking which recipe a candidate entity belongs to.

| Recipe | Input | Edge table use | One-shot orchestrator | SAR narrative composer |
|---|---|---|---|---|
| R1 Mirror Transaction | account pairs (edge scan) | primary — direct pair scan | none | none |
| R2 Pass-Through | account | derived — `intermediary_score` baked at build time | none | none |
| R3 Burst (Structuring) | account + motif | primary — runtime motif enumeration (`find_motif_structuring`, `find_motif_by_hops`) | none | none |
| R4 Weighted Reciprocity | account pairs | primary — cycle-2 motif | none | none |
| R5 Mule Profile | account | none — `find_anomalies` on anchor delta | none | none |
| R6 Concentration Risk | account + graph features | derived — degree / betweenness / pagerank baked at build time | none | none |
| R7 Benford | account amount distribution | none — distribution check | none | none |
| R8 Witness Cohort | account + delta-space neighbours | secondary — BTREE edge index for already-connected exclusion | none | none |
| **R9 Chain-Coherent Cascade** | **chain_pattern + anchor_pattern** | none — operates on stored chain pattern (chain extraction used edges at build time) | **`investigate_chain`** | **`generate_sar_rationale`** |
| R10 TBM | account + motif | primary — `find_high_potential_motifs(motif_type='fan_out')` runtime enumeration | none | none |
| R11 Mule Network | account clusters | secondary — clustering edge-free, scoring uses `contagion_score` + `cross_pattern_profile` | none | none |

**Three patterns this matrix surfaces:**

- **External-chain workflow applies only to R9** — R9 is the only recipe whose input is a chain. R1-R8 / R10-R11 take accounts / pairs / motifs / clusters — chain identifiers don't enter their signature. To unlock external-chain ingestion for them, would need composite primitives (e.g. "R3 burst scoped to chain members", "R8 witness cohort seeded from chain hops") — out of scope today, see [external-chains-as-anchor-line.md](../../hypertopos-py/docs/external-chains-as-anchor-line.md) for the R9 path.
- **Edge-free recipes are R5 / R7 / R9** — R5 reads only anchor delta features, R7 reads only the amount distribution column, R9 reads chain anchor pattern + member anchor pattern. The other 8 either scan the edge table runtime (R1, R3, R4, R10) or read derived edge-features baked into the anchor at build time (R2, R6, R8, R11).
- **Only R9 has end-to-end orchestration + SAR narrative today** — `investigate_chain` aggregates the four R9 primitives into one server-side call and `generate_sar_rationale` composes the resulting evidence into a paragraph-structured draft. Other recipes are manual call sequences; per-recipe orchestrators (`investigate_account`, `investigate_burst`, `investigate_network`, etc.) and per-recipe SAR templates would be the natural extensions.

### Per-recipe detail

Tool sequences, interpretation rules, false-positive guards, and uniqueness rationale for every recipe live in [`references/recipes.md`](references/recipes.md). Read on demand when actually executing a recipe — the summary table above and the matrix below are the index.

The summary placeholder for each recipe (one-line "Pattern" sentence) is below; the full per-recipe block was extracted to keep this file scannable.

### R1 — Mirror Transaction Detection

**Pattern:** Entity A sends to B, and B sends back to A the same (or similar) amount within the same day. Classic circular flow indicator. Full detail in [`references/recipes.md#r1`](references/recipes.md).

### R2 — Pass-Through / Rapid Layering

**Pattern:** Entity receives funds and sends within 2 hours. The entity is a conduit, not a destination. Full detail in [`references/recipes.md#r2`](references/recipes.md).

### R3 — Burst Detection (Structuring / Smurfing)

**Pattern:** Many transactions to the same target within 24 hours, each below a reporting threshold. Full detail in [`references/recipes.md#r3`](references/recipes.md).

### R4 — Weighted Reciprocity

**Pattern:** Balanced bidirectional flow between two entities — min(out, in) / max(out, in) close to 1.0. Full detail in [`references/recipes.md#r4`](references/recipes.md).

### R5 — Financial Profile (Mule Detection)

**Pattern:** Entity's total flow reveals its role: source (high out, low in), sink (high in, low out), or mule (high both, near-zero net). Full detail in [`references/recipes.md#r5`](references/recipes.md).

### R6 — Concentration Risk

**Pattern:** A single counterparty dominates an entity's flow — potential control relationship. Full detail in [`references/recipes.md#r6`](references/recipes.md).

### R7 — Benford's Law (Amount Distribution)

**Pattern:** Natural financial data follows Benford's Law for first digits. Fabricated transactions often don't. Full detail in [`references/recipes.md#r7`](references/recipes.md).

### R8 — Witness Cohort Discovery (Fraud Cohort Expansion)

**Pattern:** A confirmed launderer X has a ring of accomplices. Some are already in X's counterparty network. Others share X's anomaly signature — same witness dimensions, drifting in the same geometric direction — but are NOT yet connected to X via transactions. `find_witness_cohort` surfaces these geometric peers, with already-connected entities filtered out. **Honest scope:** investigative cohort expansion, NOT edge forecasting. Full detail in [`references/recipes.md#r8`](references/recipes.md).

### R9 — Chain-Coherent Cascade (Composition-based Layering)

**Pattern:** A stored chain (anchor pattern built from `chain_lines:` BFS extraction OR ingested from an external system per the [external-chains convention](../../hypertopos-py/docs/external-chains-as-anchor-line.md)) hops through ≥ N consecutive entity-anchor positions that are individually anomalous in the entity-anchor pattern AND share the same dominant delta dimension. The chain-level shape (hop count, time span, amount decay) may look unremarkable, but the *composition* of the path — every consecutive entity flagged for the same structural reason — is the signal.

**External chain source.** When chains come from upstream (SAR typology engine, ERP workflow audit, EHR pathway, customer-journey platform), declare the chain table as an anchor line and populate the `chain_keys` column per the convention (comma-joined member primary_keys in chain order). The full R9 loop — `find_chains_with_coherent_anomaly`, `anomaly_propagation_in_chain`, `classify_chain_typology`, `extend_chain`, `chain_investigation_summary`, `investigate_chain` — works identically on those chains. No code change, just the schema convention. See [external-chains-as-anchor-line.md](../../hypertopos-py/docs/external-chains-as-anchor-line.md) for the worked example.

**Tool sequence (full investigative loop — triage → flag → trace → label → extend, with one-shot orchestrator shortcut):**
0. **Triage (optional, recommended on unfamiliar spheres)** — `chain_investigation_summary(chain_pattern_id="<chain_pattern>", anchor_pattern_id="<entity_anchor>")` returns one-shot population aggregates: `coherent_run_rate`, `cross_pattern_overlap.jaccard` with chain-shape anomalies, `top_dims_in_coherent_runs`, `recommended_min_hops`, `run_length_distribution`. Triage rules: `coherent_run_rate < 0.005` AND low jaccard → skip the deep R9 loop, fall back to `find_anomalies(<chain_pattern>)`; `coherent_run_rate > 0.05` → expect a productive loop, proceed; `recommended_min_hops > 3` → use the recommended threshold in step 1 to focus on the strongest cases. Cost is one coherent-anomaly sweep — exactly what step 1 would pay anyway, with the aggregates surfaced for free.

**Per-chain shortcut (after step 1 surfaces a target chain_id):** `investigate_chain(chain_id, "<chain_pattern>", anchor_pattern_id="<entity_anchor>")` runs steps 2-5 (trace + typology + shape-anomaly lookup + extension forward + extension backward) server-side in a single call and returns the aggregated report with a `summary` block. Strength scoring uses **four chain-composition signals** (coherent run length >= 3, typology position not "no-run", forward extension has an anomalous candidate, backward extension has an anomalous candidate) — chain-shape anomaly is reported as evidence but NOT scored, so the R9 sweet spot (composition anomalous, chain shape normal) reaches `strong` without needing chain-shape agreement. Buckets: `score >= 3` → `strong` → `escalate to SAR`; `score == 2` → `moderate` → `continue investigation`; `score 0-1` → `weak` → `false-positive candidate`. Rationale = concatenated SAR-ready paragraph. Use when the investigator already knows which chain to drill into and wants the full R9 read in one round-trip; the per-step granular tools (steps 2-5 below) remain available when finer per-step control is needed.

**SAR narrative draft (final step in the per-chain shortcut path):** `generate_sar_rationale(chain_id, "<chain_pattern>", anchor_pattern_id="<entity_anchor>", evidence=<investigate_chain output>)` composes a 3-5 paragraph SAR-ready narrative from the R9 evidence — chain identification + typology, per-hop trace, boundary extensions, chain-shape corroboration, aggregated strength + recommended action. No LLM call; pure template composition. Returns `sar_narrative` (paragraph-separated string), `evidence_anchors` (structured pointers per claim — investigator can audit every line against source data), `confidence` (`high` / `moderate` / `low` derived from strength + evidence completeness), and a `regulatory_template_hint` passthrough (`"FinCEN SAR"` default; tag with `"EU AMLR Annex II"` or internal template name for downstream filing systems). Pass the `investigate_chain` return verbatim as `evidence` to skip re-running the R9 loop. Honesty discipline: narrative uses "evidence indicates" / "the per-hop trace shows" / "corroborating evidence" — never "confirms" — and is positioned as a starting draft for the investigator, NOT a final verdict.
1. **Flag** — `find_chains_with_coherent_anomaly(pattern_id="<chain_pattern>", anchor_pattern_id="<entity_anchor>", min_hops=3, max_results=100)` sweeps all chains in the chain pattern, returns ranked runs (chain_id, run_start_idx, run_length, top_dim, run_keys, max_delta_norm).
2. **Trace** — for each top flagged chain: `anomaly_propagation_in_chain(chain_id, "<chain_pattern>", anchor_pattern_id="<entity_anchor>")` returns the full per-hop progression — see WHERE the anomaly intensity peaks and WHERE it breaks. Hops carry `is_anomaly`, `delta_norm`, `top_dim`, `delta_rank_pct`. Run end with low `delta_rank_pct` = clean exit; with elevated rank = soft boundary worth extending.
3. **Label** — `classify_chain_typology(chain_id, ...)` wraps the trace and returns a five-axis operational tag: `shape` (rising / falling / peak-position), `position_in_chain` (leading / transit / terminal / full-chain), `extension_signals` (forward / backward booleans), plus `dominant_top_dim` across the whole chain. Lets investigator triage chains by typology without re-reading hop sequences.
3b. **Witness coordination** — `chain_witness_intersection(chain_id, "<chain_pattern>", member_pattern="<entity_anchor>")` intersects the top witness dimensions of the chain's members. `coordinated=True` (mean pairwise Jaccard `>= min_jaccard`, default 0.5) means every member is anomalous for the same structural reason — a single geometric diagnosis for the chain instead of N independent member-level explanations. Use as the witness-side refinement of the typology label: same `shape` + `coordinated=True` reads stronger as a typology candidate than the same `shape` + `coordinated=False`.
3c. **Drift trajectory** — `chain_drift_trajectory(chain_id, "<chain_pattern>", member_pattern="<entity_anchor>", n_windows=4)` returns the per-member regime over time-bucketed `delta_norm` plus a chain-level rollup (`normalizing` / `deteriorating` / `neutral` / `mixed`) and a numeric drift score. Use to spot chains whose members are *jointly* drifting toward anomaly before any single hop crosses the threshold — the rolling-onset surface complementary to the static trace in step 2.
4. **Cross-check** — compare with `find_anomalies(<chain_pattern>)`: chains in BOTH sets are highest-confidence (chain shape AND chain composition agree); chains in coherent-anomaly-only are missed by chain-shape scoring; chains in baseline-only are flagged for shape (e.g. amount decay) but not composition. The two are orthogonal detectors.
5. **Extend** — when `extension_signals.forward` or `.backward` is True (or the trace's breakpoint hop is in `delta_rank_pct >= 80` band): `extend_chain(chain_id, ..., direction="forward"|"backward")` returns ranked candidate entities that follow / precede the boundary in OTHER chains. Anomalous candidates with high `delta_norm` are prime targets for widening the investigation into the surrounding ring.
6. **Deep-dive per candidate** — for each high-rank extension target: `find_chains_for_entity(candidate_key, <chain_pattern>)` to enumerate which chains contain it, then standard Phase 2 (Entity 360) on those chains' anchor entities.

**Interpretation:** A run of length 4 with `top_dim="find_motif_structuring_max"` is textbook structuring — every account on the path is individually flagged for high motif-structuring activity. Mixed top_dim across the population (3-5 distinct drivers in the top 100 results) signals diverse layering vectors, not a single typology.

**Agent-friendly natural-language entry points** (via `detect_pattern` smart-mode router): "find chains where consecutive accounts are individually anomalous" routes to step 1 Flag; `trace chain CHAIN-XXX hop by hop` → step 2; `classify chain CHAIN-XXX typology` → step 3; `extend chain CHAIN-XXX forward` → step 5. Step 6 deep-dive (`find_chains_for_entity`) requires direct call (entity keys vary per sphere, not extractable from query). Full per-step language table + keyword-fallback notes + complementary-not-replacement detail in [`references/recipes.md#r9`](references/recipes.md).

### R10 — Trade-Based Money Laundering (TBM)

**Pattern:** Multi-currency cross-bank flows with amount mismatches typical for over/under-invoicing. Discriminator: `currency_diversity >= 2` AND `cross_bank_count >= 2` along a fan-out path. **Status:** Exploratory — empirical lift TBD. Full detail in [`references/recipes.md#r10`](references/recipes.md).

### R11 — Mule Network Discovery

**Pattern:** A *group* of low-volume accounts forming a dense subgraph with coordinated reciprocal flows. Detected as a cluster (not from a confirmed seed — that's R8). **Status:** Exploratory — empirical lift TBD. Full detail in [`references/recipes.md#r11`](references/recipes.md).

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
- `extract_chains` without `seed_nodes` causes hub monopolization — prefer `discover_chains` which takes a single primary_key
- `find_geometric_path` with `scoring="shortest"` finds direct connections but misses suspicious intermediaries — use `scoring="anomaly"` for fraud investigation
- `discover_chains` `time_window_hours` defaults broadly — narrow it (e.g. 24-72h) to focus on rapid layering patterns
- Typologies are agent-level rules on GDS primitives, not core engine logic
- Single-source flags have high FP rate — multi-source confirmation (2+ patterns) is stronger

## Examples + Troubleshooting

Worked end-to-end examples (full AML screening / typology detection / false-positive elimination) and the canonical troubleshooting tree (`passive_scan` not available / `extract_chains` empty-or-times-out / source_count=1 collapse / `find_counterparties` hub explosion) live in [`references/examples-and-troubleshooting.md`](references/examples-and-troubleshooting.md). Read on demand when an investigation hits one of those modes — the workflow above already covers the happy path.

## Advanced operational workflows

Beyond the per-suspect Phase 2 playbook above, four canonical scenarios get full per-step recipes in [`references/advanced-workflows.md`](references/advanced-workflows.md). One-line entry points:

- **Detect coordinated population-shift attack** — group-level `find_group_influence` with `reinforcing_factor > 1.5` test; catches collusion rings / mule networks / duplicate-record injections that single-account anomaly detection misses by design.
- **AML hidden-influencer triage → SAR candidate workflow** — `find_calibration_influencers(classify="hidden")` + AML adversarial-typology atom matching + `delta_norm ≥ 0.7 × theta_norm` near-threshold filter + group-coordination cross-check.
- **Cross-pattern lead-lag for AML rapid-escalation** — `find_lead_lag` between two anchor patterns over the SAME entity line (structural prerequisite); detects whether behaviour shifts precede stress signatures. Today's typical AML config has one anchor per entity space, so this needs a sphere rebuild before it's usable.
- **Edge-derived dimensions for AML detection** — five build-time per-edge dim functions (`pair_edge_count`, `position_in_chain`, `time_since_pair_last_edge`, `pair_amount_zscore`, `find_motif_structuring`) declared via `edge_dimensions:` on event patterns; lift per-edge geometry signal for layering / structuring detection.

<!-- Detail moved to references/advanced-workflows.md to keep SKILL.md scannable. Each scenario keeps full per-step Python templates, validation rules, and "when not to use" guidance in the reference. -->

## Aggregated edge dims (account-level + chain-level recall)

The `edge_dim_aggregations:` YAML block on an anchor pattern (or inside `chain_lines:` for chain-anchor patterns) bakes per-edge sidecar dim aggregates (`_mean`, `_max`, `_std`, `_p95`, `_count_above_threshold`) into the polygon. Lifts recall on workflows where per-tx geometry signal is faint but accumulates at the entity / chain level. Full schema, per-aggregate semantics, and how to read them on a suspect in [`references/edge-aggregations.md`](references/edge-aggregations.md). Anchor regimes covered: `single`, `pair`, `chain` (BFS-extracted OR external-table-ingested), k>2 composite.

## Feature curation via stratified correlation gates

Before relying on a feature in a SAR rationale, sanity-check it carries signal independent of the obvious confounders (chain length, account transaction volume). Three-lens approach: all-population gate, stratified per-bucket gate (ROBUST / DIRECTION-INCONSISTENT / LENGTH-MEDIATED / VOLUME-MEDIATED / NOISE verdicts), partial point-biserial correlation. Wire verdicts into `find_anomalies(dimension_weights={...})` runtime ranking: NOISE → `0.0`, DIRECTION-INCONSISTENT / VOLUME-MEDIATED → `0.5`, ROBUST → leave at `1.0`. Full machinery + verdict interpretation + heterogeneous-label hypothesis in [`references/correlation-gates.md`](references/correlation-gates.md).

## Declarative motif cheatsheets

`find_motif_by_hops` opens up custom motif queries beyond the closed vocab (`find_motif_structuring`, `cycle_2`, `fan_out`, etc.) — declarative structuring chains with `amount_ratio_to_prev`, long-chain layering with global `time_window_hours`, and per-hop anomalous-intermediary filtering with `require_anomalous_entity`. Full per-pattern recipes + YAML/Python templates + rules in [`references/declarative-motifs.md`](references/declarative-motifs.md).
