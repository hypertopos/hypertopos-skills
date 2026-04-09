# Changelog

All notable changes to `hypertopos-skills` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
