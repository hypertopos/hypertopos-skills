# Root-Cause Tools

Detail for the root cause chain and entity 360 steps of the investigator skill.

## trace_root_cause — parameters, signals, sample tree

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
  top_dimensions: [{dim, kind, delta, pct_of_total}, ...]
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
        {"dim": "amount_out_std", "kind": "gaussian", "delta": 13.3, "pct_of_total": 0.28},
        {"dim": "sum_in", "kind": "gaussian", "delta": 10.7, "pct_of_total": 0.18}
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
- Global screening: `find_high_potential_motifs(pattern_id, motif_type="cycle_3", top_n=20)` surfaces the most suspicious triads in the whole pattern. First call per (pattern, motif_type, window, …, k) is cold and can take a while on large patterns — subsequent calls hit the LRU cache.
- Entity drill-down: `score_motif(suspect, motif_type="cycle_2", pattern_id=pattern_id)` checks whether the suspect is the seed of a high-score round-trip without committing to a specific counterparty.

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

## Multi-hypothesis and compliance cross-check

**Multi-hypothesis investigation**: when `explain_anomaly` shows
`reliability_flags.single_dim_driven=True`, call
`find_diverse_explanations(primary_key, pattern_id, n_hypotheses=3)` to
surface alternative hypotheses beyond the dominant dim. Top-anomalous
entities often degrade to 1-2 hypotheses with
`degraded_reason="insufficient_diverse_mass"` — that's correct semantic,
signals the anomaly is genuinely single-dim. When the result returns
2+ hypotheses with `diversity_score > 0.8`, each is an independent
investigation path worth probing.

**Compliance + anomaly cross-check** — `find_conformance_violations(pattern_id)` returns entities that broke declarative rules in `sphere.yaml`. Independent from `delta_norm` anomaly flags; compose via `investigate_entity` on top violators to test "does the rule break also reflect a geometric anomaly?" If yes — confirmed compliance issue with structural backing. If no — rule break alone, may indicate policy gap or genuine compliance violation without geometric signature.

## Root cause chain — tension, topology and counterfactual tools

```
find_graph_geometry_tension(primary_key, pattern_id, line_id) -> behavioural-vs-edge 2x2 cross-tab.
  hidden_cluster = similar-but-disconnected (lookalike cohort never seen together);
  suspicious_links = connected-but-distant (out-of-peer-group). On AML-class data the signal
  sits in suspicious-links count; hidden_cluster saturates at k_geometric.
find_topological_anomalies(pattern_id) -> WHO sits in a local H_1 cycle in delta space.
  Ranks by raw h1_max_persistence (not the auxiliary normalised topo_score). Empirical lift
  is mid-rank — composition input for HMP / passive_scan, NOT a top-N drill-down replacement.
simulate_edge_removal(primary_key, pattern_id, line_id, top_n=5) -> WHICH edges made this entity anomalous.
  Per-edge counterfactual: for each candidate edge in the entity's adjacency, simulate removal and
  rank by drop in delta_norm. Returns (edge_id, drop_pct, dominant_dim_label) per edge. Covers
  relations + edge_dim_aggregations dims (count_above_threshold held constant). Investigator-drilldown
  only, not a population scoring axis.
simulate_counterparty_removal(primary_key, pattern_id, line_id, top_n=5) -> WHICH counterparty
  made this entity anomalous. Counterparty-level counterfactual: aggregates all edges to one
  counterparty per call; ranks counterparties by aggregate delta_norm drop. SAR-friendlier than
  per-edge ("removing transactions with ACC-X drops delta_norm by 38 %") when the edge ranking
  is too granular.
select_minimal_joint_edge_removal(primary_key, pattern_id, line_id, target_drop_pct=50, k_max=8)
  -> SMALLEST edge set whose joint removal drops delta_norm by ≥50% (typically sufficient
  to flip the verdict). Greedy edge-set search; returns edge_ids[] + achieved_drop_pct +
  target_reached. The minimal joint set is the literal "evidence of anomaly" — what you
  cite in the SAR narrative as the cause.
simulate_dimension_change(primary_key, pattern_id, line_id, set_dimension, top_n=5) -> would this
  entity still be anomalous if dim X were at value V. What-if dimension override: override one or
  more raw shape-vector dims, recompute delta_norm under the pattern's calibration, and report
  delta_norm_before/after, the anomaly-flag flip, and the new top witness dims. Companion to
  simulate_edge_removal for non-edge dimensions. set_dimension is {dim_label: new_value} in raw
  shape-vector units — call explain_anomaly first to pick dim_labels and read current values.
```

## Chain-coherent loop tools (entity 360 on spheres with chains)

```
# When the sphere DOES have a chain anchor pattern (built from chain_lines:),
# the chain-coherent investigative loop offers a stronger composition:
find_chains_with_coherent_anomaly(chain_pattern_id, anchor_pattern_id, min_hops=3)
  Population sweep — flag chains where ≥min_hops consecutive entities are
  individually anomalous AND share the same dominant delta dim. Distinct
  axis from find_anomalies on the chain pattern (which scores chain shape).
anomaly_propagation_in_chain(chain_id, chain_pattern_id, anchor_pattern_id)
  Drill into a single flagged chain — returns hop-by-hop anomaly trace
  (is_anomaly + delta_norm + top_dim + delta_rank_pct per hop).
classify_chain_typology(chain_id, ...)
  Five-axis label per chain: shape / peak_position / position_in_chain /
  extension_signals / dominant_top_dim. Triage tag without re-reading hops.
extend_chain(chain_id, ..., direction="forward"|"backward")
  Suggest extension entities at the boundary of the chain's anomalous run
  using the chain reverse index. Anomalous candidates are "where to look
  next" investigation targets.
chain_witness_intersection(chain_id, chain_pattern, member_pattern,
                           min_jaccard=0.5, top_k_witness=5)
  Intersect the top witness dims of the chain's members.
  coordinated=True (mean pairwise Jaccard >= min_jaccard) means every
  member is anomalous for the same structural reason — a single
  geometric diagnosis for the chain.
chain_drift_trajectory(chain_id, chain_pattern, member_pattern, n_windows=4)
  Per-member regime (normalizing / deteriorating / neutral) over
  time-bucketed delta_norm, plus a chain-level rollup
  (mixed when members disagree) + numeric drift score. Spots chains
  jointly drifting toward anomaly before any single hop crosses θ.
find_chains_for_entity(primary_key, chain_pattern_id)
  Reverse lookup — list every chain a given entity participates in
  (deduplicated; cyclic / self-revisiting chains surface once). The
  deep-dive accessor for chain extension candidates returned by
  extend_chain. See gds-fraud-investigator R9 for the full
  flag→trace→label→cross-check→extend→deep-dive workflow.
```
