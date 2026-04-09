---
name: gds-fraud-investigator
description: AML/fraud detection and financial anomaly investigation via GDS geometric analysis — 3-phase process with 25 typology recipes. Use when user asks to "detect fraud", "screen for money laundering", "find suspicious accounts", "run AML scan", "investigate suspicious transactions", "typology detection", "passive scan for fraud", "detect insider trading", "find anomalous billing", "investigate suspicious transactions", or "financial crime detection". Use this skill for ANY financial anomaly investigation, not just AML. Requires hypertopos MCP server with a financial transaction sphere (account + pair + chain patterns).
license: Apache-2.0
compatibility: Requires hypertopos MCP server with a financial transaction sphere (account, pair, chain patterns).
metadata:
  author: Karol Kędzia
  version: 0.1.0
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

Auto-discover with borderline:
```
passive_scan(home_line_id="accounts", include_borderline=true, borderline_rank_threshold=80)
```

If `passive_scan` is not available, manually combine:
```
1. find_anomalies("account_pattern", top_n=50)     → anomalous accounts
2. For pair patterns: execute EVERY profiling_alert call (rank_by_property on each alerted dim)
   If total_found >> 50: aggregate_anomalies("pair_pattern", group_by="account_key")
3. find_anomalies("chain_pattern", top_n=50)        → anomalous chains → expand chain_keys
4. Intersect: accounts in 2+ sources = high confidence
```

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

### Recipe: Network Expansion

From a confirmed suspect, expand the investigation network:

```
1. find_counterparties(suspect)           → list of transaction partners
2. Filter anomalous counterparties        → subset with is_anomaly=true
3. For each anomalous counterparty:
     cross_pattern_profile(cp_key)        → are THEY multi-source flagged?
4. extract_chains(seed_nodes=[suspect])   → chains involving this account
5. Filter: is_cyclic=true                 → round-trip chains
```

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
| FAN-OUT | high out_degree + many dest_banks | find_counterparties → check all targets |
| FAN-IN | high in_sources | find_counterparties → who feeds this account? |
| CYCLE | pair anomalies + chain is_cyclic | find_counterparties → counterpart overlap |
| STACK | delta_rank_pct 90-95 (borderline) | check chain membership |
| BIPARTITE | pair anomalies + community | aggregate(group_by_property="community_id") |
| RANDOM | chain anomalies | highest chain recall pattern |

## Common Pitfalls

- `is_anomaly` alone misses most fraud — check each pattern separately and combine signals
- `find_similar_entities` returns shape twins, not new suspects — use `find_counterparties` for network expansion
- High `delta_norm` can mean legitimate high-activity entity — cross-reference with business context
- Pair pattern captures relationship anomalies invisible at account level — always check it
- `extract_chains` without `seed_nodes` causes hub monopolization — pass seed accounts
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
1. `extract_chains(seed_nodes=[suspects], min_hops=2, max_hops=5)` — find chains
2. Filter: `is_cyclic=true AND time_span_hours <= 24 AND n_distinct_categories >= 2`
3. For each cyclic chain: `cross_pattern_profile(first_key)` — multi-source confirmation

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
Solution: Always pass `seed_nodes=[suspect_list]`. Verify chain pattern exists via `get_sphere_info()`.

### All suspects have source_count=1
Cause: Sphere has only one pattern (account only). Pair and chain patterns not built.
Solution: Rebuild sphere with `composite_lines` (pairs) and `chain_lines` (chains). Single-pattern detection has 17.9% recall vs 52.1% with 3 patterns.

### `find_counterparties` returns too many results
Cause: Hub account with hundreds of counterparties.
Solution: Focus on anomalous counterparties only (filter by `is_anomaly=true` in results). Use `cross_pattern_profile` for quick triage instead of enumerating all counterparties.
