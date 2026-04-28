---
name: gds-fraud-investigator
description: AML/fraud detection and financial anomaly investigation via GDS geometric analysis — 3-phase process with 25 typology recipes. Use when user asks to "detect fraud", "screen for money laundering", "find suspicious accounts", "run AML scan", "investigate suspicious transactions", "typology detection", "passive scan for fraud", "detect insider trading", "find anomalous billing", "investigate suspicious transactions", or "financial crime detection". Use this skill for ANY financial anomaly investigation, not just AML. Requires hypertopos MCP server with a financial transaction sphere (account + pair + chain patterns).
license: Apache-2.0
compatibility: Requires hypertopos MCP server with a financial transaction sphere (account, pair, chain patterns).
metadata:
  author: Karol Kędzia
  version: 0.5.2
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
7. dive_solid(pk, "<anchor_pattern>")     → WHEN behavior changed
8. investigation_coverage(pk, "<event_pattern>",
     explored_keys=checked)               → coverage check, add unexplored to leads
```

**Graph confirmation chain (steps 4→5→6):**
- `contagion_score > 0.3` → neighborhood is infected, not an isolated outlier
- `witness_cohort_size > 3` → anomaly signature is shared by non-connected peers
- `find_novel_entities` surfaces entities whose geometry deviates from what their neighbors predict — catches entities that contagion and witness miss

**One-call root-cause tracing.** Instead of running steps 1–8 above manually, `trace_root_cause(suspect_pk, "<anchor_pattern>")` returns a bounded DAG of evidence in one shot — root witness dimensions, edge-counterparty branch (sorted by anomaly, not transaction volume — catches structural cps that high-volume sort would miss), neighbour-contamination branch with `anomalous_cp_keys` + `revisits_root` clique flag, hub branch when population ≤ `hub_pop_limit`. Use it as the default first pass during Phase 2 confirmation; drop into the manual chain only when `truncated=true` signals the tree missed context you need or when you explicitly want per-step control.

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

**Structural motif detection — `score_motif` and `find_high_potential_motifs`.** Where `edge_potential` scores one anomalous edge, `score_motif` scores the whole structural shape of a suspicious k-edge motif. The scoring rule is product-of-edge_potential across motif edges — a motif survives the score only if ALL its edges are rare and its endpoints are geometrically distant. One regular high-volume edge in a triad collapses the score to near-zero (correct: a triad with a payroll edge is not laundering).

Closed vocabulary mapped to the 25 documented typologies in `references/typologies.md`:

- **`cycle_2`** (default 24h window) — A↔B bidirectional round-trip. Structural atom of T2 Flash-Burst Round-Trip and T4 Bidirectional Burst.
- **`cycle_3`** (default 72h window, strict temporal ordering `ts_ab < ts_bc < ts_ca`) — directed triad A→B→C→A. Structural atom of T3 Round-Tripping 3-Party, T5 Long-Cycle, T11 Multi-Round-Tripping, T19 Multi-Direction Feedback Loop. **Note:** closed-triad fraud cycles in IBM AML are effectively absent (autoresearch caught 0/0 TPs on HI+LI-small). Real AML cycles are 4+ hops or open chains — use `structuring` below for the canonical AML atom.
- **`fan_out`** (default 168h window, min k=3 targets) — hub → k distinct targets in the window. Structural atom of T6 Offshore Hub, T13 Concentrator / Sink (source side).
- **`fan_in`** (default 168h window, min k=3 sources) — k distinct sources → sink, mirror of `fan_out`. Structural atom of T12 Parallel Layering (destination side) and T13 Concentrator / Sink.
- **`chain_k`** (default 168h window, open directed chain of length k, 3 ≤ k ≤ 8) — A→B→…→Z with no cycle closure, no node revisit, strict monotone timestamps, total span ≤ window. Amount-free open-chain counterpart to `structuring`. Structural atom of T5 Long-Cycle Multi-Stage Layering and T18 Multi-Jurisdiction Latency Chain; tune `k` to the layering depth under investigation (default 4; `k=3` for fast shallow scans, `k≥6` for targeted deep investigations).
- **`structuring`** (default 1h window, amount-gated) — open linear chain A→B→C→D with hop1 amount ≥ `amt1_min` and hops 2 and 3 amount ≤ `amt2_max`, strict temporal ordering. Classic cash-deposit-split-and-wire pattern for evading reporting thresholds. Defaults `amt1_min=10000, amt2_max=10000` match the USD CTR threshold; override per jurisdiction — GBP CTR is 10000 GBP, EU CTR is 10000 EUR, crypto exchange thresholds vary. Empirically this is the dominant find_motif typology on IBM AML labelled fraud (closed-triad `cycle_3` is effectively absent there — AML fraud cycles are 4+ hops or open chains). Structural atom of structuring / smurfing typology.
- **`split_recombine`** (default 24h window, `min_k` default 3, `direction` parameter `"forward"|"backward"`) — diamond scatter-gather smurfing: source S → k distinct intermediaries {M₁,…,Mₖ} → single sink D, with stacked-bipartite temporal order (all split-hops precede all recombine-hops within the window). Forward mode anchors the seed as the source S; backward mode anchors the seed as the sink D. Amount-free counterpart to `structuring` for the diamond shape. Structural atom of T1 Structured Layering (when split goes through multiple categories), T12 Parallel Layering (forward — multiple chains converge on the sink), and T13 Concentrator / Sink (backward — sink receives from many independent intermediaries).
- **`bipartite_burst`** (default 24h window, `min_k` default 3 sources, `min_m` default 3 sinks) — complete K_{k,m} bipartite subgraph in a tight time window: k distinct sources each transact with every one of m distinct sinks. Greedy single-core enumeration (not maximal): tries seed-as-source first, falls back to seed-as-sink. Complements `fan_out` + `fan_in` by requiring completeness on both sides rather than just density at a single anchor — flags coordinated mule-ring and parallel-collusion shapes that single-anchor fans miss. The K_{k,m} completeness constraint is why this catches collusion: random anomalous traffic rarely produces a fully-saturated bipartite subgraph in a tight window.

**AML workflow (Phase 2 confirmation):**

```
1. find_high_potential_edges(pattern_id, min_pair_count=1)       → single-edge rare-pair candidates
2. For top-K candidates → score_motif(suspect, motif_type="structuring", pattern_id)
3. If structuring is_high_potential → escalate as deposit-split-and-wire (highest-recall AML atom)
4. Otherwise score_motif(suspect, motif_type="cycle_2", pattern_id)
5. If cycle_2 is_high_potential → flash-burst round-trip
6. Otherwise score_motif(suspect, motif_type="cycle_3", pattern_id)   # only if non-IBM-AML domain
7. For hub-candidates (high out_degree) → score_motif(suspect, motif_type="fan_out", pattern_id)
```

**Global motif screening:**

```
find_high_potential_motifs(pattern_id, motif_type="structuring", top_n=20)        # top deposit-split-and-wire (AML default)
find_high_potential_motifs(pattern_id, motif_type="cycle_3", top_n=20)            # top round-tripping 3-party rings
find_high_potential_motifs(pattern_id, motif_type="fan_out", top_n=20)            # top concentrator hubs
find_high_potential_motifs(pattern_id, motif_type="cycle_2", top_n=20)            # top flash-burst round-trips
find_high_potential_motifs(pattern_id, motif_type="split_recombine", top_n=20)    # top scatter-gather diamonds (forward by default)
find_high_potential_motifs(pattern_id, motif_type="bipartite_burst", top_n=20)    # top K_{k,m} coordinated bursts
```

On AML-shaped spheres start with `structuring` — it is the only find_motif branch with material recall lift on IBM AML benchmarks (`cycle_3`, `round_trip`, `pass_through` branches are effectively inactive there because AML fraud cycles are 4+ hops or open chains, not 3-node closed triads). `cycle_2` remains useful as the fast flash-burst prior. `fan_out` captures concentrator hubs. `cycle_3` is retained as a non-AML-default typology match for domains where closed triads appear (crypto wash-trade rings, some fraud typologies outside the IBM AML label universe).

**Structuring thresholds per jurisdiction.** Default `amt1_min=10000, amt2_max=10000` match the US Currency Transaction Report (CTR) threshold. Adjust per sphere:

| Jurisdiction | `amt1_min` | `amt2_max` | Rationale |
|---|---|---|---|
| US CTR | 10000 | 10000 | FinCEN 31 CFR 1010.311 |
| UK STR (cash) | 10000 | 10000 | MLR 2017, threshold in GBP |
| EU CTR | 10000 | 10000 | AMLD5 Article 11, threshold in EUR |
| Crypto exchange (typical) | 3000 | 3000 | Varies by exchange — consult risk policy |
| Custom / investigative | user-specified | user-specified | For suspected-pattern hunts with known amount signature |

`amt1_min` and `amt2_max` are part of the ranking LRU cache key, so changing thresholds on the same pattern triggers recompute. Budget one cold call per unique (pattern, threshold pair) in a session.

First call per (pattern, motif_type, window) is cold — 30–90s on >500k-entity patterns. Subsequent calls hit an LRU cache (cap 8). Plan exploration with this in mind: pick one motif_type, spend the cold cost, drill down on the top results before switching to a different motif_type.

**Warm-up order for cold-session cost.** The per-pattern adjacency index is shared across eight graph primitives (`find_counterparties`, `entity_flow`, `contagion_score`, `discover_chains`, `anomalous_edges`, `find_geometric_path`, `detect_network_novelty`, and motif ranking). The first of these primitives called on a given pattern pays the full edge-table materialization; every subsequent call that touches the same pattern reuses it. If the investigation starts with `find_anomalies` → `find_counterparties(suspect, pattern_id)` on the top candidates, the adjacency is already warm by the time `find_high_potential_motifs` is called — the motif cold cost drops by roughly half. Conversely, starting with a cold `find_high_potential_motifs` and only later calling `find_counterparties` pays the cold cost on motif ranking first. Either way works; the order only affects which call wears the one-time materialization.

**Scale threshold — when NOT to use `find_high_potential_motifs`.** Global motif ranking materializes the full pattern adjacency in memory (roughly 200 bytes / edge in Python tuple overhead). Up to ~10M edges (covers Berka, NYC Taxi, IBM AML HI-small / LI-small) the default path is fine. Above that, memory and cold build time become prohibitive. On spheres where `edge_count > 10M` switch to the seed-first pattern:

```
find_anomalies(pattern_id, top_n=1000)              # cheap, pre-computed delta_norm
for seed in top_anomalous_seeds:
    score_motif(seed, motif_type="cycle_2", pattern_id)
    score_motif(seed, motif_type="cycle_3", pattern_id)
```

`score_motif` uses a Lance BTREE point query (`read_edges(from_keys=[seed])`) per call — ~10ms per seed regardless of sphere size, zero full-adjacency materialization. The trade-off: this only finds motifs seeded at already-anomalous entities. For detecting novel motifs whose endpoints are not yet individually anomalous, the global ranking path is required — budget accordingly.

**`motif_potential` enrichment on trace_root_cause.** `trace_root_cause.edge_counterparty.evidence` now automatically includes a `motif_potential` block when the suspect is the seed of a high-scoring motif that passes through the counterparty. This closes a gap — an analyst doesn't need a separate call to find out "is there a triad around this pair". When you see `edge_potential.score` AND `motif_potential.score` both above p95, the suspect-counterparty relationship is both edge-rare (single tx, distant deltas) and structurally-rare (part of a named AML motif). Treat as top priority for Phase 3 escalation.

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

### R1 — Mirror Transaction Detection

**Pattern:** Entity A sends to B, and B sends back to A the same (or similar) amount within the same day. Classic circular flow indicator.

**Tool sequence:**
1. `discover_chains(primary_key, pattern_id, direction="both", time_window_hours=24, min_hops=2)` — find short loops
2. Filter chains where `keys[0] == keys[-1]` (cyclic) or where the chain returns to a known counterparty
3. `anomalous_edges(from_key, to_key, pattern_id)` — inspect individual transactions between the mirror pair
4. Compare amounts: if `abs(edge_a.amount - edge_b.amount) / max(amounts) < 0.05` → strong mirror signal

**Interpretation:** Mirror ratio > 0.95 with same-day timing is a strong indicator. Check if both edges are individually anomalous (`is_anomaly=true` in event geometry).

### R2 — Pass-Through / Rapid Layering

**Pattern:** Entity receives funds and sends within 2 hours. The entity is a conduit, not a destination.

**Tool sequence:**
1. `discover_chains(primary_key, pattern_id, time_window_hours=2, min_hops=2, max_chains=50)` — find rapid chains
2. `entity_flow(primary_key, pattern_id)` — check if `net_flow ≈ 0` (pass-through entities have balanced flow)
3. `anomalous_edges(from_key, to_key, pattern_id)` — inspect individual transactions at the bottleneck hop

**Interpretation:** net_flow near zero + chains with tight time windows = layering. `degree_velocity` showing acceleration confirms recent ramp-up.

### R3 — Burst Detection (Structuring / Smurfing)

**Pattern:** Many transactions to the same target within 24 hours, each below a reporting threshold.

**Tool sequence:**
1. `discover_chains(primary_key, pattern_id, time_window_hours=24, max_chains=100)` — find all outgoing activity
2. Group chains by terminal entity — look for repeated targets
3. `anomalous_edges(from_key, target_key, pattern_id, top_n=50)` — get all edges to the repeated target
4. Check if individual amounts are below a threshold but sum exceeds it

**Interpretation:** 5+ transactions to same target in 24h with amounts clustered just below a round reporting threshold is a strong structuring signal.

### R4 — Weighted Reciprocity

**Pattern:** Balanced bidirectional flow between two entities — min(out, in) / max(out, in) close to 1.0.

**Tool sequence:**
1. `entity_flow(primary_key, pattern_id)` — get per-counterparty net flow
2. For each counterparty with both outgoing AND incoming: `reciprocity = min(out, in) / max(out, in)`
3. `anomalous_edges(from_key, counterparty_key, pattern_id)` — inspect the transactions

**Interpretation:** Reciprocity > 0.8 between two entities = suspicious round-tripping. Cross-reference with `contagion_score` — if the counterparty is also contagious, the pair is high-priority.

### R5 — Financial Profile (Mule Detection)

**Pattern:** Entity's total flow reveals its role: source (high out, low in), sink (high in, low out), or mule (high both, near-zero net).

**Tool sequence:**
1. `entity_flow(primary_key, pattern_id)` — get totals
2. `cross_pattern_profile(primary_key, line_id)` — anomaly status across all patterns
3. Classify: `net_flow > 0.7 * outgoing_total` → source; `net_flow < -0.7 * incoming_total` → sink; else → mule candidate

**Interpretation:** Mule candidates (balanced flow, multiple patterns flagged) warrant `propagate_influence` to map the network they serve.

### R6 — Concentration Risk

**Pattern:** A single counterparty dominates an entity's flow — potential control relationship.

**Tool sequence:**
1. `entity_flow(primary_key, pattern_id, top_n=5)` — get top counterparties by abs(net_flow)
2. Compute `concentration = abs(top_1_net_flow) / (outgoing_total + incoming_total)`
3. `contagion_score(primary_key, pattern_id)` — check if the concentrated counterparty is anomalous

**Interpretation:** Concentration > 0.6 means one counterparty controls >60% of flow. If that counterparty is anomalous (contagion), the entity is at high risk.

### R7 — Benford's Law (Amount Distribution)

**Pattern:** Natural financial data follows Benford's Law for first digits. Fabricated transactions often don't.

**Tool sequence:**
1. `find_counterparties(primary_key, line_id, from_col, to_col, pattern_id)` — get top counterparties
2. For each counterparty pair: `anomalous_edges(from_key, to_key, pattern_id, top_n=50)` — collect per-transaction amounts
3. Aggregate all `edge.amount` values across counterparties, compute first-digit distribution
4. Compare with expected Benford distribution: 1→30.1%, 2→17.6%, 3→12.5%, etc.

**Note:** `discover_chains` returns only `total_amount` per chain (aggregate sum), not individual transaction amounts. Use `anomalous_edges` to get per-transaction amounts needed for Benford analysis.

**Interpretation:** Chi-squared test against Benford expected frequencies. p-value < 0.05 = amounts are likely not organic. Most effective with 100+ transactions.

### R8 — Witness Cohort Discovery (Fraud Cohort Expansion)

**Pattern:** A confirmed launderer X has a ring of accomplices. Some are already in X's counterparty network (visible via `find_counterparties`). Others share X's anomaly signature — same witness dimensions, drifting in the same geometric direction — but are NOT yet connected to X via transactions. `find_witness_cohort` surfaces these geometric peers, ranked by composite witness/delta/trajectory/anomaly score, with already-connected entities filtered out.

**Honest scope:** This is **investigative cohort expansion**, NOT edge forecasting. The function does NOT predict that X and the cohort members will transact in the future. It surfaces existing peers worth investigating, not future connections.

**Tool sequence:**
1. Identify a confirmed or suspected launderer `X` (from typology recipes R1–R7 or external intel)
2. `find_witness_cohort(X, anchor_pattern_id, top_n=10)` — top peers excluding existing counterparties
3. Inspect `members[]` — focus on entries with `witness_overlap >= 0.5` AND `is_anomaly == true`
4. For each candidate `Y`: `cross_pattern_profile(Y, line_id)` to verify multi-pattern confirmation
5. For high-confidence candidates: `find_witness_cohort(Y, anchor_pattern_id)` — recursive expansion to map the cohort

**Interpretation:** A cohort member with `witness_overlap = 1.0` and `trajectory_alignment > 0.95` is strong — shares the SAME structural anomaly signature and the same geometric drift direction as X. The lack of an existing edge is the agent-guidance value: existing counterparties are skipped (often legitimate), so the cohort is denser in unknown peers worth investigating.

**False positive guard:** Two competitors or two unrelated entities can also share witness profiles. Use cohort members as INVESTIGATIVE RANKING, not as evidence. Combine with domain context before escalation. Many cohort members will not be laundering even when the seed is — the function narrows the search space, it does not eliminate the need for human verification.

**Why this is unique vs other tools:**
- `find_similar_entities + is_anomaly = true`: returns shape twins via plain ANN. `find_similar_entities` does not exclude existing counterparties (often legitimate), does not score witness overlap, and does not weight trajectory alignment
- Neo4j GDS link prediction: topological features only (Adamic-Adar, common neighbors), no witness sets, no population-relative geometry
- ML link prediction (Node2Vec, GraphSAGE): requires training, no interpretability, no labeled data needed for hypertopos
- Vector DB ANN: nearest neighbors but no graph awareness, no edge exclusion

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

## Examples

### Example 1: Full AML screening

User says: "Screen this sphere for money laundering"

Actions:
1. `passive_scan("accounts", threshold=2)` — multi-source screening
2. For top 5 suspects: `cross_pattern_profile(pk, "accounts")` — triage by source_count and risk_score
3. For source_count >= 2: `get_polygon(pk, "account_pattern")` — read anomaly_dimensions
4. `find_counterparties(pk, "transactions", "from_account", "to_account", pattern_id="account_pattern")` — network check

Result: "Screened 515K accounts. 847 flagged by 2+ sources. Top suspect: account 800737690 (source_count=3, risk_score=2.1, connected_risk=87). Anomaly driven by: n_currencies_out (4.1 sigma), burst_tx_out (3.8 sigma)."

### Example 2: Typology detection

User says: "Check for round-tripping patterns"

Actions:
1. `discover_chains(suspect, pattern_id="transaction_pattern", time_window_hours=24, max_hops=5, min_hops=2)` — runtime chain discovery
2. Filter: cyclic chains with `n_distinct_categories >= 2`
3. For each cyclic chain: `cross_pattern_profile(first_key)` — multi-source confirmation
4. Optional: `find_geometric_path(from_key=A, to_key=A, scoring="anomaly")` — trace the ring path through anomalous intermediaries

Result: "Found 12 cyclic chains under 24h. 3 chains with source_count >= 2 flagged as ROUND_TRIPPING_3PARTY. Top chain: A→B→C→A, total_amount top 1%, 3 currencies involved."

### Example 3: False positive elimination

User says: "Is account 8013C4030 really suspicious?"

Actions:
1. `cross_pattern_profile("8013C4030", "accounts")` — source_count=1 (single source)
2. `find_similar_entities("8013C4030", "account_pattern", filter_expr="is_anomaly = false", top_n=20)` — 18 normal accounts with identical shape
3. `find_counterparties("8013C4030", ...)` — all counterparties normal

Result: "Likely false positive. Single-source only, 18 normal geometric twins, all counterparties clean. Recommend close alert."

## Troubleshooting

### `passive_scan` not available
Cause: Tool may not be configured in this MCP version.
Solution: Manually combine `find_anomalies` across all patterns (account, pair, chain) and intersect results. See Phase 1 manual fallback recipe.

### `extract_chains` returns empty or times out
Cause: No chain pattern built in sphere, or missing `seed_nodes` parameter (full BFS hangs on hubs).
Solution: Use `discover_chains(primary_key, pattern_id, max_hops=5)` instead — it runs
temporal BFS on the edge table at query time and does not require pre-built chain_lines.
If `discover_chains` is not available, pass `seed_nodes=[suspect_list]` to `extract_chains`.
Verify edge table exists via `edge_stats(pattern_id)`.

### All suspects have source_count=1
Cause: Sphere has only one pattern (account only). Pair and chain patterns not built.
Solution: Rebuild sphere with `composite_lines` (pairs) and `chain_lines` (chains). Single-pattern detection has significantly lower recall than multi-pattern.

### `find_counterparties` returns too many results
Cause: Hub account with hundreds of counterparties.
Solution: Focus on anomalous counterparties only (filter by `is_anomaly=true` in results). Use `cross_pattern_profile` for quick triage instead of enumerating all counterparties.
