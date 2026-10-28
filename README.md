# hypertopos-skills

> Business data as geometry that AI agents navigate — the investigation workflows.

[![License: Apache 2.0](https://img.shields.io/badge/license-Apache%202.0-green.svg)](LICENSE)
![Version](https://img.shields.io/badge/version-0.10.0-%235500FF.svg)

Install the skills into your agent, connect [hypertopos-mcp](https://github.com/hypertopos/hypertopos-mcp) to a sphere, and ask in plain language — the matching skill decides which tool to call, in which order, what the result means, and what to check next.

- **Judgement, not code.** Nine Markdown skills — decision frameworks with worked examples — that an agent reads before it acts. No Python, nothing to run.
- **Every investigation ends in a report, not a tool dump.** The skills carry a quality checklist, a report structure and anti-patterns, so a finding states its evidence, its confidence and its false-positive risk.
- **They know the tools' edges.** Compact responses and the `omitted` map, sessions pinned to a snapshot, branches for what-if recalibration, and the parameter names a call must use.

**Get started** (one minute). Needs an agent that reads skills and a connected hypertopos MCP server:

```bash
npx skills add hypertopos/hypertopos-skills     # Claude Code, Cursor, Codex, Copilot, Amp, Cline, Windsurf, …
# or: copy the gds-*/ folders into your agent's skills directory, e.g. .claude/skills/ for Claude Code
```

In Claude Code, the repository is also a plugin marketplace, so the skills install and update as a plugin:

```
/plugin marketplace add hypertopos/hypertopos-skills
/plugin install hypertopos-skills@hypertopos-skills
```

Then, in your AI assistant:

```
explore this sphere
why is account 85 anomalous?
check sphere health
run an AML scan
```

That's it. You get nine skills, each a folder with a `SKILL.md` (and `references/` where the skill carries worked examples and detail the `SKILL.md` links to):

```
gds-analyst/            drives an investigation: goal, plan, delegation, report
gds-explorer/           orientation in a sphere you have never seen
gds-investigator/       root cause, entity deep-dive, hypothesis testing, false-positive assessment
gds-detective/          detection recipes: event rates, bursts, Simpson's paradox, collective drift
gds-scanner/            cross-pattern, neighbour-contamination, trajectory-shape and segment-shift scans
gds-monitor/            health checks, drift, regime changes, calibration epochs, safe recalibration
gds-fraud-investigator/ AML and fraud: screening, per-suspect investigation, typology recipes
gds-sphere-designer/    from a description of the data to a sphere.yaml, a build, and tuning
gds-spark/              anomaly detection on a Spark DataFrame: roles, find_anomalies, triage, explain, score
```

Works with Claude Code, Cursor, Codex, GitHub Copilot, Amp, Cline, Windsurf, and [other supported agents](https://skills.sh).

> [!NOTE]
> The skills need [hypertopos-mcp](https://github.com/hypertopos/hypertopos-mcp) connected to a built sphere; they call its tools and read its responses. [hypertopos](https://github.com/hypertopos/hypertopos-py) builds the sphere. `gds-sphere-designer` works before a sphere exists, and `gds-spark` works in a Spark or Databricks notebook with `hypertopos[spark]`, needing the MCP server only to hand a result to an agent.

## See it in action

A skill is text the agent follows. This is what `gds-monitor` says about recalibrating when the entities under review may be distorting the baseline (verbatim from its "Safe recalibration via branch" section):

```text
# 1. Recalibrate on a branch (optionally excluding known influencers)
recalibrate(pattern_id, branch="my_branch", exclude_keys=["KEY1", "KEY2"])

# 2. Compare the branch to main (from the main session)
compare_branches(pattern_id, branch="my_branch")
  -> anomalies_from/to, flipped_in/flipped_out, flipped_in_top/flipped_out_top,
     jaccard, top_movers, calibration_drift
#    To drill into the branch, open_sphere(path, branch="my_branch"); reopen
#    main with open_sphere(path) before promoting or deleting it.

# 3a. Calibration looks correct — promote to main (replays recipes, verifies, deletes branch)
manage_branch(action="promote", name="my_branch")

# 3b. Calibration looks wrong — discard the branch
manage_branch(action="delete", name="my_branch")
```

And this is what an agent following it did on a small synthetic banking sphere, having found account `20` at the top of `find_anomalies` (real tool output, trimmed):

```text
recalibrate(pattern_id="account_behavior_pattern", branch="trial", exclude_keys=["20"])
{"records_recalibrated": 121, "n_excluded": 1, "branch": "trial", …}

compare_branches(pattern_id="account_behavior_pattern", branch="trial")
{"from_ref": "main", "to_ref": "trial", "anomalies_from": 5, "anomalies_to": 6,
 "flipped_in": 1, "flipped_out": 0, "jaccard": 0.8333333,
 "flipped_in_top": [{"primary_key": "119", "is_anomaly_from": false, "is_anomaly_to": true, …}],
 "calibration_drift": {"overall_drift_rms": 0.01551632, …}, …}

manage_branch(action="promote", name="trial")
{"promoted": {"name": "trial", "recipes_replayed": 1, …}}
```

The skill supplies the verdict: the fit moved by 0.02 σ RMS and no flagged account left the alarm budget, so the calibration was not anchored on the suspect; one account (`119`) entered it, so the agent investigated `119` first and promoted once it checked out. The same section tells the agent that `promote` refuses when main has moved and aborts when the replayed fit does not match, so the call is safe without manual verification.

Orientation works the same way. `gds-explorer` opens every exploration with three calls and then tells the agent which signals in the answers decide the next step:

```text
sphere_overview()        -> patterns, anomaly rates, calibration health, profiling_alerts
get_sphere_info()        -> lines, columns, aliases, FTS availability
anomaly_summary(top_pattern) -> what drives anomalies
```

— `dimension_kinds` to see count-heavy versus magnitude-heavy geometry, `profiling_alerts` to run each alert's suggested `find_anomalies(rank_by_property=…)`, `dim_quality_warnings` to raise a calibration ticket instead of investigating a one-dimension detector.

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
| [`gds-spark`](gds-spark/SKILL.md) | Anomaly detection on Spark / Databricks with the one-call API | "find anomalies in this table", "score today against last month" |

## How they work together

```text
Design:      gds-sphere-designer
On Spark:    gds-spark  (one call on a DataFrame; hands the result to the skills below)
Investigate: gds-analyst (drives the investigation)
               ├── gds-explorer        (orientation, profiling)
               ├── gds-investigator    (root cause, entity 360)
               ├── gds-detective       (detection recipes)
               ├── gds-scanner         (advanced detection)
               ├── gds-monitor         (temporal, drift, regimes)
               └── gds-fraud-investigator  (AML/fraud domain)
```

**gds-analyst** drives investigations — reads the task, picks the right approach, delegates to specialized skills. The other skills are the toolbox; each ends with a "Skill delegation" section naming where to hand off. **gds-sphere-designer** comes before all of them: it turns a description of the data into the `sphere.yaml` the others navigate, and tunes the patterns when detection looks wrong. **gds-spark** is the entry point when the data lives in Spark: it runs the one-call API in a notebook and hands the flagged entities to the MCP server and the investigation skills.

## Common prompts

```text
"explore this sphere"                          # gds-explorer: orientation → profiling → hand-off
"investigate this sphere" / "find anomalies"   # gds-analyst: plans the investigation, delegates, writes the report
"why is account 85 anomalous?"                 # gds-investigator: root cause chain, entity 360
"is this a false positive?"                    # gds-investigator: false positive assessment, hypothesis testing
"detect copula" / "windowed comparison"        # gds-detective: a named detection recipe
"cross-pattern discrepancy"                    # gds-scanner: one of the four advanced scans
"check sphere health" / "detect drift"         # gds-monitor: quick health check → alerts → drift → calibration epochs
"run an AML scan" / "detect fraud"             # gds-fraud-investigator: screening → per-suspect investigation → typologies
"design a sphere from my data"                 # gds-sphere-designer: discover → design → configure → build → iterate
"why is my recall low?"                        # gds-sphere-designer: iterate and troubleshoot
```

### Conventions the skills rely on

- **Parameter names.** Tools name the entity `primary_key` and the result count `top_n`. A call with an unknown parameter fails, and the error suggests the closest declared name when there is one — a wrong name never silently falls back to a default.
- **Compact responses.** Tools answer in a compact format by default; an `omitted` map names what was left out (positional `delta` vectors, numeric property columns) and how to get it back (`summary=false`, or `open_sphere(path, response_format="full")`). **gds-explorer** and **gds-investigator** read the map before asking for more.
- **Pinned sessions.** A session reads the sphere as it was when `open_sphere` ran. `get_session_stats()` reports `snapshot.stale` when a newer state exists; `open_sphere(path)` moves to it. **gds-monitor** builds this into periodic checks.
- **Branches for what-if work.** `recalibrate(pattern_id, branch=..., exclude_keys=[...])` fits a calibration on a sphere branch without touching main; `compare_branches` shows what changes; `manage_branch` promotes or deletes the branch. **gds-monitor**, **gds-fraud-investigator** and **gds-sphere-designer** show the workflow; **gds-explorer** shows how to open a branch.
- **Calibration epochs.** A rebuild and a `recalibrate` each record an epoch, so **gds-monitor** runs `compare_calibrations` after either to see what moved.

> [!IMPORTANT]
> `sphere_overview()` comes first. Until it runs, the agent sees only the session, gateway and edge-table tools; the skills assume the full toolset and start with it.

> [!TIP]
> `gds-explorer` keeps `detail="summary"` in `sphere_overview` for interactive work. `detail="full"` runs event-rate scans that can take minutes on large spheres; the skill reserves it for deep-dive diagnostics.

> [!WARNING]
> Promote or delete a branch only from a session open on main. The skills say to reopen main with `open_sphere(path)` after exploring a branch; `manage_branch` refuses otherwise.

## Troubleshooting

| Symptom | Fix |
|---|---|
| The agent does not pick up a skill | Use a trigger phrase from the table above, or name the skill. Check the skill folders are in the agent's skills directory. |
| "Tool X is not available" right after `open_sphere` | The agent skipped orientation. `sphere_overview()` unlocks the manual toolset. |
| Temporal recipes (`gds-monitor`, trajectory scans) find no tools | The sphere has no temporal data. Rebuild with a temporal configuration; `open_sphere` reports `capabilities.has_temporal`. |
| `unknown parameter 'limit' … did you mean 'top_n'?` | The agent used a name from another API. The skills' examples use the declared names; the error names the right one. |
| `detect_pattern` returns nothing useful | Rephrase with the entity type and a dimension name; the skills prefer `sphere_overview` plus targeted calls when the query is vague. |
| The agent reports an anomaly that is one dimension's artefact | `dim_quality_warnings` fired `dominant_dim_mass`. `gds-explorer` says to raise a calibration ticket (`gds-sphere-designer`) instead of investigating. |
| Answers do not change after new data arrived | The session is pinned to a snapshot. `get_session_stats()` → `snapshot.stale`, then `open_sphere(path)`. |
| A fraud recipe says "no edge table" | The event pattern has fewer than two relations to the same anchor line, or the sphere was built with `--no-edges`. `gds-fraud-investigator` checks this first ("Prerequisites — Edge Table Check"). |

## What is inside a skill

<details>
<summary><b>Section map</b> (click to expand)</summary>

| Skill | Sections |
|---|---|
| `gds-analyst` | Starting an investigation · When you know what to look for · When you're exploring freely · Budget allocation · Quality checklist · Report structure · Anti-patterns · Skill delegation |
| `gds-explorer` | Orientation · Profiling raw dimensions · Matching signals to next steps · Tools by question · Parameter guidance · Selection modes for anomaly results · Avoiding common exploration traps · Minimal spheres · Report structure · When things don't work |
| `gds-investigator` | Core principle · Investigating with / without ground truth · FDR control and diverse selection · Root cause chain · Entity 360 · Hypothesis testing · Population contamination · Cross-pattern lead-lag · False positive assessment · Investigation memory · Failure guards · Decision scoring · Output format · Hidden influencers · Anomaly by absence |
| `gds-detective` | FDR control · Multi-resolution FDR · Choosing a recipe · Event anomaly rate · Composite subgroup analysis (Simpson's paradox) · Temporal burst detection · Neighbor contamination · Trajectory shape · Population segment shift · Collective drift · Multi-pattern investigation · passive_scan for confirmation · Exhaustive enumeration · Re-ranking by property · Temporal artifact check · Physical bound validation · Payment type segmentation · Unregistered entity check |
| `gds-scanner` | FDR control, diverse selection and confidence filtering · Choosing which scans to run · Alias anomaly scan · Cross-pattern discrepancy · Geometric neighbor contamination · Temporal trajectory shape · Population segment shift |
| `gds-monitor` | Quick health check · Responding to alerts · Event rate divergence · Windowed volume comparison · Drift detection · Compare calibration epochs · Safe recalibration via branch · Threshold sensitivity · Dim-quality warnings · Regime changes · Emerging anomalies · Handing off · Anti-patterns |
| `gds-fraud-investigator` | Core principle · Prerequisites — edge table check · Threshold convention · Phase 1 Screening · Phase 2 Investigation (per suspect) · Phase 3 Typology detection · Per-pattern guide · Edge table recipes · Investigation memory · Failure guards · Decision scoring · Common pitfalls · Aggregated edge dims · Declarative motif cheatsheets |
| `gds-spark` | Map the columns to roles · Decide on group_by · Run it · Read the result (triage) · Explain an entity · Score new data · Hand the result to an agent · Anti-patterns |
| `gds-sphere-designer` | Core principle · Phase 1 Discover · Phase 2 Design · Phase 3 Configure · Phase 4 Build + verify · Phase 5 Iterate · Calibration recommendations · Troubleshooting · Unsupervised ceiling · Examples |

Every skill also carries a "When things don't work" or "Troubleshooting" section and a "Skill delegation" section.

</details>

## Design

Skills are intentionally reactive — decision frameworks, not rigid checklists. MCP provides the tools and sphere data. Skills provide judgement, scope, and workflows.

## Status

Research-stage project. Skills and workflows are actively evolving.

## License

[Apache 2.0](LICENSE)
