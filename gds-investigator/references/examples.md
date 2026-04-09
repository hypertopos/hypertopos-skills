# GDS Investigator — Output Examples

Concrete output examples for each investigation recipe. Shows what the tool
returns and how to interpret it.

## Ground Truth Workflow — Example

**Call:** `check_anomaly_batch(keys=["ACCT-42", "ACCT-107", "ACCT-215", "ACCT-330", "ACCT-501"], pattern_id="account_pattern")`

**Output (key fields):**
```json
{
  "pattern_id": "account_pattern",
  "total_checked": 5,
  "anomaly_count": 3,
  "recall_if_all_bad": 0.60,
  "results": [
    {"key": "ACCT-42",  "is_anomaly": true,  "delta_rank_pct": 0.97},
    {"key": "ACCT-107", "is_anomaly": true,  "delta_rank_pct": 0.94},
    {"key": "ACCT-215", "is_anomaly": false, "delta_rank_pct": 0.62},
    {"key": "ACCT-330", "is_anomaly": true,  "delta_rank_pct": 0.91},
    {"key": "ACCT-501", "is_anomaly": false, "delta_rank_pct": 0.45}
  ]
}
```

**What to look for:**
- `recall_if_all_bad` = what fraction of "bad" keys are caught as anomalies
- Run this on EACH pattern covering these entities — pick the one with
  highest recall
- `delta_rank_pct > 0.90` = entity is in the extreme tail even if not
  flagged as anomaly (near-miss)
- Missed keys with low `delta_rank_pct` (ACCT-501 at 0.45) = geometrically
  normal, pattern does not capture this type of "bad"

**Finding (recall table):**

| Pattern | Anomalies | "Bad" caught | Recall | Precision |
|---------|-----------|-------------|--------|-----------|
| account_pattern | 312 | 3/5 | 60% | 1.0% |
| transaction_pattern | 187 | 4/5 | 80% | 2.1% |

"transaction_pattern achieves 80% recall on defaulted accounts. ACCT-501
(missed) has delta_rank_pct=0.45 — geometrically normal, likely a different
failure mode. Run composite_risk_batch on missed keys."

---

## No Ground Truth Workflow — Example

**Call:** `anomaly_summary(pattern_id="supplier_pattern")`

**Output (key fields):**
```json
{
  "pattern_id": "supplier_pattern",
  "total_entities": 1842,
  "anomaly_count": 168,
  "anomaly_rate": 0.091,
  "top_dimensions": [
    {"dimension": "total_spend", "mean_abs_delta": 2.41, "contribution": 0.38},
    {"dimension": "n_invoices", "mean_abs_delta": 1.87, "contribution": 0.29},
    {"dimension": "avg_late_days", "mean_abs_delta": 1.12, "contribution": 0.18}
  ]
}
```

**Call:** `find_anomalies(pattern_id="supplier_pattern", top_n=10)`

**Output (key fields):**
```json
{
  "results": [
    {"key": "SUPP-1234", "delta_norm": 4.82, "is_anomaly": true, "anomaly_dimensions": ["total_spend", "n_invoices"]},
    {"key": "SUPP-5678", "delta_norm": 3.91, "is_anomaly": true, "anomaly_dimensions": ["avg_late_days"]},
    {"key": "SUPP-9012", "delta_norm": 3.44, "is_anomaly": true, "anomaly_dimensions": ["total_spend"]}
  ]
}
```

**What to look for:**
- `anomaly_rate` around 5-15% = healthy calibration. Above 20% = theta
  too permissive, recalibrate first.
- `top_dimensions` shows what DRIVES anomalies overall — not per entity.
- `anomaly_dimensions` per entity shows which specific dimensions are
  anomalous FOR THAT entity.
- CHECK BOTH ENDS: if all top anomalies have the same sign (e.g. all
  high spenders), use `attract_boundary` to check the other extreme.

**Finding:** "168 anomalies (9.1% rate) in supplier_pattern. Primary driver
is total_spend (38% contribution). Top anomaly SUPP-1234 (delta_norm=4.82)
is anomalous on both total_spend and n_invoices — high-volume supplier with
unusual invoice pattern. Proceed with entity 360 on top 3."

---

## Root Cause Chain — Example

**Step 1 — Call:** `explain_anomaly(key="SUPP-1234", pattern_id="supplier_pattern")`

**Output (key fields):**
```json
{
  "key": "SUPP-1234",
  "delta_norm": 4.82,
  "is_anomaly": true,
  "anomaly_dimensions": [
    {"dimension": "total_spend", "delta": 3.91, "population_mu": 45200, "entity_value": 312000},
    {"dimension": "n_invoices", "delta": 2.67, "population_mu": 24, "entity_value": 187}
  ],
  "repair_set": {
    "repair_size": 2,
    "labels": ["total_spend", "n_invoices"],
    "residual_norm": 0.82,
    "theta_norm": 1.45
  }
}
```

**Step 2 — Call:** `dive_solid(key="SUPP-1234", pattern_id="supplier_pattern")`

**Output (key fields):**
```json
{
  "key": "SUPP-1234",
  "slices": [
    {"version": "v1", "timestamp": "2024-01-15", "delta_norm": 1.12, "is_anomaly": false},
    {"version": "v2", "timestamp": "2024-04-15", "delta_norm": 1.34, "is_anomaly": false},
    {"version": "v3", "timestamp": "2024-07-15", "delta_norm": 3.01, "is_anomaly": true},
    {"version": "v4", "timestamp": "2024-10-15", "delta_norm": 4.82, "is_anomaly": true}
  ]
}
```

**What to look for:**
- explain_anomaly: `repair_set` tells you the minimal dimensions to "fix"
  to make entity normal. `residual_norm < theta_norm` confirms the repair
  is sufficient.
- dive_solid: WHEN did the entity become anomalous? Sudden jump (v2->v3)
  = discrete event. Gradual increase = behavioral drift.
- Here: entity was normal through v2, jumped to anomalous in v3 (July 2024)
  = something happened in Q3 2024.

**Finding:** "SUPP-1234 anomaly driven by total_spend (312K vs population
mean 45K) and n_invoices (187 vs mean 24). Repair: zero 2 dims
(total_spend, n_invoices) -> residual 0.82 (below theta=1.45). Entity became
anomalous in v3 (July 2024) — sudden jump from delta_norm 1.34 to 3.01.
Investigate Q3 2024 events."

---

## Entity 360 — Example

**Call:** `cross_pattern_profile(key="SUPP-1234", line_id="suppliers")`

**Output (key fields):**
```json
{
  "key": "SUPP-1234",
  "line_id": "suppliers",
  "source_count": 3,
  "patterns": [
    {
      "pattern_id": "supplier_pattern",
      "is_anomaly": true,
      "delta_norm": 4.82,
      "delta_rank_pct": 0.98
    },
    {
      "pattern_id": "invoice_pattern",
      "is_anomaly": true,
      "delta_norm": 3.11,
      "delta_rank_pct": 0.94
    },
    {
      "pattern_id": "payment_pattern",
      "is_anomaly": false,
      "delta_norm": 1.23,
      "delta_rank_pct": 0.71
    }
  ]
}
```

**What to look for:**
- `source_count >= 2` = entity is anomalous in multiple independent patterns
  = high confidence finding, investigate thoroughly
- `source_count == 1` = single-source signal, likely false positive
- `delta_rank_pct` shows where entity sits in the population distribution
  even when not anomalous (payment_pattern: rank 71st percentile — elevated
  but not extreme)
- Here: SUPP-1234 is anomalous in 2 of 3 patterns = strong multi-source
  signal

**Finding:** "SUPP-1234 is anomalous in 3 independent views (source_count=3):
supplier_pattern (delta_norm=4.82, rank 98th pct), invoice_pattern
(delta_norm=3.11, rank 94th pct). Payment pattern shows elevated but
sub-threshold geometry (rank 71st pct). Multi-source confirmation —
HIGH confidence anomaly."

---

## Hypothesis Testing — Example

### Hypothesis: SUPP-1234 anomaly is driven by a single large contract awarded in Q3 2024

**Test:** `get_event_polygons(key="SUPP-1234", event_pattern="invoice_pattern", time_from="2024-07-01", time_to="2024-09-30")`

**Result:**
```json
{
  "events": [
    {"counterparty": "CUST-8901", "amount": 142000, "date": "2024-07-12"},
    {"counterparty": "CUST-8901", "amount": 89000,  "date": "2024-08-15"},
    {"counterparty": "CUST-8901", "amount": 67000,  "date": "2024-09-20"}
  ],
  "total_events": 42,
  "filtered_events": 3
}
```

**Verdict:** Partially confirmed. Three large invoices to CUST-8901 in Q3
total 298K (vs 312K total_spend), but these are spread across 3 months,
not a single contract. Pattern is consistent with an ongoing large engagement
rather than a one-off.

---

### Hypothesis: SUPP-1234 is a false positive — other large suppliers look the same

**Test:** `find_similar_entities(key="SUPP-1234", pattern_id="supplier_pattern", filter_expr="is_anomaly = false", top_n=20)`

**Result:**
```json
{
  "results": [
    {"key": "SUPP-7890", "similarity": 0.72, "is_anomaly": false, "delta_norm": 1.31},
    {"key": "SUPP-2345", "similarity": 0.68, "is_anomaly": false, "delta_norm": 1.18}
  ],
  "total_found": 2
}
```

**Verdict:** Rejected. Only 2 normal entities have similar geometric shape,
and their similarity scores are below 0.75. SUPP-1234's geometry is
distinctive — not a common pattern misclassified as anomalous.

---

## contrast_populations — Example

**Call:** `contrast_populations(pattern_id="account_pattern", group_a={"keys": ["ACCT-42", "ACCT-107", "ACCT-330"]})`

**Output (key fields):**
```json
{
  "n_group_a": 3,
  "n_group_b": 1839,
  "dimensions": [
    {"dimension": "avg_late_days",   "cohens_d": 2.14, "mean_a": 47.3, "mean_b": 4.8},
    {"dimension": "n_missed_payments", "cohens_d": 1.67, "mean_a": 8.2, "mean_b": 0.9},
    {"dimension": "total_balance",   "cohens_d": 0.34, "mean_a": 52100, "mean_b": 48200},
    {"dimension": "n_transactions",  "cohens_d": -0.12, "mean_a": 31, "mean_b": 34}
  ]
}
```

**What to look for:**
- Focus on the TOP dimension by |d|, not the average
- |d| > 0.8 = large effect size (practically significant)
- |d| > 1.5 = very large effect (strong discriminator)
- Positive d = group_a higher; negative d = group_b higher
- Here: avg_late_days (d=2.14) and n_missed_payments (d=1.67) are very
  large effects — these dimensions strongly separate the groups
- total_balance (d=0.34) = small effect, not a discriminator
- n_transactions (d=-0.12) = negligible, groups are the same on this dim

**Finding:** "Defaulted accounts differ from population primarily on
avg_late_days (Cohen's d=2.14, mean 47.3 vs 4.8 days) and n_missed_payments
(d=1.67, mean 8.2 vs 0.9). Balance and transaction volume show no
meaningful difference — default risk is driven by payment behavior,
not account size."

---

## False Positive Assessment — Example

**Call:** `find_similar_entities(key="CUST-5678", pattern_id="customer_pattern", filter_expr="is_anomaly = false", top_n=20)`

**Output (key fields):**
```json
{
  "query_key": "CUST-5678",
  "query_delta_norm": 2.89,
  "total_found": 14,
  "results": [
    {"key": "CUST-1111", "similarity": 0.94, "is_anomaly": false, "delta_norm": 1.38},
    {"key": "CUST-2222", "similarity": 0.92, "is_anomaly": false, "delta_norm": 1.41},
    {"key": "CUST-3333", "similarity": 0.91, "is_anomaly": false, "delta_norm": 1.35},
    {"key": "CUST-4444", "similarity": 0.89, "is_anomaly": false, "delta_norm": 1.29}
  ]
}
```

**What to look for:**
- 10+ normal entities with similarity > 0.85 = likely false positive.
  The entity's geometric shape is common in the normal population.
- Here: 14 normal entities match with similarity 0.89-0.94. CUST-5678
  has the same shape as many normal entities — it is just slightly more
  extreme (delta_norm 2.89 vs their ~1.35).
- If only 0-2 normal matches found = entity truly has a unique shape,
  anomaly likely genuine.
- Check: is the entity dormant/inactive? Dormant accounts flagged as
  anomalous are expected, not a finding.

**Finding:** "CUST-5678 likely FALSE POSITIVE. 14 normal entities share
its geometric shape (similarity 0.89-0.94). Entity is slightly more extreme
(delta_norm 2.89 vs ~1.35 for neighbors) but same archetype. Recommend
deprioritizing unless business context suggests otherwise."
