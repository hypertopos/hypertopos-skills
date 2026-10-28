# Phase 2 Signals and Tuning

Detail behind the screening and Phase 2 investigation steps of the fraud investigator skill.

### Alarm budget and diverse selection

On a calibrated pattern (non-empty `score_dims`), `is_anomaly` means `conformal_p ≤ 1 − anomaly_percentile/100` — the alarm budget: at the 95th percentile about 5 % of a clean population is flagged. False positives waste investigator time and erode trust in the alert pipeline, so size the screening queue to that load: work it in `anomaly_score` order (the `find_anomalies` default) and send the smallest `conformal_p` to investigators first. There is no entity-level FDR to switch on. When requesting K>10 results, use `select="diverse"` to surface different fraud typologies (structuring, layering, round-tripping) instead of 50 variants of the same high-volume pattern; this leverages submodular facility location to maximize typological coverage in the result set. `select` also applies to `attract_boundary`, `find_hubs` and `find_drifting_entities`.

**Where alerts concentrate — the cell gate.** When the pattern declares `fdr_hierarchy:` (e.g. bank, region) or `fdr_temporal_hierarchy:` (e.g. week), `find_anomalies(..., fdr_alpha=0.05, fdr_resolution="<level>")` keeps the cells enriched in flagged accounts (hypergeometric test per cell, Storey over cells by default) — the evidence for "this bank is the hot-spot" in a SAR narrative (`cell_q_spatial`, `cell_path` on each survivor).

**Counterfactual suite (4 tools).** Different questions, different aggregation level.

- `simulate_edge_removal(suspect_pk, "<anchor_pattern>", line_id="<event_pattern>", top_n=5)` — "which transactions made this entity anomalous?". Ranks the entity's edges by contribution to `delta_norm`. For each edge: `(delta_norm_before, delta_norm_after, drop_pct, dominant_dim_label)`. Covers `relations` and `edge_dim_aggregations` dims; the `count_above_threshold` aggregation is held constant (reported in `dimensions_skipped`), and `event_dimensions` / `prop_columns` carry no per-edge contribution by design.
- `simulate_counterparty_removal(suspect_pk, "<anchor_pattern>", line_id="<event_pattern>", top_n=5)` — "which counterparty made this entity anomalous?". Aggregates all edges to one counterparty per call; ranks counterparties by aggregate `delta_norm` drop. Use when the edge-level ranking is too granular and you want a counterparty-level SAR narrative ("removing transactions with ACC-X drops delta_norm by 38 %").
- `select_minimal_joint_edge_removal(suspect_pk, "<anchor_pattern>", line_id="<event_pattern>", target_drop_pct=50, k_max=8)` — "smallest set of edges whose joint removal drops delta_norm by ≥50% (typically sufficient to flip the anomaly verdict)". Greedy edge-set search; returns `edge_ids[]` + `achieved_drop_pct` + `target_reached` flag. SAR composition: the smallest joint set is the literal "evidence of laundering" that explains why this entity crosses θ.
- `simulate_dimension_change(suspect_pk, "<anchor_pattern>", line_id="<event_pattern>", set_dimension={dim_label: new_value}, top_n=5)` — "would this entity still be anomalous if dim X were value V?". Override one or more raw shape-vector dims, recompute `delta_norm` under the pattern's calibration; reports `delta_norm_before/after`, the anomaly-flag flip, and new top witness dims. Companion to `simulate_edge_removal` for non-edge dimensions. Call `explain_anomaly` first to pick dim_labels and read current raw values.

**Fraud-specific tuning.**
- **`edge_counterparty_top_n=3..5`** — mule networks typically involve multiple anomalous counterparties; default 1 shows only the single most anomalous cp. For Phase 3 escalation on high-risk suspects, raise to expand each anomalous cp as a separate subtree.
- **`branches_enabled=["neighbor_contamination"]`** — fast targeted scan: for a batch of candidate suspects, contagion-only trace avoids the edge_counterparty recursion and π7 scan; orders of magnitude cheaper than default once cache is warm. Use to rank-by-contagion before committing to full traces on the top-N.
- **`contagion_min_counterparties=5`** — raise default 3 for high-confidence fraud networks; filters out statistical noise from small-counterparty suspects.

**Clique detection signals (read these on every trace).**
- **`revisits_root: [keys]`** on a contagion branch = confirmed 2+ node clique with the root. Every name in this list transacted with the root AND is itself anomalous. Immediate Phase 3 escalation criterion.
- **`previously_seen_as_cp_of: [X, Y]`** = session ledger signal: other suspects you already traced (X, Y) listed THIS entity as their anomalous cp. The entity is a shared counterparty across multiple confirmed suspects — high-value graph-centre node worth prioritising.
- **`anomalous_cp_keys`** list → all 10+ anomalous neighbours available without a second `find_counterparties` call. Use directly as Phase 3 investigation queue.

**Edge-level rare-pair detection — `find_high_potential_edges`.** AML layering often shows as a one-off transaction between two geometrically divergent accounts (high-income sender, dormant/low-history receiver). Node-level delta_norm misses it because each endpoint alone looks only borderline. `find_high_potential_edges(pattern_id, top_n=20, min_pair_count=1)` catches these by scoring the edge itself.

- Use early in Phase 2 — run AFTER `find_anomalies` on the anchor pattern but BEFORE committing a full `trace_root_cause` on each candidate. High-edge-potential edges point directly at suspect pairs without requiring a candidate selection.
- Cross-reference with `trace_root_cause.edge_counterparty.edge_potential` — if the same edge appears in both (top of find_high_potential AND enriched evidence on trace), that's a confirmed rare-pair layering signal worth immediate Phase 3 escalation.
- Raise `min_pair_count=3` on spheres where legitimate frequent-but-large pairs (corporate payroll) dominate the top — shifts focus to truly one-off suspicious edges.

**Dual-query playbook — singletons AND recurring suspects.** The default ranking with `min_pair_count=1` is dominated by one-off singleton pairs (classic placement/layering signature). But recurring suspect pairs (a handful of transactions between two anomalous accounts) are a DIFFERENT fraud shape — e.g. structuring below reporting thresholds over several days. A single ranking call catches one shape; run BOTH:

```
# Pass 1: singletons (placement / one-off layering)
find_high_potential_edges(pattern_id, top_n=20, min_pair_count=1)

# Pass 2: recurring suspects (structuring / smurfing)
find_high_potential_edges(pattern_id, top_n=10, min_pair_count=5)
```

The second pass returns pairs with 5+ transactions between the same two anomalous accounts — that pattern rarely appears in legitimate business and is a strong smurfing signal. Use both outputs as two parallel Phase 3 queues. Each `is_high_potential=true` entry is above the pattern's p95 of scores and warrants immediate review.

**Reading the `is_high_potential` flag + the rank percentile.** The scalar `score` is not cross-sphere-comparable (it scales with delta dimensionality), but every result carries a percentile within the pattern — `score_rank_pct_global` on `find_high_potential_edges` (plus `score_rank_pct_in_filter` when `from_key` / `to_key` / `min_pair_count` narrow the ranking), `score_rank_pct` on motif results — and `is_high_potential` (boolean, true when score ≥ p95). `score_edge` fills both only with `include_ranking=True`. Use these for triage: `is_high_potential=true` is the actionable flag; a percentile `>= 99` marks the extreme tail for immediate escalation.

**Structural motif detection — `score_motif` and `find_high_potential_motifs`.** Where `edge_potential` scores one anomalous edge, `score_motif` scores the whole structural shape of a suspicious k-edge motif. The scoring rule is product-of-edge_potential across motif edges — a motif survives only if ALL its edges are rare and its endpoints are geometrically distant. One regular high-volume edge in a triad collapses the score to near-zero (correct: a triad with a payroll edge is not laundering).

Eight closed motif types (`cycle_2`, `cycle_3`, `fan_out`, `fan_in`, `chain_k`, `structuring`, `split_recombine`, `bipartite_burst`) map to the 25 typologies in [`typologies.md`](typologies.md). On AML-shaped spheres start with `structuring` (usually the highest-recall motif on AML data, where closed-triad `cycle_3` rarely fires). `cycle_2` for flash-burst priors. `fan_out` for concentrator hubs.

Full motif vocabulary specs (defaults, temporal-ordering rules, structural-atom mappings to typologies T1–T19) + AML Phase 2 workflow + jurisdictional structuring thresholds (US CTR / UK STR / EU CTR / crypto) + cold-call cost / warm-up order / scale threshold (>10M edges → seed-first pattern with `score_motif`) + `motif_potential` enrichment on `trace_root_cause` are documented in [`motif-detection.md`](motif-detection.md).
