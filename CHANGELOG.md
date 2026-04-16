# Changelog

All notable changes to `hypertopos-skills` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

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
