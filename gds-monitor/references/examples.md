# GDS Monitor — Output Examples

Concrete output examples for each monitoring recipe. Shows what the tool
returns and how to interpret it.

## Event Rate Divergence — Example

**Call:** `aggregate(pattern_id="invoice_pattern", group_by_line="suppliers", geometry_filters={"is_anomaly": true})`

**Output (key fields):**
```json
{
  "total_groups": 1842,
  "groups_with_matches": 127,
  "results": [
    {"key": "SUPP-1234", "count": 48, "anomaly_count": 19, "anomaly_rate": 0.396},
    {"key": "SUPP-5678", "count": 112, "anomaly_count": 22, "anomaly_rate": 0.196},
    {"key": "SUPP-9012", "count": 35, "anomaly_count": 6, "anomaly_rate": 0.171}
  ]
}
```

**What to look for:**
- `anomaly_rate > 0.15` = entity has concentrated anomalous events
- Cross-check: is this supplier ALSO anomalous in the anchor pattern? If not,
  the anomaly is hidden at the event level only — temporal investigation needed.
- `anomaly_rate` near 1.0 = nearly all events anomalous — likely a structural
  issue (wrong counterparty, data quality) rather than behavioral drift.

**Finding:** "SUPP-1234 has 40% anomalous invoices (19/48) but is NOT flagged
in the supplier anchor pattern — concentrated temporal anomalies invisible at
entity level. Investigate with drift detection."

---

## Windowed Volume Comparison — Example

**Call (Period A):** `aggregate(pattern_id="invoice_pattern", group_by_line="suppliers", metric="count", time_from="2025-01-01", time_to="2025-03-31", limit=20)`

**Call (Period B):** `aggregate(pattern_id="invoice_pattern", group_by_line="suppliers", metric="count", time_from="2025-04-01", time_to="2025-06-30", limit=20)`

**Output Period A (key fields):**
```json
{
  "results": [
    {"key": "SUPP-1234", "count": 12},
    {"key": "SUPP-5678", "count": 45},
    {"key": "SUPP-3456", "count": 8}
  ]
}
```

**Output Period B (key fields):**
```json
{
  "results": [
    {"key": "SUPP-1234", "count": 47},
    {"key": "SUPP-5678", "count": 41},
    {"key": "SUPP-3456", "count": 31}
  ]
}
```

**What to look for:**
- 2x+ count increase = burst candidate (order splitting, seasonal spike,
  data ingestion anomaly)
- Sudden drop to 0 = entity went inactive or data pipeline issue
- Stable counts across periods = healthy, no action needed
- Compare ratios, not absolute numbers — a supplier with 8->31 (3.9x) is
  more interesting than one with 45->41 (stable)

**Finding:** "SUPP-1234 invoice volume jumped 3.9x (12 to 47) between Q1
and Q2. SUPP-3456 also spiked (8 to 31, 3.9x). Both warrant drift
investigation — possible order splitting or onboarding of new product lines."

---

## Drift Detection — Example

**Call:** `find_drifting_entities(pattern_id="supplier_pattern", top_n=10, forecast_horizon=3)`

**Output (key fields):**
```json
{
  "pattern_id": "supplier_pattern",
  "total_with_history": 1842,
  "results": [
    {
      "key": "SUPP-1234",
      "displacement": 3.82,
      "direction": [0.71, -0.34, 0.58, 0.12],
      "forecast_anomaly": true,
      "reliability": "high",
      "current_anomaly": false
    },
    {
      "key": "SUPP-5678",
      "displacement": 2.91,
      "direction": [0.85, 0.11, -0.05, 0.02],
      "forecast_anomaly": false,
      "reliability": "medium",
      "current_anomaly": false
    },
    {
      "key": "SUPP-9012",
      "displacement": 2.44,
      "direction": [-0.12, 0.92, 0.03, -0.08],
      "forecast_anomaly": true,
      "reliability": "high",
      "current_anomaly": true
    }
  ]
}
```

**What to look for:**
- `displacement > 2.0` = significant behavioral change
- `forecast_anomaly=true` + `reliability=high` = early warning, entity
  trending toward anomaly boundary
- `current_anomaly=false` + `forecast_anomaly=true` = the critical case:
  entity is normal NOW but predicted to become anomalous
- Direction vector: similar directions across top drifters = systemic
  population shift; random directions = independent entity changes
- SUPP-1234 and SUPP-5678 both have high first-dimension values (0.71, 0.85)
  = likely the same dimension driving drift. Use `rank_by_dimension` to
  isolate per-dimension drift.

**Finding:** "3 of top 10 drifters share dominant drift on dimension 1
(likely total_spend). SUPP-1234 is currently normal but forecast to become
anomalous with high reliability — early warning. Recommend dive_solid to
confirm whether drift is gradual or sudden."

---

## Regime Change Detection — Example

**Call:** `find_regime_changes(pattern_id="supplier_pattern", n_regimes=5)`

**Output (key fields):**
```json
{
  "pattern_id": "supplier_pattern",
  "n_changepoints": 2,
  "regimes": [
    {"regime": 1, "start": "2024-01-01", "end": "2024-06-30", "n_entities": 1842},
    {"regime": 2, "start": "2024-07-01", "end": "2024-12-31", "n_entities": 1856},
    {"regime": 3, "start": "2025-01-01", "end": "2025-06-30", "n_entities": 1901}
  ],
  "changepoints": [
    {
      "date": "2024-07-01",
      "magnitude": 1.34,
      "dominant_dimension": "burst_monthly",
      "direction": "increase"
    },
    {
      "date": "2025-01-01",
      "magnitude": 0.42,
      "dominant_dimension": "n_categories",
      "direction": "decrease"
    }
  ]
}
```

**What to look for:**
- `magnitude > 1.0` = large regime shift (external event, policy change)
- `magnitude < 0.5` = gradual evolution, usually not actionable
- 1 unidirectional shift = regime change (investigate external cause)
- 2 shifts in opposite directions = oscillation (seasonal, cyclical)
- ALL entities affected simultaneously = could be a data boundary artifact
  (batch load boundary, fiscal year reset), not real behavioral change
- Use `compare_time_windows` to drill into which dimensions shifted

**Finding:** "Major regime shift detected at 2024-07-01 (magnitude 1.34)
driven by burst_monthly increase. Minor shift at 2025-01-01 (magnitude 0.42)
on n_categories — likely seasonal. The July shift warrants
compare_time_windows to identify root cause."

---

## Alert Types — Example

**Call:** `check_alerts()`

**Output (key fields):**
```json
{
  "alerts": [
    {
      "type": "anomaly_rate_spike",
      "severity": "high",
      "pattern_id": "supplier_pattern",
      "message": "Anomaly rate 18.2% exceeds baseline 9.4% (1.94x)",
      "suggested_action": "anomaly_summary(pattern_id='supplier_pattern')"
    },
    {
      "type": "calibration_drift",
      "severity": "medium",
      "pattern_id": "invoice_pattern",
      "message": "Population mu shifted 0.34 sigma since last calibration",
      "suggested_action": "recalibrate(pattern_id='invoice_pattern')"
    },
    {
      "type": "theta_miscalibration",
      "severity": "high",
      "pattern_id": "customer_pattern",
      "message": "Theta captures 22% of population (expected 5-15%)",
      "suggested_action": "recalibrate(pattern_id='customer_pattern')"
    },
    {
      "type": "regime_changepoint",
      "severity": "medium",
      "pattern_id": "supplier_pattern",
      "message": "Changepoint detected at 2025-01-01 (magnitude 1.34)",
      "suggested_action": "find_regime_changes(pattern_id='supplier_pattern')"
    }
  ]
}
```

**What to look for:**
- `severity=high` alerts = act immediately
- `anomaly_rate_spike`: rate nearly doubled — population composition changed
  or calibration stale. Run `anomaly_summary` then `contrast_populations`
  to find what's driving the spike.
- `calibration_drift`: mu shifted — results still usable but getting less
  accurate. Note for next rebuild cycle.
- `theta_miscalibration`: theta captures 22% of population — anomaly
  detection is too permissive. `recalibrate` immediately or anomaly results
  are unreliable.
- `regime_changepoint`: population structure changed — investigate with
  `find_regime_changes` to understand what shifted and when.

**Finding:** "2 high-severity alerts: anomaly rate spike on supplier_pattern
(18.2% vs 9.4% baseline) and theta miscalibration on customer_pattern
(22% flagged, expected 5-15%). Immediate action: recalibrate customer_pattern,
then investigate supplier anomaly rate spike via contrast_populations."
