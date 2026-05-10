# Motif Detection Deep-Dive

> Loaded on demand. Closed motif vocabulary, jurisdictional structuring thresholds, cold-call optimisation, and the `motif_potential` enrichment block. The Phase 2 Entity 360 recipe in `SKILL.md` references this file for the structural-motif lookup.

## Closed motif vocabulary

Eight motif types are mapped to AML typologies via `score_motif(motif_type=...)` and `find_high_potential_motifs(motif_type=...)`. The vocabulary closes when the structural shape is named — for free-form predicates use `find_motif_by_hops` (see `declarative-motifs.md`).

- **`cycle_2`** (default 24h window) — A↔B bidirectional round-trip. Structural atom of T2 Flash-Burst Round-Trip and T4 Bidirectional Burst.
- **`cycle_3`** (default 72h window, strict temporal ordering `ts_ab < ts_bc < ts_ca`) — directed triad A→B→C→A. Structural atom of T3 Round-Tripping 3-Party, T5 Long-Cycle, T11 Multi-Round-Tripping, T19 Multi-Direction Feedback Loop. **Note:** closed-triad fraud cycles in IBM AML are effectively absent (autoresearch caught 0/0 TPs on HI+LI-small). Real AML cycles are 4+ hops or open chains — use `structuring` below for the canonical AML atom.
- **`fan_out`** (default 168h window, min k=3 targets) — hub → k distinct targets in the window. Structural atom of T6 Offshore Hub, T13 Concentrator / Sink (source side).
- **`fan_in`** (default 168h window, min k=3 sources) — k distinct sources → sink, mirror of `fan_out`. Structural atom of T12 Parallel Layering (destination side) and T13 Concentrator / Sink.
- **`chain_k`** (default 168h window, open directed chain of length k, 3 ≤ k ≤ 8) — A→B→…→Z with no cycle closure, no node revisit, strict monotone timestamps, total span ≤ window. Amount-free open-chain counterpart to `structuring`. Structural atom of T5 Long-Cycle Multi-Stage Layering and T18 Multi-Jurisdiction Latency Chain; tune `k` to the layering depth under investigation (default 4; `k=3` for fast shallow scans, `k≥6` for targeted deep investigations).
- **`structuring`** (default 1h window, amount-gated) — open linear chain A→B→C→D with hop1 amount ≥ `amt1_min` and hops 2 and 3 amount ≤ `amt2_max`, strict temporal ordering. Classic cash-deposit-split-and-wire pattern for evading reporting thresholds. Defaults `amt1_min=10000, amt2_max=10000` match the USD CTR threshold; override per jurisdiction. Empirically this is the dominant `find_motif` typology on IBM AML labelled fraud. Structural atom of structuring / smurfing typology.
- **`split_recombine`** (default 24h window, `min_k` default 3, `direction` parameter `"forward"|"backward"`) — diamond scatter-gather smurfing: source S → k distinct intermediaries {M₁,…,Mₖ} → single sink D, with stacked-bipartite temporal order (all split-hops precede all recombine-hops within the window). Forward mode anchors the seed as the source S; backward mode anchors the seed as the sink D. Amount-free counterpart to `structuring` for the diamond shape. Structural atom of T1 Structured Layering, T12 Parallel Layering (forward), T13 Concentrator / Sink (backward).
- **`bipartite_burst`** (default 24h window, `min_k` default 3 sources, `min_m` default 3 sinks) — complete K_{k,m} bipartite subgraph in a tight time window: k distinct sources each transact with every one of m distinct sinks. Greedy single-core enumeration (not maximal): tries seed-as-source first, falls back to seed-as-sink. Complements `fan_out` + `fan_in` by requiring completeness on both sides rather than just density at a single anchor — flags coordinated mule-ring and parallel-collusion shapes that single-anchor fans miss. The K_{k,m} completeness constraint is why this catches collusion: random anomalous traffic rarely produces a fully-saturated bipartite subgraph in a tight window.

## AML workflow (Phase 2 confirmation)

```
1. find_high_potential_edges(pattern_id, min_pair_count=1)       → single-edge rare-pair candidates
2. For top-K candidates → score_motif(suspect, motif_type="structuring", pattern_id)
3. If structuring is_high_potential → escalate as deposit-split-and-wire (highest-recall AML atom)
4. Otherwise score_motif(suspect, motif_type="cycle_2", pattern_id)
5. If cycle_2 is_high_potential → flash-burst round-trip
6. Otherwise score_motif(suspect, motif_type="cycle_3", pattern_id)   # only if non-IBM-AML domain
7. For hub-candidates (high out_degree) → score_motif(suspect, motif_type="fan_out", pattern_id)
```

## Global motif screening

```
find_high_potential_motifs(pattern_id, motif_type="structuring", top_n=20)        # top deposit-split-and-wire (AML default)
find_high_potential_motifs(pattern_id, motif_type="cycle_3", top_n=20)            # top round-tripping 3-party rings
find_high_potential_motifs(pattern_id, motif_type="fan_out", top_n=20)            # top concentrator hubs
find_high_potential_motifs(pattern_id, motif_type="cycle_2", top_n=20)            # top flash-burst round-trips
find_high_potential_motifs(pattern_id, motif_type="split_recombine", top_n=20)    # top scatter-gather diamonds (forward by default)
find_high_potential_motifs(pattern_id, motif_type="bipartite_burst", top_n=20)    # top K_{k,m} coordinated bursts
```

On AML-shaped spheres start with `structuring` — it is the only `find_motif` branch with material recall lift on IBM AML benchmarks (`cycle_3`, `round_trip`, `pass_through` branches are effectively inactive there because AML fraud cycles are 4+ hops or open chains, not 3-node closed triads). `cycle_2` remains useful as the fast flash-burst prior. `fan_out` captures concentrator hubs. `cycle_3` is retained as a non-AML-default typology match for domains where closed triads appear (crypto wash-trade rings, some fraud typologies outside the IBM AML label universe).

## Structuring thresholds per jurisdiction

Default `amt1_min=10000, amt2_max=10000` match the US Currency Transaction Report (CTR) threshold. Adjust per sphere:

| Jurisdiction | `amt1_min` | `amt2_max` | Rationale |
|---|---|---|---|
| US CTR | 10000 | 10000 | FinCEN 31 CFR 1010.311 |
| UK STR (cash) | 10000 | 10000 | MLR 2017, threshold in GBP |
| EU CTR | 10000 | 10000 | AMLD5 Article 11, threshold in EUR |
| Crypto exchange (typical) | 3000 | 3000 | Varies by exchange — consult risk policy |
| Custom / investigative | user-specified | user-specified | For suspected-pattern hunts with known amount signature |

`amt1_min` and `amt2_max` are part of the ranking LRU cache key, so changing thresholds on the same pattern triggers recompute. Budget one cold call per unique (pattern, threshold pair) in a session.

## Cold-call cost and warm-up order

First call per (pattern, motif_type, window) is cold. Subsequent calls hit an LRU cache (cap 8). Plan exploration with this in mind: pick one motif_type, spend the cold cost, drill down on the top results before switching to a different motif_type.

**Warm-up order for cold-session cost.** The per-pattern adjacency index is shared across eight graph primitives (`find_counterparties`, `entity_flow`, `contagion_score`, `discover_chains`, `anomalous_edges`, `find_geometric_path`, `detect_network_novelty`, and motif ranking). The first of these primitives called on a given pattern pays the full edge-table materialization; every subsequent call that touches the same pattern reuses it. If the investigation starts with `find_anomalies` → `find_counterparties(suspect, pattern_id)` on the top candidates, the adjacency is already warm by the time `find_high_potential_motifs` is called — the motif cold cost drops by roughly half. Conversely, starting with a cold `find_high_potential_motifs` and only later calling `find_counterparties` pays the cold cost on motif ranking first. Either way works; the order only affects which call wears the one-time materialization.

## Scale threshold — when NOT to use `find_high_potential_motifs`

Global motif ranking materializes the full pattern adjacency in memory (roughly 200 bytes / edge in Python tuple overhead). On smaller spheres the default path is fine. Above ~10M edges, memory and cold build time become prohibitive. Switch to the seed-first pattern:

```
find_anomalies(pattern_id, top_n=1000)              # cheap, pre-computed delta_norm
for seed in top_anomalous_seeds:
    score_motif(seed, motif_type="cycle_2", pattern_id)
    score_motif(seed, motif_type="cycle_3", pattern_id)
```

`score_motif` uses a Lance BTREE point query (`read_edges(from_keys=[seed])`) per call — sub-15ms per seed regardless of sphere size, zero full-adjacency materialization. The trade-off: this only finds motifs seeded at already-anomalous entities. For detecting novel motifs whose endpoints are not yet individually anomalous, the global ranking path is required — budget accordingly.

## `motif_potential` enrichment on `trace_root_cause`

`trace_root_cause.edge_counterparty.evidence` automatically includes a `motif_potential` block when the suspect is the seed of a high-scoring motif that passes through the counterparty. This closes a gap — an analyst doesn't need a separate call to find out "is there a triad around this pair". When you see `edge_potential.score` AND `motif_potential.score` both above p95, the suspect-counterparty relationship is both edge-rare (single tx, distant deltas) and structurally-rare (part of a named AML motif). Treat as top priority for Phase 3 escalation.
