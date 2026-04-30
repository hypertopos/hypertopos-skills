# hypertopos-skills

> Investigation workflows for AI agents exploring geometric data spaces.

[![License: Apache 2.0](https://img.shields.io/badge/license-Apache%202.0-green.svg)](LICENSE)
![Version](https://img.shields.io/badge/version-0.6.0-%235500FF.svg)

Structured behaviors that guide AI agents through exploration, investigation, detection, and monitoring of geometric data spaces. Not code — decision frameworks with examples.

## Skills

| Skill | Focus | Typical trigger |
|-------|-------|-----------------|
| [`gds-analyst`](gds-analyst/SKILL.md) | Investigation driver — goal-oriented, reads hints, plans approach | "investigate this sphere", "find anomalies" |
| [`gds-explorer`](gds-explorer/SKILL.md) | Orientation — understand a sphere you've never seen | "explore this sphere", "give me an overview" |
| [`gds-investigator`](gds-investigator/SKILL.md) | Root cause tracing, entity deep-dive, hypothesis testing | "why is this anomalous", "validate this finding" |
| [`gds-detective`](gds-detective/SKILL.md) | Detection cookbook — event rates, composites, drift, passive_scan | "detect copula", "windowed comparison" |
| [`gds-scanner`](gds-scanner/SKILL.md) | Advanced detection — cross-pattern, neighbor, trajectory, segment shift | "cross-pattern discrepancy", "trajectory shape" |
| [`gds-monitor`](gds-monitor/SKILL.md) | Monitoring — drift detection, regime changes, temporal health | "check sphere health", "detect drift" |
| [`gds-fraud-investigator`](gds-fraud-investigator/SKILL.md) | AML/fraud investigation and 25 typology recipes | "detect fraud", "run AML scan" |
| [`gds-sphere-designer`](gds-sphere-designer/SKILL.md) | End-to-end sphere design, construction, and tuning | "design a sphere", "build from my data" |

## How They Work Together

```text
Design:      gds-sphere-designer
Investigate: gds-analyst (drives the investigation)
               ├── gds-explorer        (orientation, profiling)
               ├── gds-investigator    (root cause, entity 360)
               ├── gds-detective       (detection recipes)
               ├── gds-scanner         (advanced detection)
               ├── gds-monitor         (temporal, drift, regimes)
               └── gds-fraud-investigator  (AML/fraud domain)
```

**gds-analyst** drives investigations — reads the task, picks the right approach, delegates to specialized skills. The other skills are the toolbox.

## Install

### Recommended

```bash
npx skills add hypertopos/hypertopos-skills
```

Works with Claude Code, Cursor, Codex, GitHub Copilot, Amp, Cline, Windsurf, and [other supported agents](https://skills.sh).

### Manual

Copy the `gds-*/` skill folders into your agent's skills directory (e.g. `.claude/skills/` for Claude Code).

### Prerequisites

- [hypertopos-mcp](https://github.com/hypertopos/hypertopos-mcp) available and connected
- [hypertopos](https://github.com/hypertopos/hypertopos-py) installed in the same environment
- A built sphere for skills that operate on live data

## Design

Skills are intentionally reactive — decision frameworks, not rigid checklists. MCP provides the tools and sphere data. Skills provide judgment, scope, and workflows.

## Status

Research-stage project. Skills and workflows are actively evolving.

## License

[Apache 2.0](LICENSE)
