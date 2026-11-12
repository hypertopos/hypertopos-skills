---
name: gds-spark
description: Anomaly detection on Spark / Databricks with the hypertopos one-call API — map a DataFrame's columns to roles, run find_anomalies, read and triage the result, explain an entity, score new data against a frozen calibration, and hand the result to an agent through the MCP server. Use when the user has events in a Spark DataFrame, a Delta table or a Databricks notebook and asks to "find anomalies in this table", "score these accounts", "which customers are unusual", "run hypertopos on Spark", "explain why this entity is flagged", "score today's data against last month", or "set up daily anomaly alerts". Requires hypertopos[spark] in the Spark environment; the MCP server is needed only for the hand-off.
license: Apache-2.0
compatibility: Requires pip install "hypertopos[spark]" (PySpark 3.5 or 4, Python 3.12; on Databricks, Runtime 16+ or serverless environment 3+). Works on classic clusters and Spark Connect. Hand-off to an agent needs hypertopos-mcp.
metadata:
  author: Karol Kędzia
  version: 0.12.0
---

# GDS on Spark

hypertopos on Spark takes a table of events (payments, calls, shipments,
logins) and gives every entity a position relative to its population: how far
it is from the centre, whether that is beyond the population's own threshold,
and which dimensions move it. One call does it; the work for the agent is in
choosing the roles, reading the result honestly and deciding what to do next.

Full reference: the Spark page of the hypertopos docs (`docs/spark.md`).

**Profiles:** the Spark steps need no MCP server. The hand-off (section 7) runs on the default `investigator` profile; the deeper steps of `gds-investigator` and `gds-explorer` need `full`: set `HYPERTOPOS_MCP_PROFILE=full` in the server's `env`.

---

## 1. Map the columns to roles

Look at the schema first (`df.printSchema()`, a few rows). Then pick:

| Role | What to look for | If absent |
|---|---|---|
| `entity` (required) | who the event belongs to: sender account, customer, device, card | — |
| `counterparty` | the other side: receiver, merchant, called number | omit; no direction / counterparty dimensions |
| `amount` | numeric size of the event | omit; no money dimensions |
| `time` | timestamp, timestamp_ntz, date, or epoch seconds | omit; no rhythm dimensions |

Rules of thumb:
- A **string time column** is rejected. Cast it first: `F.to_timestamp(...)`, or `timestamp_ntz` for local wall-clock times.
- The **entities are the keys of the `entity` column**. An event also counts for its counterparty when that key is an entity (its incoming side), but a key seen only as a counterparty (a merchant, an account that only receives) gets no row and no score. To score those keys, make a second call with the roles swapped, and say in the report which side was scored. An entity that no event names as the counterparty has every incoming count, sum, mean and extreme at 0 and can be flagged for that alone: read `calibration_card(result)["one_sided"]`, and when it shows flags, report them as a separate group (`evidence_record` marks each with `no_incoming_events`).
- Do not pass a label column (fraud flag, SAR, chargeback) as a role or an extra dimension. It leaks the answer and destroys any evaluation.

## 2. Decide on `group_by`

`group_by` names an entity property (segment, country, product, customer type), taken per entity as the most frequent value in its own events.

| Situation | Choice |
|---|---|
| Entities of very different scale mixed together (retail and business, small and large merchants) | `group_by` without `score_per_group`: `delta_norm` and deviation reasons are group-relative, but the flag and score stay global — one population-wide ranking |
| Groups are distinct kinds of entity with different normal behaviour and each has many members | `group_by` + `score_per_group=True`: each group scored against its own reference; `anomaly_score`, `p_value` and `is_anomaly` are within-group; each group keeps its own alarm budget |
| One homogeneous population | no `group_by` |
| Many tiny groups | keep `min_group_size` (default 100; with `score_per_group=True` the default and minimum is the reference size, 199 at the 99th percentile): smaller groups are pooled and marked `calibration_scope = "pooled"` |

Default (`score_per_group=False`): `delta_norm` and the reasons' `delta` are group-relative; `anomaly_score`, `p_value`, `score_rank_pct` and `is_anomaly` stay global. With `score_per_group=True`, all four become within-group; `calibration_scope` says which reference scored each entity. Compare inside a group by filtering on the group column.

## 3. Run it

```python
from hypertopos.spark import find_anomalies, explain

result = find_anomalies(events, entity="from_account", counterparty="to_account",
                        amount="amount", time="ts", group_by="segment", score_per_group=True)
result.orderBy("anomaly_score", ascending=False).show(20, truncate=False)
```

- A second call on the same table with the same settings reuses the cached calibration. Pass `cache=False` after rewriting a plain file in place.
- No data to hand? `hypertopos.datasets.synthetic_aml(n_accounts=..., seed=...)` gives labelled payments to try the flow.

## 4. Read the result

| Column | How to read it |
|---|---|
| `anomaly_score` | calibrated evidence score; rank by it |
| `is_anomaly` | `p_value` at most `1 − percentile/100` (99 by default): the alarm budget, about 1 % of a clean population. A budget, not a verdict |
| `p_value` | conformal p of the score: share of the calibration population scoring at least as high. 0.001 means 1 in 1,000 |
| `score_rank_pct` | 0–100 position of the score in the population |
| `delta_norm` | distance from the (group's) centre in standard deviations; the shape, for reference, not the flag |
| `top_reasons` | up to three dimensions that drive `anomaly_score` most, with signed `delta` and share of the score in percent (the split `explain_anomaly` gives on a sphere) |
| `calibration_scope` | with `group_by`: `group`, `pooled` or `global` (with `score_per_group=True`: which reference scored the entity) |

The score has the Mahalanobis axis (`mahal`: how rare the combination of dimensions is) and, by default when `time` is given and `score_per_group` is off, `current`: how unusually the entity's activity changed between the two calendar halves of the events against entities in the same state; with `current` the score equals that of a sphere built from the same entities and events with an event pattern. Without `time` or with `score_per_group=True` the default is `mahal` alone, and `score_axes=("mahal",)` leaves `current` out (about a fifth less fit time); leave it out when the behaviour you look for does not change between the two halves of the window, since `current` then only adds noise to the ranking. `current` needs events on both sides of the window's middle day, so on a one-day window (a daily calibration) it changes no p-value, rank or flag; pass `score_axes=("mahal",)` there to save its cost. With `score_per_group=True`, `score_axes=("mahal", "current")` adds it, each group's reference fitting its own velocity field. Without `current`, a sphere with an event pattern scores motion that Spark does not, so do not compare those numbers as one score. `score` scores new data on `mahal` alone even against a calibration with `current`. `top_reasons` split only the `mahal` part over the dimensions: an entity can rank high on `current` with modest reasons, so say which axis drives it (compare `delta_norm` and the reasons with the score).

Triage before reporting anything:
0. **Read `calibration_card(result)` once.** A group whose `can_flag` is false can never be flagged, however unusual its members are (its reference is smaller than `2 / alpha - 1`): say so instead of reporting "no anomalies" for it. `flag_drivers` says which dimension leads how many of the flags: when one or two dimensions lead most of them, tell the user the flags are mostly a ranking on those dimensions, and name them. A dimension in `tied_dimensions` has one value for at least 95 % of the entities, so any other value is an extreme deviation on its own; a flag whose top reason is such a dimension rests on one rare value.
1. **Read `top_reasons` for every entity you report.** If one dimension holds most of the score (`pct_of_total` > 80), the finding is that one dimension being extreme. Check that the dimension is meaningful and not a data artefact, such as a unit change, a duplicated feed or a test account.
2. **Check the event counts.** Shares, spreads and ratios on few events are already pulled toward the mean, but an entity with very few events is still a weak finding. `explain` shows the adjusted value; quote the counts (`n_out`, `n_in` columns of `sphere_of(result).geometry()`, with the default dimensions) when it matters.
3. **Treat `pooled` and `global` scopes with care.** A pooled entity was compared with a mix of small groups, and a `global` one (from `score`) belonged to a group never seen at calibration, or, with `score_per_group`, to a pool too small for its own reference.
4. **Many flags on one segment or one day** usually means a population shift or a data problem, not many independent anomalies. Group the flagged rows by segment and date before reporting them one by one.
5. **Say what the 1 % means.** The threshold flags about 1 % by construction, even in clean data. Report flagged entities as "most unusual relative to their population", not as fraud.

## 5. Explain an entity

```python
explain(result, top_n=10).show(30, truncate=False)          # the highest-scoring entities
explain(result, entities=["ACC-1029"]).show(truncate=False)  # named ones, every dimension
```

Each row gives `value`, the population `mu` and `sigma` (the group's with `group_by`), `delta`, `pct_of_total` and `direction`. Write the explanation in the data's terms. For example: "sends to 14 distinct counterparties against a population mean of 3 (+5.2σ), 46 % of its anomaly score". `explain` needs the result DataFrame itself; selecting away `entity` drops the link to the calibration.

`pct_of_total` divides only the `mahal` part of the score. When the result has the `current` axis, check `axis_evidence(result, entities=[...])` first: `bits_current` close to `bits` means the entity stands out because its activity changed between the two halves of the period, not because its values are extreme. Say so instead of reading the dimension rows as the whole reason. `bits_current` is null without `current` (no `time` role, or `score_per_group` without `score_axes=("mahal", "current")`).

**Finding entities by position.** When the question names dimensions ("accounts with many counterparties but small amounts", "the top 1 % by inflow that are not flagged"), run with `dimension_ranks=True` and filter the result in SQL on `dimension_rank_pct['<dimension>']` (0-100, the share of the population below the entity's measured value; within the entity's group with `score_per_group`). Do not rebuild the ranks from the events with window functions: the column ranks against the population the entity was scored in. A rank near 50 on a dimension listed in `tied_dimensions` means "the common value", not "average". The rank is of the value as measured: the score uses shares and spreads after a small-count adjustment (so `explain` can show a different value), but equal measured values always rank equal.

**Writing up a case.** When an entity goes into a report, a ticket or a table, take its row from `evidence_record(result, entities=[...])` instead of assembling it from `explain`, `axis_evidence` and the card. Quote `notes` in the write-up: `tied_dimension:<dim>` means the reason rests on one rare value, `reference_cannot_flag` means the population was too small to flag anyone, `scope_pooled` / `scope_global` mean the entity was not compared with its own group. Keep `calibration` with the case: it identifies the population the verdict was made against.

## 6. Score new data without recalibrating

```python
from hypertopos.spark import score

today = score(spark.table("main.payments.today"), like=result)
today.where("is_anomaly").write.mode("append").saveAsTable("main.payments.daily_alerts")
```

Use this for monitoring with **windows of the same length**: today against yesterday (or the same weekday a week earlier), a fixed number of days against the same number. Counts, sums, distinct counterparties and active days grow with the window, so a day scored against a month flags almost everyone. A scheduled job runs in a new process: calibrate with `sphere_root` on persistent storage (a Unity Catalog Volume on Databricks), keep `str(sphere_of(result).path)`, and score later runs with `score(df, like=sphere_of(path))`. On serverless, pass `sphere_root=` (a custom Spark conf cannot be set there). To explain a scored alert, pass the events it was scored on: `explain(today, events=new_events)`. A recalibration on every batch would move the baseline with the data. Recalibrate deliberately, when the population has really changed, by running `find_anomalies` on the new reference window.

Check that the baseline still holds before trusting a batch's flags:

```python
from hypertopos.spark import calibration_status, score

today = score(new_events, like=result, dimensions=True).persist()
status = calibration_status(today, like=result)
```

`status["status"]` is `ok`, `drift` (report `drifted_dimensions`, the dimensions that moved, and recalibrate on a recent window), `window_mismatch` (the scored window, `window_days`, is not 95–100 % of the calibration's `calibration_window_days`: recalibrate on a window of the same length; the `reason` names both), or `too_few_rows` (under 100 rows). With `group_by`, `status["groups"]` adds one entry per group (`scope`, `rows`, `flagged`, `status`, the p-values). With `group_by` alone, every group is scored against the whole population, so the tests run on the whole batch (an unseen or renamed group included) and each group is `untested`. With `score_per_group=True`, each group is tested against its own reference: report `drifted_groups` and each drifted group's `drifted_dimensions`. A group under 100 rows is `too_few_rows`, and with `score_per_group` entities scored against the population (`scope` `global`, an unseen group) are `untested` (all of them if the group values were recoded: then the status is `too_few_rows`); neither is drift. Keep the group column in the scored frame. Use fixed-length windows (e.g. 28 or 30 days), never calendar months: 28 against 31 days already fails. A calibration saved before the window check reports `calibration_window_days` `None` and skips it. Pass the whole scored batch, never `today.where("is_anomaly")` or another subset: a filtered frame reads as drift. It runs one Spark action over `today`, so persist or write the scored frame first or the scoring runs twice; on serverless compute that refuses `persist` (`CACHE_REQUIRES_NEWER_CLIENT_ON_SERVERLESS_GC`), write it to a table and read it back. A `find_anomalies` result is refused: it needs `score(..., dimensions=True)`. `score_axes` (always `["mahal"]`) and `calibration_axes` say which axes the scored batch and the calibration's own result were ranked on: a rescored entity's `anomaly_score` differs from its `find_anomalies` score when `calibration_axes` includes `current`. In a scheduled job, append each run's status to a Delta table (one row for the batch, one per group; the recipe is in the Spark guide, "Keep the checks in a table"), and recalibrate when the batch row is not `ok`.

**Following entities over runs.** In a scheduled job, append each run's scored rows (`entity`, `p_value`, `is_anomaly`, `anomaly_score`, `top_reasons`) with the run date and `sphere_of(...).fingerprint` to a history table. "Since when is it flagged" is then `MIN(run_date)` over its flagged rows, and "newly flagged" is flagged now and not in the previous run. Compare flags only between runs with the same calibration fingerprint, and say so when a change coincides with a recalibration.

**What changed since the last run.** `compare(previous, current)` takes two `score` results scored `like` the same sphere (never a `find_anomalies` result: score the calibration window's events with `score(events, like=result)` first) and returns one row per entity with `transition`: `entered`, `left`, `stayed_flagged`, `stayed_unflagged`, `appeared` or `disappeared`, plus both p-values, flags and scores. Report `entered` as entities to look at, not as incidents: a few entities change sides between ordinary windows by chance. It does not say why a verdict changed, and do not infer it from which dimension's rank moved most between the runs (ranks of dimensions built from few events move by themselves); take the reasons from the current run's `top_reasons` or `explain`. Results of two different calibrations are refused: recalibrating starts a new series.

**Sharing a population.** To let someone else score their events against this population, give them `sphere_of(result).export_reference(root)`, not the calibration directory: the export has no entity table, keys or states. Tell them it is not anonymised (it holds every dimension's quantiles, the sorted scores and, with `group_by`, the group names and sizes), that their events need the same column names, and to read `calibration_status` first: `drift` means their entities are not distributed like the reference population, so their flags mean "unlike that population", not "anomalous among themselves".

## 7. Hand the result to an agent

```python
from hypertopos.spark import sphere_of

sphere = sphere_of(result)
sphere.geometry()          # every dimension's value and delta, as a table
sphere.lance_path()        # a navigable sphere with the same distances and flags
sphere.mcp_config()        # MCP client configuration for that sphere
```

With the MCP server on that sphere, the investigation continues with the other skills: `gds-investigator` for an entity deep-dive and false-positive checks, and `gds-explorer` for clusters and population structure. `mcp_config()` starts the default `investigator` profile (screening, explanation, money flow); both skills need `full` for their deeper steps, so add `"HYPERTOPOS_MCP_PROFILE": "full"` to its `env` (or `"inspector"` to walk records, polygons and lines). In the sphere, dimensions are labelled `t:<dimension>`, and with `group_by` the pattern is grouped by `calibration_group`.

## Anti-patterns

- Reporting `is_anomaly = true` as "fraud" or "suspicious" without reading `top_reasons` and the event counts.
- Mixing entities of different kinds and reporting the flagged ones without checking the group-relative reasons.
- Recalibrating on each new batch for monitoring, instead of `score(..., like=...)`.
- Scoring a batch against a calibration of a different window length (a day against a month): window-length dimensions push nearly every entity over the budget.
- Passing labels as dimensions, or tuning `percentile` until a known case is flagged.
- Collecting the event table to the driver (`toPandas()` on raw events). Only the per-entity result is small.
- Treating `max_entities` as a memory guarantee. It counts rows; the real limit is the driver's memory (about 2.5 GB per million entities at the peak, with `mahal` alone or with `current`; size the driver at about 3 GB per million). On Spark Connect, `score` also sends the calibration with its tasks (about 16 MB per million fitted entities) and a message above 128 MB is refused, so `score` there works up to about 8 million fitted entities. Where the driver's memory cannot be set, as on Databricks serverless, it can also vary between runs. Compare the estimate with the driver before a large run; if it is close, lower `max_entities` or use a driver with more memory.
