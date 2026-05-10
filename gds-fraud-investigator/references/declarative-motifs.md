# Declarative Motif Cheatsheets

> Loaded on demand when designing custom motif queries via `find_motif_by_hops`. The closed-vocab atoms (`find_motif_structuring`, `cycle_2`, `fan_out`, `fan_in`, `bipartite_burst`, `chain_k`, `split_recombine`) cover the standard typologies; reach for `find_motif_by_hops` only when the closed vocab doesn't fit the query shape.

## Declarative structuring chain detection

When the closed-vocab `find_motif_structuring` is too rigid (you want a different chain length, a custom amount-decay shape, or a per-hop edge-dim filter), reach for `find_motif_by_hops` with the `amount_ratio_to_prev` predicate. The classic deposit → split → wire signature is "each next hop carries materially less than the previous one" — that maps directly to a per-hop ratio cap.

```yaml
# 3-hop chain: deposit → split → wire
# Each subsequent hop must be ≤ 60% of the previous hop's amount.
hops:
  - amount_min: 10000.0          # hop[0] — deposit at or above CTR floor
  - amount_ratio_to_prev: 0.6    # hop[1] — split is ≤ 60% of deposit
  - amount_ratio_to_prev: 0.6    # hop[2] — wire is ≤ 60% of split
```

```python
nav.find_motif_by_hops(
    pattern_id="tx_pattern",
    hops=[
        HopPredicate(amount_min=10000.0),
        HopPredicate(amount_ratio_to_prev=0.6, time_delta_max_hours=24.0),
        HopPredicate(amount_ratio_to_prev=0.6, time_delta_max_hours=24.0),
    ],
    seed_keys=suspect_account_ids,   # restrict to candidate sources
    max_results=200,
)
```

**Rules:**
- `amount_ratio_to_prev` must be in `(0, 1.0]`. Values >1 are rejected at validation (growth-cap semantic is intentionally not overloaded onto this field).
- `hops[0].amount_ratio_to_prev` must be `None` (no previous amount to compare against). Same first-hop rule as `time_delta_max_hours`.
- Edges where either `prev_amount ≤ 0` or `current_amount ≤ 0` are silently skipped, matching the existing `find_motif_structuring` convention.
- Combine with `time_delta_max_hours` to bound the chain temporally — the ratio alone says nothing about how fast the cascade runs.

**When to prefer this over `find_motif_structuring`:**
- Chain length ≠ 4 (the closed-vocab atom is fixed at A→B→C→D).
- Custom decay curve (e.g. ratio 0.3 between hop 0 and hop 1, then 0.8 between later hops — `find_motif_structuring` only enforces `amt2_max` which is an absolute floor).
- Need to layer per-hop edge-dim filters (e.g. `pair_edge_count >= 5` on later hops to scope to recurring counter-party pairs).

## Long-chain layering with a global time window

Per-hop `time_delta_max_hours` bounds *consecutive* hops; it does not say anything about total chain duration. For a long layering cascade where the operator cares about "the entire chain must fit inside one week" (regardless of how long each individual hop takes), use the top-level `time_window_hours` parameter — it caps `abs(current_edge_ts - first_edge_ts)` on every hop after the first.

```python
# 8-hop layering chain that must fit inside a week, with per-hop
# decay and a per-hop fast-window guard.
nav.find_motif_by_hops(
    pattern_id="tx_pattern",
    hops=[
        HopPredicate(amount_min=10000.0),
        *[
            HopPredicate(
                amount_ratio_to_prev=0.7,
                time_delta_max_hours=24.0,
            )
            for _ in range(7)
        ],
    ],
    seed_keys=suspect_account_ids,
    max_results=100,
    time_window_hours=168.0,   # one week total chain span
)
```

**Rules for `time_window_hours`:**
- Optional, default `None` — when omitted, only per-hop windows apply.
- Must be strictly positive when set; bad values raise before any sphere-state-dependent early-return (so misconfigured calls on edge-table-less spheres surface as errors rather than silent empty results).
- Independent semantic from `time_delta_max_hours` — both apply when both are set; `time_window_hours` is the looser global cap, `time_delta_max_hours` is the per-hop cap.
- Hop count cap: `len(hops)` is `1..8` (matches the `chain_k` motif vocabulary).

## Filter chains by anomalous intermediaries

When the operator already has a population of flagged-anomalous accounts (calibrated `is_anomaly=True` in the anchor pattern) and wants to surface chains that *route through* those accounts — classic "structuring chains via known suspicious nodes" — use the per-hop `require_anomalous_entity` predicate. The flag enforces that the hop's destination entity (`nodes[i+1]` of the resulting motif) carries `is_anomaly=True` in the resolved anchor companion pattern. Multiple hops can set this independently; constraints AND across hops.

```python
# 2-hop chain where BOTH intermediaries are themselves anomalous.
# Use case: structuring detection where the operator only cares about
# chains touching known-suspicious accounts on every hop.
nav.find_motif_by_hops(
    pattern_id="tx_pattern",
    hops=[
        HopPredicate(
            amount_min=1000.0,
            require_anomalous_entity=True,   # nodes[1] anomalous
        ),
        HopPredicate(
            amount_ratio_to_prev=0.7,
            require_anomalous_entity=True,   # nodes[2] anomalous
        ),
    ],
    seed_keys=suspect_account_ids,
    max_results=50,
)
```

**Rules:**
- Per-hop `bool`, default `False` (no-op preserving prior behavior).
- `require_anomalous_entity=True` on hop `i` enforces `is_anomaly=True` on `nodes[i+1]`. Seed (`nodes[0]`) is never checked — pre-filter `seed_keys` upfront if seed coverage is needed.
- Filter runs at the navigator post-BFS, pre-scoring (saves scoring work on motifs that get dropped).
- `max_results` applies AFTER the filter, so a restrictive flag combination can return fewer than `max_results` motifs. Bump `max_results` if you need more output.
- Raises when the queried event pattern has no anchor companion configured, or when the anchor pattern has no `is_anomaly` column (calibration must run first).
- No sphere format change, no rebuild required — `is_anomaly` is already populated at calibration time.

**When to prefer this over post-filtering motif results yourself:**
- You want to combine anomaly filtering with `score=True` ranking — the post-BFS filter runs *before* scoring, so the score field reflects only the kept motifs.
- You want the `n_results` count to mirror what the filter actually surfaced (post-yourself filter would have to overshoot `max_results`).
