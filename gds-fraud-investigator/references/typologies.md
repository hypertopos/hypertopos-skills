# Phase 3 — Typology Detection

Each typology is a sequence of MCP calls. Execute step-by-step.

### Typology 1: Structured Layering

Multi-hop structuring with rounded amounts across jurisdictions.

```
1. passive_scan("accounts", threshold=2)
2. For top suspects: get_polygon("account_pattern")
3. Filter: amount_uniformity > 0.7
         AND n_currencies_out >= 2
         AND n_dest_banks >= 3
4. For passing: extract_chains(seed_nodes=[pk], min_hops=3)
5. Filter chains: hop_count >= 3 AND n_distinct_categories >= 2
6. Flag: STRUCTURED_LAYERING
```

### Typology 2: Flash-Burst Round-Trip

Short intense burst with return flow (A->X->A in <24h).

```
1. passive_scan("accounts", threshold=1)
2. For suspects: get_polygon("account_pattern")
3. Filter: burst_tx_out > 10 AND n_currencies_out >= 2
4. extract_chains(seed_nodes=[pk], time_window_hours=24)
5. Filter chains: is_cyclic=true AND time_span_hours < 24
         AND total_amount in top percentile for population
6. Confirm: cross_pattern_profile(pk) → connected_risk > 80
7. Flag: FLASH_BURST_ROUNDTRIP
```

### Typology 3: Round-Tripping 3-Party

Classic A->B->C->A cycle in short time window.

```
1. extract_chains(seed_nodes=[suspects], min_hops=2, max_hops=5)
2. Filter: is_cyclic=true
         AND time_span_hours <= 24
         AND n_distinct_categories >= 2
         AND total_amount in top percentile for population
3. For each cyclic chain: cross_pattern_profile(first_key)
4. Flag if source_count >= 2: ROUND_TRIPPING_3PARTY
```

### Typology 4: Bidirectional Burst

Intense ping-pong between two accounts (A<->B).

```
1. find_anomalies("pair_pattern", top_n=50)
2. For anomalous pairs: split key on "→"
3. find_counterparties(account_A) → check if B in both outgoing AND incoming
4. Filter: pair_tx_count > 10 AND pair_sum_amount in top percentile
5. Confirm: both A and B anomalous in account_pattern
6. Flag: BIDIRECTIONAL_BURST
```

### Typology 5: Long-Cycle Multi-Stage Layering

4+ hop chain across multiple jurisdictions.

```
1. extract_chains(seed_nodes=[suspects], min_hops=4)
2. Filter: hop_count >= 4
         AND total_amount in top percentile for population
         AND n_distinct_categories >= 2
3. For chain members: cross_pattern_profile(key)
4. Flag if any member source_count >= 2: MULTI_STAGE_LAYERING
```

### Typology 6: Offshore Hub

Shell-company hub connecting jurisdictions. Requires jurisdiction risk data.

```
1. passive_scan("accounts", threshold=2)
2. cross_pattern_profile(pk) → source_count >= 2 AND risk_score > 0.5
3. find_counterparties(pk) → get counterparty list
4. Check: n_dest_banks >= 3 AND n_currencies_out >= 3
5. If counterparty in HIGH_RISK jurisdiction → flag
6. Flag: OFFSHORE_HUB
```

Requires: `jurisdiction_risk` as tracked_property on accounts.

### Typology 7: Salary Abuse

Salary account with unexpected cross-border high-value activity.

```
1. search_entities("accounts", "account_type", "salary")
2. For each: get_polygon("account_pattern")
3. Filter: n_dest_banks >= 2 AND sum_out in top percentile AND tx_out_count > 5
4. cross_pattern_profile(pk) → source_count >= 1
5. Flag: SALARY_ABUSE
```

Requires: `account_type` column on accounts line.

### Typology 8: Flash-Burst Multi-Hop

Very intense burst with multi-hop chain and return flow.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern")
3. Filter: burst_tx_out > 15 AND n_currencies_out >= 2
4. extract_chains(seed_nodes=[pk], time_window_hours=1)
5. Filter: total_amount in top percentile for population AND n_distinct_categories >= 2
6. Check return flow: extract_chains with reverse seed
7. Flag: FLASH_BURST_MULTIHOP
```

### Typology 9: Shell-Company Layering

Corporation/shell entity with high counterparty volume and cross-border flow.
Requires: `account_type` column on accounts line.

```
1. search_entities("accounts", "account_type", "corporation")
2. For each: get_polygon("account_pattern")
3. Filter: n_out_targets >= 10 AND n_currencies_out >= 2
         AND n_dest_banks >= 2 AND tx_out_count > 20
4. cross_pattern_profile(pk) → source_count >= 2
5. find_counterparties(pk) → check if counterparties span multiple segments
6. Flag: SHELL_COMPANY_LAYERING
```

### Typology 10: Acceleration with Dormancy

Dormant account suddenly active with high-value cross-border transactions.
Uses `max_rolling_z` from temporal build.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern")
3. Filter: max_rolling_z > 3.0 (behavioral regime change)
         AND burst_tx_out > 10
         AND n_currencies_out >= 2
4. Confirm dormancy: dive_solid(pk) → early slices near-zero, late slices spike
5. cross_pattern_profile(pk) → connected_risk assessment
6. Flag: ACCELERATION_DORMANCY
```

### Typology 11: Multi-Round-Tripping

Repeated cyclic flows between same accounts (not one cycle but many).

```
1. extract_chains(seed_nodes=[suspects], min_hops=2, max_hops=4)
2. Group chains by (first_key, last_key) pair
3. Filter groups: count >= 3 (same pair cycles 3+ times)
         AND total_amount across cycles in top percentile
         AND n_distinct_categories >= 2
4. For flagged pairs: cross_pattern_profile(each key)
5. Flag: MULTI_ROUND_TRIPPING
```

### Typology 12: Parallel Layering (Multiple Chains to Same Target)

Multiple independent chains converge on the same destination account.

```
1. extract_chains(seed_nodes=[suspects], min_hops=2)
2. Group chains by last_key (destination account)
3. Filter destinations: count_chains >= 3 (3+ independent chains arrive here)
         AND sum(total_amount across chains) in top percentile
         AND n_distinct first_keys >= 3 (different origin accounts)
4. For each flagged destination: cross_pattern_profile(dest_key)
5. Flag: PARALLEL_LAYERING
```

### Typology 13: Concentrator / Sink Account

Account receives many small payments, then sends few large ones.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern")
3. Filter: in_degree >= 10 (many incoming sources)
         AND fan_asymmetry < 0.3 (mostly receiving)
         AND max_out > 10 * mean_out (few large outgoing vs many small incoming)
4. cross_pattern_profile(pk) → check pair anomalies
5. find_counterparties(pk) → verify many-to-few flow pattern
6. Flag: CONCENTRATOR_SINK
```

### Typology 14: Geographic Spread

Flow spread across many jurisdictions with no single country dominating.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern")
3. Filter: n_dest_banks >= 5 AND n_currencies_out >= 3
4. find_counterparties(pk) → count distinct bank jurisdictions
5. Check: no single destination gets > 50% of outflow
6. cross_pattern_profile(pk) → source_count assessment
7. Flag: GEOGRAPHIC_SPREAD
```

### Typology 15: Attenuation Pattern (Many Small-Step Chains)

Multiple chains where each hop is below reporting thresholds but aggregate is significant.

```
1. extract_chains(seed_nodes=[suspects], min_hops=3)
2. Filter chains: amount_cv < 0.3 (uniform small amounts per hop)
         AND total_amount in top percentile for population
         AND hop_count >= 3
         AND n_distinct_categories >= 2
3. Group by seed account: count_chains >= 3
4. cross_pattern_profile(seed) → multi-source confirmation
5. Flag: ATTENUATION_PATTERN
```

### Typology 16: Mirror-Flow Burst

Multiple counterparties with near-symmetric in/out flows (A sends ~X to B, B sends ~X back).

```
1. passive_scan("accounts", threshold=2)
2. find_counterparties(pk) → get top counterparties
3. For each counterparty B:
     Check: counterpart_overlap > 0 (bidirectional relationship exists)
     Compare outgoing vs incoming amounts to B
4. Filter: abs(sum_out_to_B - sum_in_from_B) / total_flow < 0.2
         AND total_flow is significant (top percentile)
         AND n_such_mirrors >= 3 (3+ counterparties with mirror flow)
5. cross_pattern_profile(pk) → connected_risk assessment
6. Flag: MIRROR_FLOW_BURST
```

### Typology 17: Seasonality Breaker

Account with historically stable regular pattern that suddenly becomes chaotic.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern") → check max_rolling_z
3. Filter: max_rolling_z > 3.0 (behavioral regime change)
4. dive_solid(pk) → inspect temporal slices
5. Check: early slices have low variance (stable pattern)
         AND late slices have high variance (chaotic)
         AND n_currencies_out >= 2
6. Flag: SEASONALITY_BREAKER
```

### Typology 18: Multi-Jurisdiction Latency Chain

Long chains with deliberate time gaps between hops (slow-motion layering).

```
1. extract_chains(seed_nodes=[suspects], min_hops=3, time_window_hours=168)
2. Filter chains: hop_count >= 4
         AND time_span_hours > 48 (spread over days, not hours)
         AND time_span_hours < 168
         AND n_distinct_categories >= 2
         AND total_amount in top percentile for population
3. Check: avg time between hops > 12h (deliberate pacing)
4. cross_pattern_profile(first_key) → multi-source confirmation
5. Flag: MULTI_JURISDICTION_LATENCY
```

### Typology 19: Multi-Direction Feedback Loop

Complex feedback pattern with direction changes (A->B->C->B->A).

```
1. extract_chains(seed_nodes=[suspects], min_hops=3,
     bidirectional=true, time_window_hours=48)
2. Check chain keys for direction reversal:
     keys[i] == keys[i+2] (bounce pattern: A->B->A)
     OR keys[0] == keys[-1] AND hop_count >= 4 (long cycle)
3. Filter: total_amount in top percentile for population
         AND n_distinct_categories >= 2
4. cross_pattern_profile(pk) → connected_risk > 70
5. Flag: FEEDBACK_LOOP
```

### Typology 20: Sub-Threshold Stacking

Many transactions always below reporting threshold but aggregating to significant amounts.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern")
3. Filter: tx_out_count > 50 (high volume)
         AND mean_out < max_out * 0.1 (many small, few large)
         AND amount_uniformity > 0.5 (suspiciously uniform amounts)
         AND n_currencies_out >= 2
4. cross_pattern_profile(pk) → source_count >= 2
5. Flag: SUB_THRESHOLD_STACKING
```

### Typology 21: Branch-Switching (Multi-Sub-Entity Flow)

Account collects from many small sub-entities, then sends few large outflows.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern")
3. Filter: in_degree >= 5 (many incoming sources)
         AND fan_asymmetry < 0.4 (mostly receiving)
         AND mean_out > 10 * mean_in (few large vs many small)
         AND n_currencies_out >= 2
4. find_counterparties(pk) → verify many small senders, few large receivers
5. Flag: BRANCH_SWITCHING
```

### Typology 22: Delta-Drift Burst

Stable account that suddenly shifts multiple behavioral dimensions at once.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern") → check max_rolling_z
3. Filter: max_rolling_z > 4.0 (extreme regime change)
4. dive_solid(pk) → count dimensions that shifted significantly
5. Check: n_anomalous_dims >= 3 (multi-dimensional shift, not just volume)
         AND n_currencies_out >= 2
6. Flag: DELTA_DRIFT_BURST
```

### Typology 23: Multi-Tier Income Layering

Collects many small incoming, relabels as payments/investments on outflow.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern")
3. Filter: in_degree >= 10 AND fan_asymmetry < 0.4
         AND tx_in_count > 20
         AND max_out > 10 * mean_in
         AND n_payment_formats >= 3 (different outgoing formats = relabeling)
4. cross_pattern_profile(pk) → check pair anomalies
5. Flag: MULTI_TIER_INCOME_LAYERING
```

### Typology 24: Multi-Format Stealth

Same counterparty tested through multiple payment channels.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern")
3. Filter: n_payment_formats >= 4 AND n_currencies_out >= 2
4. find_counterparties(pk) → get top counterparties
5. Check: same counterparty appears in 3+ different formats
6. Flag: MULTI_FORMAT_STEALTH
```

### Typology 25: Micro-Amount Stacking

Many micro-transactions individually insignificant but aggregating to large sums.

```
1. passive_scan("accounts", threshold=1)
2. get_polygon("account_pattern")
3. Filter: tx_out_count > 100
         AND mean_out < 500 (micro amounts)
         AND sum_out in top percentile (significant aggregate)
         AND n_out_targets >= 10
         AND n_currencies_out >= 2
4. cross_pattern_profile(pk) → multi-source confirmation
5. Flag: MICRO_AMOUNT_STACKING
```
