---
name: gds-fraud-investigator
description: AML/fraud detection and financial anomaly investigation via GDS geometric analysis — 3-phase process with 25 typology recipes. Use when user asks to "detect fraud", "screen for money laundering", "find suspicious accounts", "run AML scan", "investigate suspicious transactions", "typology detection", "passive scan for fraud", "detect insider trading", "find anomalous billing", "investigate suspicious transactions", or "financial crime detection". Use this skill for ANY financial anomaly investigation, not just AML. Requires hypertopos MCP server with a financial transaction sphere (account + pair + chain patterns).
license: Apache-2.0
compatibility: Requires hypertopos MCP server with a financial transaction sphere (account, pair, chain patterns).
metadata:
  author: Karol Kędzia
  version: 0.2.3
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
edge_stats(pattern_id="transaction_pattern")
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
passive_scan("accounts", threshold=2)
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
passive_scan(home_line_id="accounts", sources='[
  {"type": "geometry", "pattern_id": "account_pattern"},
  {"type": "borderline", "pattern_id": "account_pattern", "rank_threshold": 80},
  {"type": "points", "line_id": "accounts", "rules": {"n_currencies_out": [">=", 2]}, "combine": "AND"}
]')
```

Auto-discover with borderline (also auto-detects graph contagion source if edge table exists):
```
passive_scan(home_line_id="accounts", include_borderline=true, borderline_rank_threshold=80)
```

After passive_scan, batch-score the top suspects by neighborhood contamination:
```
contagion_score_batch(suspect_keys, pattern_id)
```
Entities with high contagion ratio are network hubs, not isolated actors — prioritize them.

**As-of reconstruction:** `contagion_score`, `contagion_score_batch`, `entity_flow`, `degree_velocity`, `propagate_influence`, and `find_counterparties` all accept an optional `timestamp_cutoff` parameter (Unix seconds). When set, only edges with `timestamp <= cutoff` are considered — use this to reconstruct what an entity's neighborhood looked like at the time of a known incident, or to validate a detection recipe retroactively against a prior snapshot of the graph.

If `passive_scan` is not available, manually combine:
```
1. find_anomalies("account_pattern", top_n=50)     → anomalous accounts
2. For pair patterns: execute EVERY profiling_alert call (rank_by_property on each alerted dim)
   If total_found >> 50: aggregate_anomalies("pair_pattern", group_by="account_key")
3. find_anomalies("chain_pattern", top_n=50)        → anomalous chains → expand chain_keys
4. Intersect: accounts in 2+ sources = high confidence
```

### FDR control and diverse selection

When using `find_anomalies` for fraud screening, set `fdr_alpha=0.05` to apply Benjamini-Hochberg FDR control — false positives waste investigator time and erode trust in the alert pipeline, so controlling the false discovery rate is critical. When requesting K>10 results, use `select="diverse"` to surface different fraud typologies (structuring, layering, round-tripping) instead of 50 variants of the same high-volume pattern; this leverages submodular facility location to maximize typological coverage in the result set. Both parameters also apply to `attract_boundary`, `find_hubs`, and `find_drifting_entities`.

### Recipe: Risk Triage

For each suspect from Phase 1:

```
cross_pattern_profile(suspect_key, line_id="accounts")
```

Interpret the result:
- `source_count >= 3` → investigate immediately (anomalous in all layers)
- `source_count == 2` → high priority
- `source_count == 1` → likely false positive, deprioritize
- `risk_score > 0.5` → high anomaly density across patterns
- `connected_risk > 80` → counterparties are also anomalous (network signal)

Sort suspects by `risk_score` descending for investigation order.

## Phase 2 — Investigation (per suspect)

### Recipe: Entity 360

Full investigation of a single suspect:

```
1. cross_pattern_profile(pk)              → multi-source risk overview
2. get_polygon(pk, "account_pattern")     → anomaly_dimensions (WHY anomalous)
3. find_counterparties(pk, "transactions",
     "from_account", "to_account",
     pattern_id="account_pattern")        → WHO they transact with
4. dive_solid(pk, "account_pattern")      → WHEN behavior changed
```

Key signals:
- `anomaly_dimensions` shows WHICH behavioral features drive the detection
- Counterparties with `is_anomaly=true` = network confirmation
- Temporal burst → silence pattern = classic placement/layering

Before closing: `investigation_coverage(pk, pattern_id, explored_keys)` — confirms explored vs unexplored counterparties. Low coverage means additional network exploration warranted.

### Recipe: Network Expansion

From a confirmed suspect, expand the investigation network:

```
1. find_counterparties(suspect)           → list of transaction partners
2. Filter anomalous counterparties        → subset with is_anomaly=true
3. For each anomalous counterparty:
     cross_pattern_profile(cp_key)        → are THEY multi-source flagged?
4. discover_chains(suspect, pattern_id="transaction_pattern",
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
  pattern_id="transaction_pattern", scoring="anomaly")
```
This traces how two suspects are connected through the transaction graph. `scoring="anomaly"`
prioritizes paths through anomalous intermediaries — the most suspicious route between them.
Use `scoring="geometric"` to find paths through geometrically unusual entities, or
`scoring="shortest"` for the most direct connection.

### Recipe: FP Elimination

Before closing an alert, check for exculpatory evidence:

```
1. find_similar_entities(suspect, "account_pattern",
     filter_expr="is_anomaly = false", top_n=20)
2. If 10+ normal accounts have identical shape → suspect is likely
   a legitimate high-activity entity, not a launderer
3. Check: are counterparties ALL normal? → further FP evidence
```

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

**Interpretation:** 5+ transactions to same target in 24h with amounts clustered below a round number (e.g., 9,500 when reporting threshold is 10,000) is a strong structuring signal.

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
Solution: Rebuild sphere with `composite_lines` (pairs) and `chain_lines` (chains). Single-pattern detection has 17.9% recall vs 52.1% with 3 patterns.

### `find_counterparties` returns too many results
Cause: Hub account with hundreds of counterparties.
Solution: Focus on anomalous counterparties only (filter by `is_anomaly=true` in results). Use `cross_pattern_profile` for quick triage instead of enumerating all counterparties.
