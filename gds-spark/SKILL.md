---
name: gds-spark
description: Anomaly detection on Spark / Databricks with the hypertopos one-call API — map a DataFrame's columns to roles, run find_anomalies, read and triage the result, explain an entity, score new data against a frozen calibration, and hand the result to an agent through the MCP server. Use when the user has events in a Spark DataFrame, a Delta table or a Databricks notebook and asks to "find anomalies in this table", "score these accounts", "which customers are unusual", "run hypertopos on Spark", "explain why this entity is flagged", "score today's data against last month", or "set up daily anomaly alerts". Requires hypertopos[spark] in the Spark environment; the MCP server is needed only for the hand-off.
license: Apache-2.0
compatibility: Requires pip install "hypertopos[spark]" (PySpark 3.5 or 4, Python 3.12; on Databricks, Runtime 16+ or serverless environment 3+). Works on classic clusters and Spark Connect. Hand-off to an agent needs hypertopos-mcp.
metadata:
  author: Karol Kędzia
  version: 0.10.1
---

# GDS on Spark

hypertopos on Spark takes a table of events (payments, calls, shipments,
logins) and gives every entity a position relative to its population: how far
it is from the centre, whether that is beyond the population's own threshold,
and which dimensions move it. One call does it; the work for the agent is in
choosing the roles, reading the result honestly and deciding what to do next.

Full reference: the Spark page of the hypertopos docs (`docs/spark.md`).

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
- Every event counts for **both sides**: an entity seen only as a receiver still gets a shape. So `entity` and `counterparty` should be keys of the same kind (account → account). If the counterparty is a different kind (merchant), the receivers become entities too. Say so in the report, or filter the result to the entity kind you care about.
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

Spark scores on the Mahalanobis axis only (no event motion, `current`): a sphere built from the same events with an event pattern can give the same entities different scores and p-values. Do not compare the two numbers as if they were one score.

Triage before reporting anything:
1. **Read `top_reasons` for every entity you report.** If one dimension holds most of the score (`pct_of_total` > 80), the finding is that one dimension being extreme. Check that the dimension is meaningful and not a data artefact, such as a unit change, a duplicated feed or a test account.
2. **Check the event counts.** Shares, spreads and ratios on few events are already pulled toward the mean, but an entity with very few events is still a weak finding. `explain` shows the adjusted value; quote the counts (`n_out`, `n_in` columns of `sphere_of(result).geometry()`, with the default dimensions) when it matters.
3. **Treat `pooled` and `global` scopes with care.** A pooled entity was compared with a mix of small groups, and a `global` one (from `score`) belonged to a group never seen at calibration.
4. **Many flags on one segment or one day** usually means a population shift or a data problem, not many independent anomalies. Group the flagged rows by segment and date before reporting them one by one.
5. **Say what the 1 % means.** The threshold flags about 1 % by construction, even in clean data. Report flagged entities as "most unusual relative to their population", not as fraud.

## 5. Explain an entity

```python
explain(result, top_n=10).show(30, truncate=False)          # the highest-scoring entities
explain(result, entities=["ACC-1029"]).show(truncate=False)  # named ones, every dimension
```

Each row gives `value`, the population `mu` and `sigma` (the group's with `group_by`), `delta`, `pct_of_total` and `direction`. Write the explanation in the data's terms. For example: "sends to 14 distinct counterparties against a population mean of 3 (+5.2σ), 46 % of its anomaly score". `explain` needs the result DataFrame itself; selecting away `entity` drops the link to the calibration.

## 6. Score new data without recalibrating

```python
from hypertopos.spark import score

today = score(spark.table("main.payments.today"), like=result)
today.where("is_anomaly").write.mode("append").saveAsTable("main.payments.daily_alerts")
```

Use this for monitoring: today against last month's population, so scores stay comparable from day to day. A scheduled job runs in a new process: calibrate with `sphere_root` on persistent storage (a Unity Catalog Volume on Databricks), keep `str(sphere_of(result).path)`, and score later runs with `score(df, like=sphere_of(path))`. On serverless, pass `sphere_root=` (a custom Spark conf cannot be set there). To explain a scored alert, pass the events it was scored on: `explain(today, events=new_events)`. A recalibration on every batch would move the baseline with the data. Recalibrate deliberately, when the population has really changed, by running `find_anomalies` on the new reference window.

## 7. Hand the result to an agent

```python
from hypertopos.spark import sphere_of

sphere = sphere_of(result)
sphere.geometry()          # every dimension's value and delta, as a table
sphere.lance_path()        # a navigable sphere with the same distances and flags
sphere.mcp_config()        # MCP client configuration for that sphere
```

With the MCP server on that sphere, the investigation continues with the other skills: `gds-investigator` for an entity deep-dive and false-positive checks, and `gds-explorer` for clusters and population structure. In the sphere, dimensions are labelled `t:<dimension>`, and with `group_by` the pattern is grouped by `calibration_group`.

## Anti-patterns

- Reporting `is_anomaly = true` as "fraud" or "suspicious" without reading `top_reasons` and the event counts.
- Mixing entities of different kinds and reporting the flagged ones without checking the group-relative reasons.
- Recalibrating on each new batch for monitoring, instead of `score(..., like=...)`.
- Passing labels as dimensions, or tuning `percentile` until a known case is flagged.
- Collecting the event table to the driver (`toPandas()` on raw events). Only the per-entity result is small.
