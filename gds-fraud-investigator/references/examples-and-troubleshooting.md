# Examples + Troubleshooting

> Loaded on demand. Worked end-to-end examples (full AML screening, typology detection, false-positive elimination) plus the canonical troubleshooting tree for the most common error modes.

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
Solution: Use `discover_chains(primary_key, pattern_id, max_hops=5)` instead — it runs temporal BFS on the edge table at query time and does not require pre-built chain_lines. If `discover_chains` is not available, pass `seed_nodes=[suspect_list]` to `extract_chains`. Verify edge table exists via `edge_stats(pattern_id)`.

### All suspects have source_count=1
Cause: Sphere has only one pattern (account only). Pair and chain patterns not built.
Solution: Rebuild sphere with `composite_lines` (pairs) and `chain_lines` (chains). Single-pattern detection has significantly lower recall than multi-pattern.

### `find_counterparties` returns too many results
Cause: Hub account with hundreds of counterparties.
Solution: Focus on anomalous counterparties only (filter by `is_anomaly=true` in results). Use `cross_pattern_profile` for quick triage instead of enumerating all counterparties.
