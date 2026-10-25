# Recipe Index

Which investigation tool fits which pattern type, and what each edge-table recipe operates on.

## Per-Pattern Investigation Guide

| Pattern type | Key signal | Investigation tool |
|---|---|---|
| FAN-OUT | high out_degree + many dest_banks | find_counterparties → check all targets; `edge_stats` confirms degree distribution |
| FAN-IN | high in_sources | find_counterparties → who feeds this account?; `edge_stats` for inbound concentration |
| CYCLE | pair anomalies + chain is_cyclic | `extract_chains(pattern, seed_nodes=[suspect], bidirectional=True)` → filter cyclic; `find_geometric_path(scoring="anomaly")` traces the ring |
| STACK | delta_rank_pct 90-95 (borderline) | `extract_chains(pattern, seed_nodes=[suspect], min_hops=3)` → check chain membership at runtime |
| BIPARTITE | pair anomalies + community | aggregate(group_by_property="community_id") |
| RANDOM | chain anomalies | `extract_chains(pattern, seed_nodes=[suspect], max_hops=6)` → longest reachable chains |
| BRIDGE | entity straddles two communities | `cluster_bridges(pattern_id)` → bridges with anomalous status on both sides are high-risk connectors |

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

- **External-chain workflow applies only to R9** — R9 is the only recipe whose input is a chain. R1-R8 / R10-R11 take accounts / pairs / motifs / clusters — chain identifiers don't enter their signature. To unlock external-chain ingestion for them, would need composite primitives (e.g. "R3 burst scoped to chain members", "R8 witness cohort seeded from chain hops"), which do not exist — see [external-chains-as-anchor-line.md](https://github.com/hypertopos/hypertopos-py/blob/main/docs/external-chains-as-anchor-line.md) for the R9 path.
- **Edge-free recipes are R5 / R7 / R9** — R5 reads only anchor delta features, R7 reads only the amount distribution column, R9 reads chain anchor pattern + member anchor pattern. The other 8 either scan the edge table runtime (R1, R3, R4, R10) or read derived edge-features baked into the anchor at build time (R2, R6, R8, R11).
- **Only R9 has end-to-end orchestration + SAR narrative** — `investigate_chain` aggregates the four R9 primitives into one server-side call and `generate_sar_rationale` composes the resulting evidence into a paragraph-structured draft. Other recipes are manual call sequences; for a single account, `investigate_entity` is the closest one-call composition.
