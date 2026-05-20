# Changelog

All notable changes to `hypertopos-skills` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.7.0] — 2026-05-18

### Added

#### Multi-hypothesis explanation + counterfactual cheatsheets
- `gds-investigator` and `gds-fraud-investigator` — `find_diverse_explanations` block added next to `explain_anomaly` guidance; covers when to invoke and how to read `degraded_reason="insufficient_diverse_mass"`.
- `gds-investigator` and `gds-fraud-investigator` — counterfactual cheatsheet covers the full four-tool suite: `simulate_edge_removal`, `simulate_counterparty_removal`, `select_minimal_joint_edge_removal`, and `simulate_dimension_change` — with per-tool guidance on edge-level / counterparty-level / minimal-joint-set / what-if-dimension questions.
- `gds-analyst` — root-cause section gains the `investigate_entity` one-call orchestrator entry point, the `reliability_flags` soft-hit gate (single_dim_driven / low_confidence_bucket), `find_diverse_explanations` for single-dim-driven cases, and the full counterfactual suite (4 tools).

#### Chain extensions
- `gds-fraud-investigator` — R9 recipe gains `chain_witness_intersection` and `chain_drift_trajectory` diagnostic steps; mirrored in `references/recipes.md`.
- `gds-investigator` — chain-anchor cheatsheet extended with `chain_witness_intersection` and `chain_drift_trajectory`.

#### Declarative compliance rules
- `gds-fraud-investigator` and `gds-investigator` — `find_conformance_violations` block added covering the rule-break + geometric-anomaly cross-check workflow.
- `gds-sphere-designer` — new "Declarative compliance rules (`conformance_rules:`)" section with the predicate AST schema (logical `and`/`or`/`not` over `==`/`!=`/`<`/`<=`/`>`/`>=`/`in`), `rule_id` + `severity` semantics, sample YAML with two rules, and the build-time materialization path (`_gds_meta/conformance/violations/{pattern_id}/v={N}.lance` + `rule_set_hash` invalidation).
- `gds-analyst` — hint-category table gains a "Compliance rule break" row pointing at `find_conformance_violations` with the `investigate_entity` cross-check workflow.

#### FDR cheatsheets
- `gds-detective` — "Multi-resolution FDR: spatial × temporal hierarchies" section covering `fdr_resolution` / `fdr_temporal_resolution` on `find_anomalies` with spatial-only, temporal-only, and intersection examples.
- `gds-detective` — "FDR axis selection" and "Ranking by per-dim significance" paragraphs covering `fdr_axis` and `rank_by="min_q_per_dim"` semantics on `find_anomalies`.
- `gds-scanner` — FDR-control section extended with multi-resolution FDR (`fdr_resolution` / `fdr_temporal_resolution`), per-dim FDR axis (`fdr_axis="per_dim"` / `"both"`), and `rank_by="min_q_per_dim"` re-ranking guidance.
- `gds-analyst` — FDR control paragraph added covering `fdr_alpha` baseline plus a pointer at multi-resolution and per-dim FDR with cross-link to `gds-detective` for the full cheatsheet.

#### Detector composition
- `gds-detective` and `gds-investigator` — multi-pattern section gains `combine_anomaly_pvalues([(pattern_id, p), ...], method="hmp"|"fisher")` for arbitrary-detector p-value composition and `classify_detector_consensus` for unanimous / majority / split / unanimous_normal labeling.
- `gds-analyst` — hint-category table gains a "Detector composition" row routing at `combine_anomaly_pvalues` and `classify_detector_consensus`.

#### Topological / graph-geometry scans
- `gds-scanner` — relevant-scan table gains a row for `find_topological_anomalies(pattern, top_n=20, k_neighbors=100)` with the empirical-lift caveat (mid-rank, NOT a top-N drill-down replacement; use as composition input for HMP / `passive_scan`).
- `gds-analyst` — hint-category table gains rows for `find_topological_anomalies` (local H_1 cycle persistence) and `find_graph_geometry_tension` (behavioural-vs-edge cross-tab).

#### Sphere-validation cheap-tier
- `gds-monitor` and `gds-investigator` — `sphere_overview` notes extended to cover the new `dim_quality_warnings` types `dominant_dim_mass` and `negative_space`.
- `gds-explorer` — orientation `dim_quality_warnings` bullet names the four auditors (`dead_dim`, `sparse_dim`, `dominant_dim_mass`, `negative_space`) with calibration-ticket guidance for the two pattern-level signals.

#### Reliability triage
- `gds-scanner` — FDR-control section extended with `reliability_flags` soft-hit gate (single_dim_driven / low_confidence_bucket) before feeding into cross-pattern or contamination analysis.

### Changed
- `gds-analyst` — body refreshed against the 0.6.x chain investigation surface and the full 0.7.0 detection upgrades; `metadata.version` bumped from `0.2.2`.
- `gds-scanner` — body refreshed with multi-resolution FDR, per-dim FDR, `find_topological_anomalies`, and `reliability_flags` filter; `metadata.version` bumped from `0.5.0`.
- `gds-analyst` / `gds-detective` / `gds-sphere-designer` — cross-pattern composition references refreshed from Fisher's method to the Wilson harmonic-mean p-value (HMP).

## [0.6.7] — 2026-05-10

### Added
- `gds-monitor` — new "Dim-quality warnings" cheatsheet section covering the two silent build-time failure modes surfaced via `sphere_overview().dim_quality_warnings[]`: `dead_dim` (sigma_diag below 1e-10 — z-score undefined) and `sparse_dim` (median == 0 with p99 > 0 — gaussian assumption wrong). Documents the `{type, dim_label, reason, advice}` structure and triage rules per failure class: drop / fix data source for dead, switch to Bregman or split into is_active + value_when_active for sparse.

### Changed
- `gds-fraud-investigator` — SKILL.md split into reference files per skill best practice (auto-loaded SKILL.md should be small; deep cheatsheets / per-recipe detail load on demand). SKILL.md trimmed from 1082 lines to 457 lines (-58%); seven new reference files extracted to `references/`:
  - `references/recipes.md` — full per-recipe detail (R1-R11): tool sequences, interpretation, false-positive guards, uniqueness rationale. SKILL.md keeps the recipe summary table + the input/capability matrix + a 1-line "Pattern" sentence per recipe.
  - `references/declarative-motifs.md` — `find_motif_by_hops` patterns: declarative structuring chains (`amount_ratio_to_prev`), long-chain layering (`time_window_hours`), per-hop anomalous-intermediary filtering (`require_anomalous_entity`).
  - `references/edge-aggregations.md` — account-level + chain-level recall via `edge_dim_aggregations:`; per-aggregate semantic, how to read on a suspect, anchor regime coverage.
  - `references/correlation-gates.md` — feature curation via stratified correlation gates (three-lens approach: all-population, per-bucket, partial r) + wiring gate verdicts into `find_anomalies(dimension_weights={...})` runtime ranking.
  - `references/advanced-workflows.md` — four canonical operational scenarios with full per-step Python templates: detect coordinated population-shift attack (`find_group_influence` reinforcing-factor test), AML hidden-influencer triage → SAR candidate workflow, cross-pattern lead-lag for AML rapid-escalation, edge-derived dimensions for AML detection (`edge_dimensions:` declaration with five build-time per-edge dim functions).
  - `references/examples-and-troubleshooting.md` — three worked end-to-end examples (full AML screening, typology detection, false-positive elimination) + canonical troubleshooting tree (`passive_scan` not available, `extract_chains` empty / times out, source_count=1 collapse, `find_counterparties` hub explosion).
  - `references/motif-detection.md` — closed motif vocabulary (`cycle_2`, `cycle_3`, `fan_out`, `fan_in`, `chain_k`, `structuring`, `split_recombine`, `bipartite_burst`) mapped to typology atoms, AML Phase 2 confirmation workflow, jurisdictional structuring thresholds (US CTR / UK STR / EU CTR / crypto), cold-call cost / warm-up order / scale threshold (>10M edges → seed-first pattern), `motif_potential` enrichment on `trace_root_cause`.

  Each extraction leaves a 1-3 sentence summary in SKILL.md with a cross-link, so agents auto-loading the skill see the entry points and load the full reference on demand. No content removed; the reference files preserve the original prose. SKILL.md size now in line with sibling skills (gds-detective 440, gds-analyst 389) — pattern matches `gds-investigator` / `gds-sphere-designer` / other siblings which separate the auto-loaded entry from on-demand deep-dive material.

### Added
- `gds-fraud-investigator` — new "Recipe input + capability matrix" cheatsheet block immediately after the existing per-recipe summary table. Quick-reference table covering recipe × {input type, edge-table use, one-shot orchestrator, SAR narrative composer} for all 11 recipes (R1-R11). The matrix surfaces three structural patterns directly: (a) external-chain workflow applies only to R9 because R9 is the only recipe whose input is a chain; (b) the edge-free recipes are R5/R7/R9 and the rest either scan the edge table runtime (R1, R3, R4, R10) or read derived edge-features baked at build time (R2, R6, R8, R11); (c) only R9 has end-to-end orchestration (`investigate_chain`) and SAR narrative composer (`generate_sar_rationale`) today, with per-recipe `investigate_<recipe>` orchestrators and SAR templates as the natural extensions. Existing per-recipe table also gains R10 + R11 rows that were missing from the summary.

### Added
- `gds-fraud-investigator` — R9 per-chain shortcut path gains a "SAR narrative draft" step documenting the new `generate_sar_rationale` MCP tool. Closes the investigation→SAR pipeline: investigator gets a 3-5 paragraph starting draft from R9 evidence (no LLM call; pure template composition) instead of writing from a blank page after `investigate_chain` completes. Documents the `evidence` passthrough (cheap path for repeated narratives on the same chain), the `regulatory_template` hint passthrough (FinCEN SAR / EU AMLR Annex II / internal template names), and the `confidence` derivation (`high` requires `strong` strength AND all 5 R9 surfaces ok). Honesty discipline call-out: narrative uses "evidence indicates" / "the per-hop trace shows" — never "confirms" — and is a starting draft, not a final verdict.

### Changed
- `gds-fraud-investigator` — corrected the "Anchor regimes supported" note in the chain-coherent loop section. Previously read "External chain imports (user-loaded membership) are not supported" — wrong: external chains DO work as anchor lines, and the R9 loop primitives operate on them when the `chain_keys` convention column is populated. The note now describes both BFS-extracted (`chain_lines:`) and externally-ingested chains, with a cross-link to the new `packages/hypertopos-py/docs/external-chains-as-anchor-line.md` cookbook. Recipe R9 also gains an "External chain source" paragraph documenting the workflow for SAR typology engines / ERP / EHR / customer-journey ingestion.

### Added
- `gds-fraud-investigator` — "Wiring gate verdicts into runtime ranking" cheatsheet block in the Feature curation via stratified correlation gates section. Documents how to translate gate verdicts (NOISE / DIRECTION-INCONSISTENT / VOLUME-MEDIATED / ROBUST) into the new `find_anomalies(dimension_weights={...})` parameter so the Theme E findings shape the ranked output an investigator actually sees, not just the SAR rationale narrative.
- `gds-fraud-investigator` — R9 loop gains an explicit triage step (step 0) using the new `chain_investigation_summary` MCP tool. Documents triage rules: `coherent_run_rate < 0.005` AND low cross-pattern jaccard → skip the deep loop, fall back to chain-shape anomalies; `coherent_run_rate > 0.05` → proceed; `recommended_min_hops > 3` → use the recommended threshold in step 1 to focus on the strongest cases.
- `gds-fraud-investigator` — R9 loop gains a per-chain shortcut block documenting the new `investigate_chain` one-shot orchestrator MCP tool. After step 1 surfaces a target chain_id, `investigate_chain` runs steps 2-5 (trace + typology + shape-anomaly lookup + extension forward + extension backward) in a single call and returns the aggregated report with a SAR-ready summary block (`investigation_strength`, `recommended_action`, `rationale`). The per-step granular tools remain available when finer per-step control is needed.

## [0.6.6] — 2026-05-09

### Removed
- `gds-fraud-investigator` — R12 (Cuckoo-smurfing / account hijacking) recipe removed from `SKILL.md` and the `references/typologies.md` cross-ref note. The recipe was shipped as `Status: Exploratory` and is retired rather than kept under that tag without supporting empirical evidence. R10 (TBM) and R11 (Mule network discovery) remain in `SKILL.md`.

### Added
- `gds-monitor` — new "Threshold sensitivity at a glance" cheatsheet section covering how to read the `theta_sensitivity_summary` block on each `sphere_overview` entry. Documents the four exposed fields (`stable_band_from`, `stable_band_to`, `stable_band_length`, `n_cliffs`, `theta_at_p95`) and triage rules for monitoring workflows: `stable_band_length >= 8` with `n_cliffs == 0` is a smooth pattern, `stable_band_length <= 4` or `n_cliffs >= 2` triggers a drill-down via the `theta_sensitivity(pattern_id)` tool, and a p95 boundary that participates in a cliff pair signals a recalibration risk that should not be moved upward.
- `gds-fraud-investigator` — three new typology recipes after R9 covering domain-specific laundering patterns: R10 (Trade-based money laundering — multi-currency cross-bank flows with amount mismatches), R11 (Mule network discovery — density-based clusters of low-volume coordinated accounts), R12 (Cuckoo-smurfing — account hijacking detected via neighbor contamination + velocity burst + currency diversity change). Each recipe documents Pattern, Tool sequence, Score formula, Interpretation, False positive guard, and Why-unique-vs-other-recipes; each carries an explicit `Status: Exploratory` tag with guidance for investigators to treat output as candidate sets, not verdicts. Cross-ref note added to `references/typologies.md` pointing to the R-numbered family in `SKILL.md` as the canonical entry point.

### Changed
- `gds-fraud-investigator` — added cheatsheet section "Feature curation via stratified correlation gates" covering the three-lens approach (all-population, stratified per-bucket, partial point-biserial correlation) for sanity-checking whether a chain or account feature carries laundering signal independent of the natural confounder (chain length / account transaction volume). Documents the four cross-bucket verdicts (ROBUST / DIRECTION-INCONSISTENT / LENGTH-MEDIATED / VOLUME-MEDIATED / NOISE), when to use each lens during investigation, how to interpret verdicts in a SAR / triage context, and the heterogeneous-label hypothesis for cases where multiple features show direction-inconsistency.
- `gds-fraud-investigator` — Recipe R9 (Chain-Coherent Cascade) gains "Agent-friendly query forms via `detect_pattern`" subsection. Six-row table maps each step of the R9 loop (Flag / Trace / Label / Extend / Deep-dive) to the natural-language form the `detect_pattern` smart-mode router recognises. Agents don't need to remember primitive names — chain anchor pattern + entity anchor pattern + `CHAIN-<digits>` tokens are auto-extracted from the query. Includes a fallback note explaining keyword-matching behaviour when LLM sampling is unavailable.

## [0.6.5] — 2026-05-08

### Changed
- `gds-investigator` — chain-coherent palette extended with `find_chains_for_entity` as the deep-dive accessor: reverse lookup that lists every chain a given entity participates in (deduplicated; cyclic / self-revisiting chains surface once per chain). Closes the cross-link to the `gds-fraud-investigator` R9 step 6 (per-candidate deep-dive).

## [0.6.4] — 2026-05-07

### Changed
- `gds-fraud-investigator` — Recipe R9 (Chain-Coherent Cascade) tool sequence expanded into the full six-step investigative loop: flag → trace → label → cross-check → extend → deep-dive. Documents the composition of `find_chains_with_coherent_anomaly` (population sweep), `anomaly_propagation_in_chain` (per-chain hop trace), `classify_chain_typology` (five-axis triage tag), `find_anomalies` cross-check on the chain pattern, `extend_chain` (boundary entity widening), and standard Phase 2 deep-dive on candidate entities.
- `gds-investigator` — tool palette gains a cross-link to the chain-coherent investigative loop primitives when the sphere has a chain anchor pattern. Defers the full workflow to `gds-fraud-investigator` R9 to avoid duplication.

## [0.6.3] — 2026-05-06

### Changed
- `gds-fraud-investigator` — account-level recall and chain-level recall
  sections updated to document the five canonical aggregates baked per
  source dim (`_mean` / `_max` / `_std` / `_p95` / `_count_above_threshold`)
  and the new mapping-form `dims:` selector for emitting a per-dim
  subset. The anchor regimes line is corrected — chain anchors and k>2
  composite anchors are supported.
- `gds-monitor` — calibration drift cheatsheet adds a paragraph on the
  new `edge_dim_threshold_drift` field returned by `compare_calibrations`,
  and how to read it alongside `overall_drift_rms` when an anchor
  pattern declares `edge_dim_aggregations:`.

## [0.6.2] — 2026-05-05

### Changed
- `gds-fraud-investigator` — added cheatsheet entry "Chain-level recall
  via aggregated edge dims" covering the new chain-anchor
  `edge_dim_aggregations:` regime (third anchor_kind alongside single
  and pair), how to declare it inside `chain_lines:`, and how to read
  the resulting `_mean` / `_max` aggregates on a suspicious chain.

## [0.6.1] — 2026-05-01

### Changed
- `gds-fraud-investigator` — added cheatsheet entry "Account-level recall
  via aggregated edge dims" covering the new anchor-pattern
  `edge_dim_aggregations:` YAML block (per-edge sidecar dims rolled up
  to per-anchor `_mean` / `_max` columns), the supported anchor regimes
  (`single`, `pair`), and how to read the resulting aggregates on a
  suspicious account (`pair_edge_count_max`, `find_motif_structuring_mean`,
  `position_in_chain_max`, `time_since_pair_last_edge_mean`).
- `gds-fraud-investigator` — added cheatsheet entry "Declarative
  structuring chain detection" for the new
  `HopPredicate.amount_ratio_to_prev` field on `find_motif_by_hops`,
  including the canonical deposit → split → wire example, the
  first-hop validation rule, and when to prefer this over the
  closed-vocab `find_motif_structuring`.
- `gds-fraud-investigator` — added cheatsheet entry "Long-chain
  layering with a global time window" for the new top-level
  `time_window_hours` parameter on `find_motif_by_hops` (independent
  global cap, layered with per-hop `time_delta_max_hours`); explicitly
  documents the 1..8 hop count cap matching the `chain_k` motif
  vocabulary.
- `gds-fraud-investigator` — added cheatsheet entry "Filter chains by
  anomalous intermediaries" for the new
  `HopPredicate.require_anomalous_entity` field on `find_motif_by_hops`
  (per-hop bool filtering by anchor companion `is_anomaly=True`),
  including the seed-not-checked rule and the
  `max_results`-applies-after-filter semantic. Closes the X1
  declarative predicate set.

## [0.6.0] — 2026-04-30

### Changed
- `gds-investigator` — added cheatsheet entry "Anomaly by absence — `find_density_gaps`"
  covering YAML-free invocation, gap interpretation, manual TP-verification,
  and `excluded_dims` behaviour.
- `gds-fraud-investigator` — added cheatsheet entry "Edge-derived
  dimensions for AML detection": YAML `edge_dimensions:` block on event
  patterns lifts transaction-level recall via five build-time per-edge
  dim functions (`pair_edge_count`, `position_in_chain`,
  `time_since_pair_last_edge`, `pair_amount_zscore` LOW_VAR,
  `find_motif_structuring`). Reads off `find_anomalies` polygon `delta`
  and `dimension_kinds` fields directly; no new MCP tool surface.
  metadata.version bumped.
- `gds-monitor` — added cheatsheet entry for `compare_calibrations`
  (0.6.0 multi-epoch calibration retention).
- `gds-investigator` — added cheatsheet entry for `decompose_drift`
  (0.6.0 multi-epoch calibration retention).
- `gds-investigator` — added cheatsheet entry "Find hidden influencers —
  entities defining what 'normal' means" (hidden-influencer matrix).
  metadata.version bumped.
- `gds-fraud-investigator` — added cheatsheet entries "Detect coordinated
  population-shift attack" via `find_group_influence` (collusion-ring
  detection via `reinforcing_factor > 1.5`) AND "AML hidden-influencer
  triage → SAR candidate workflow" (3-step: surface candidates near θ →
  match AML adversarial-typology atoms → group-test for ring escalation).
  metadata.version bumped.
- `gds-monitor` — added cross-link to `find_calibration_influencers` in the
  drift / recalibration section ("if alerts say population shifted unexpectedly,
  find_calibration_influencers helps explain WHY"). metadata.version bumped.
- `gds-investigator` — added cheatsheet entry "Cross-pattern lead-lag" for
  `find_lead_lag`: population-aggregated centroid drift
  cross-correlation between two anchor patterns sharing entity space; how to
  read `agreement` / `is_significant` / `degenerate_signal` / `reliability`
  fields; per-dim drill-down via `top_dim_pairs`; per-entity mode through
  `entity_key`. metadata.version bumped to 0.6.0.
- `gds-fraud-investigator` — added cheatsheet entry "Cross-pattern lead-lag for
  AML rapid-escalation": explicit caveat that AML HI/LI-small spheres at
  `window=2d` give `N=9` epochs with `reliability="low"` and high
  `max_corr_threshold`; M5 raises empty-cohort on `cohort="fixed"` when
  patterns observe disjoint entity_lines (account vs chain), so the workflow
  needs an account-level second anchor pattern (sphere rebuild) to be
  agent-actionable. metadata.version bumped to 0.6.0.

## [0.5.2] — 2026-04-28

### Changed
- `gds-fraud-investigator` SKILL.md and `references/typologies.md`: motif list gains `split_recombine` (scatter-gather smurfing, forward + backward modes) and `bipartite_burst` (complete K_{k,m} coordinated burst). Fast-path typology notes added for the affected typology atoms.
- `gds-investigator` SKILL.md: structural motif section expanded from 6 to 8 types; cheatsheet documents the `direction` parameter (`split_recombine`) and `min_m` parameter (`bipartite_burst`).
- No skill content changes. Hypertopos perf fixes (single-seed via_adj delegation across the full motif catalog, batched endpoint deltas in `_score_motif_from_edges`, small-set-first K-set intersection in `bipartite_burst`) reduce score_motif and trace_root_cause latency observed by the gds-fraud-investigator and gds-investigator skills.
- No skill content changes. Hypertopos cycle_3 enumerator pre-filter reduces score_motif latency on hub seeds observed by gds-fraud-investigator and gds-investigator skills.
- No skill content changes. Hypertopos chain_k adaptive frontier cap reduces find_high_potential_motifs cold latency observed by gds-fraud-investigator and gds-investigator skills at k>=5.

## [0.5.1] — 2026-04-21

### Changed
- `gds-fraud-investigator` SKILL.md: closed-vocabulary motif list gains `fan_in` (mirror of `fan_out`, sink-centric) and `chain_k` (open directed chain of parametric length 3 ≤ k ≤ 8). Metadata version bumped.
- `gds-fraud-investigator/references/typologies.md`: T5 (Long-Cycle Multi-Stage Layering), T13 (Concentrator/Sink), and T18 (Multi-Jurisdiction Latency Chain) recipes gain an atomic fast-path note pointing to the new `chain_k` and `fan_in` motif types. The existing multi-step `extract_chains` / `passive_scan` paths remain authoritative; the motif types add a single-call alternative for agents doing targeted detection.
- `gds-investigator` SKILL.md: structural motif section expanded from three types (`cycle_2` / `cycle_3` / `fan_out`) to six with `fan_in`, `chain_k`, and `structuring` rows. Cache-key note extended to mention the new `k` parameter. Metadata version bumped.

## [0.5.0] — 2026-04-19

### Changed
- 4 skills (gds-scanner, gds-investigator, gds-fraud-investigator, gds-detective) gained a guidance paragraph for Storey adaptive FDR — when to pair `fdr_method="storey"` with `p_value_method="chi2"`, why they must be set together, and in which calibration regimes the power recovery actually materialises. Metadata versions bumped to 0.5.0.
- `gds-fraud-investigator` now documents the adjacency warm-up order for optimal motif cold-call latency and a scale-threshold rule: on spheres where `edge_count > 10M`, skip `find_high_potential_motifs` and drive structural checks through the seed-first `find_anomalies` → per-seed `score_motif` pattern instead. The threshold reflects the underlying in-memory adjacency materialization cost. (metadata stays at 0.5.0 — same release cycle as the Storey-FDR bump.)
- `gds-fraud-investigator` recipe update for the new `structuring` motif in 0.5.0 find_motif vocabulary — open A→B→C→D amount-gated chain, default 1h window. Phase 2 workflow reordered to try `structuring` first on AML-shaped domains (the closed-triad `cycle_3` atom is effectively inactive on IBM AML labelled fraud; structuring is the dominant find_motif typology there). Adds per-jurisdiction `amt1_min`/`amt2_max` threshold table (US CTR / UK STR / EU CTR / crypto) and a note that thresholds are part of the LRU cache key. Smart-mode triggers extended with `"structuring"`, `"smurfing"`, `"split transfer"`, `"deposit split"`, `"reporting threshold"`.
- gds-monitor and gds-investigator: new guidance paragraphs on `drift_direction` triage (prioritise `"deteriorating"`, deprioritise `"normalizing"`). gds-monitor metadata version bumped to 0.5.0.
- gds-investigator and gds-fraud-investigator: new guidance paragraphs on `trace_root_cause` — the one-call DAG replacement for the manual `explain_anomaly → find_counterparties → contagion_score → π7 hub` chain.
- gds-investigator and gds-fraud-investigator: parameter cheatsheet + clique interpretation guide for `trace_root_cause` after quality pass — documents `edge_counterparty_top_n`, `branches_enabled`, `contagion_min_counterparties`, `hub_pop_limit` tuning; explains `revisits_root` / `previously_seen_as_cp_of` / `anomalous_cp_keys` evidence fields; session cache behaviour on repeat traces.
- gds-investigator: new section on `edge_potential` + `find_high_potential_edges` — how to read the edge_potential evidence field on `trace_root_cause.edge_counterparty` branches and when to use the ranking primitive.
- gds-fraud-investigator: AML-specific guidance on rare-pair detection via `find_high_potential_edges` — catches layering signals node-level delta_norm misses.
- gds-investigator and gds-fraud-investigator: new guidance on `score_motif` + `find_high_potential_motifs` — the structural pattern primitives that compose edge_potential across k edges. Explains the closed vocabulary (`fan_out`, `cycle_2`, `cycle_3`), default time windows per motif_type, how to read the `motif_potential` block now auto-attached to `trace_root_cause.edge_counterparty` evidence, and which AML typologies map to each motif structural atom (T2/T4 → cycle_2, T3/T5/T11 → cycle_3, T6/T13 → fan_out).

## [0.4.1] — 2026-04-16

### Fixed
- gds-explorer: `find_geometric_path` description corrected (beam search → bidirectional BFS).

## [0.4.0] — 2026-04-15

### Changed
- 6 skills updated for Bregman divergence, anomaly_confidence, and min_confidence.
- gds-investigator/fraud-investigator: confidence-integrated triage and Entity 360.
- gds-sphere-designer: kind tag guidance, bootstrap_iterations tuning, continuous-mode warnings, dirty timestamp troubleshooting.

## [0.3.3] — 2026-04-13

### Added

- **gds-investigator** — Investigation Memory (checked/leads/dead_ends lists, coverage protocol, deduplication, handoff), Failure Guards (depth limit, strength gate, contagion gate, consecutive call limit, stale lead expiry, force-switch), Decision Scoring (priority queue heuristic with anomaly_strength + graph_support + temporal_signal + novelty_bonus). Guidance for `dim_mask`, `metric="cosine"`, and `find_anomalies(metric="Linf")`.
- **gds-fraud-investigator** — same three sections adapted for the 3-phase fraud workflow. Entity 360 recipe extended with `find_novel_entities` as step 6 and `investigation_coverage` as step 8 in the graph confirmation chain. Triage levels (CRITICAL/HIGH/MEDIUM/LOW) integrated with decision scoring. All recipes generalized to use angle-bracket placeholders instead of sphere-specific names.

### Changed

- **6 skills** (investigator, fraud-investigator, detective, explorer, monitor, sphere-designer) — removed sphere-specific hardcoded dimension names, entity keys, pattern names, and column names from guidance text. Labeled examples preserved. Skill versions bumped to 0.3.3.

## [0.3.2] — 2026-04-13

### Changed

- **gds-sphere-designer** — added generalized dimension blocks (g/t/s) section documenting `geo_properties`, `metric_properties`, `semantic_dim` YAML keys with normalization table and YAML example.
- **gds-investigator** — added graph confirmation chain (`contagion_score` → `find_witness_cohort` → `find_novel_entities`) to Entity 360 and Root cause chain sections. Documents confidence escalation from isolated anomaly to confirmed pattern.
- **gds-fraud-investigator** — added `contagion_score` and `find_witness_cohort` steps to Entity 360 recipe. Added `find_novel_entities` as screening complement to passive_scan.

## [0.3.1] — 2026-04-12

### Changed

- **gds-fraud-investigator**, **gds-investigator**, **gds-detective**, **gds-scanner**, **gds-explorer** — document the new `fdr_alpha` (Benjamini-Hochberg FDR control) and `select="diverse"` (submodular facility location) parameters on `find_anomalies`, `attract_boundary`, `find_hubs`, and `find_drifting_entities`. Each skill frames the guidance for its audience: fraud investigation emphasizes false-positive cost and typological coverage, scanner emphasizes multiplicity at scale, explorer contrasts the two selection modes for orientation.

## [0.3.0] — 2026-04-12

### Changed

- **gds-sphere-designer** — added 5 troubleshooting rows covering edge table auto-detect pitfalls (metadata timestamp, narrow amount heuristic), NB-Split contagion resolution, and temporal build performance guidance. Temporal threshold raised from 2min to 5min with `--no-temporal` iteration strategy. Skill version bumped to 0.3.0.

## [0.2.2] — 2026-04-11

### Changed

- **gds-fraud-investigator**, **gds-investigator**, **gds-detective**, **gds-analyst**, **gds-monitor**, **gds-scanner** — document the new `timestamp_cutoff` as-of-graph-reconstruction parameter on the edge-table graph primitives. Each skill gets a one-line note alongside the existing recipe explaining how to reconstruct contagion / flow / influence state at a prior point in time.

## [0.2.1] — 2026-04-11

### Changed

- **gds-fraud-investigator** — new R8 Witness Cohort recipe in Edge Table Investigation Recipes section. Uses `find_witness_cohort` for fraud cohort expansion: find more accounts that share the target's anomaly signature, excluding already-connected counterparties. Honest framing: surfaces existing peers, NOT predictions of future edges. Bumped skill metadata.version to 0.2.1.

## [0.2.0] — 2026-04-10

### Changed

- **gds-fraud-investigator** — `discover_chains` replaces `extract_chains` as primary chain tool, `find_geometric_path` for suspect connection tracing, `edge_stats` prerequisite check, updated typology table and examples, 7 new edge table investigation recipes (Mirror Transaction, Pass-Through, Burst Detection, Weighted Reciprocity, Financial Profile, Concentration Risk, Benford's Law), `contagion_score_batch` in Phase 1 screening, `investigation_coverage` in Entity 360, `cluster_bridges` BRIDGE typology
- **gds-investigator** — `find_geometric_path` in root cause chain, `discover_chains` in entity 360, `edge_stats` prerequisite, new anti-patterns, `entity_flow`/`contagion_score`/`degree_velocity` in root cause chain, `anomalous_edges` in entity 360, `propagate_influence` hypothesis test, `investigation_coverage` anti-pattern
- **gds-explorer** — `edge_stats` in orientation phase, graph tools for binary FK patterns, 6 new edge tools in Tools by question table (`entity_flow`, `contagion_score`, `degree_velocity`, `investigation_coverage`, `cluster_bridges`, `anomalous_edges`)
- **gds-analyst** — `edge_stats` in orient, `entity_flow`/`contagion_score` in root cause chain, `investigation_coverage` in quality checklist
- **gds-detective** — `degree_velocity` for temporal burst corroboration, `contagion_score`/`contagion_score_batch` as preferred path for neighbor contamination
- **gds-scanner** — `contagion_score_batch` as preferred contamination path (over manual recipe), `cluster_bridges` complement for cross-pattern discrepancy, edge table row in scan selection table
- **gds-monitor** — `edge_stats` in quick health check, `degree_velocity` for drift/spike diagnostics, `contagion_score_batch` for emerging anomaly triage
- **gds-sphere-designer** — comprehensive edge table design section in Phase 2 (when to use, when not, auto-detect vs explicit config, coexistence with chain_lines, anti-patterns table)

---

## [0.1.0] — 2026-04-08

First release. 8 agent skills for GDS sphere navigation via hypertopos MCP server.

### Added

- **gds-analyst** — goal-oriented investigation driver with decision framework
- **gds-explorer** — sphere orientation, profiling, clustering, reactive if-then guidance
- **gds-investigator** — root cause tracing, entity deep-dive, hypothesis testing, bottom-up recall
- **gds-detective** — detection recipes (event rates, composites, drift, passive_scan)
- **gds-scanner** — advanced detection (cross-pattern, neighbor, trajectory, segment shift)
- **gds-monitor** — temporal drift, regime changes, health monitoring, dimension-specific drift
- **gds-fraud-investigator** — AML/fraud investigation with 25 typology recipes
- **gds-sphere-designer** — end-to-end 5-phase sphere design, build, calibration

All skills written in advisory style — knowledge bases with decision tables and error recovery, not command sequences. Each skill includes `references/examples.md` with concrete tool output examples.

Apache-2.0 licensed. pip installable via `pyproject.toml`.
