# Detective Recipes — Output Examples

Concrete tool output examples for every detection recipe. Each shows what
to call, what the output looks like, and what to write in the report.

---

## Event anomaly rate — Example

Catches multi-dimensional joint deviations invisible to single-column profiling.

**Step 1:** Two aggregate calls on the same event pattern.

```
aggregate(pattern_id="ev_orders", group_by_line="suppliers",
          geometry_filters={"is_anomaly": true}, limit=50)
```
```json
{
  "results": [
    {"key": "SUPP-1234", "count": 47},
    {"key": "SUPP-5678", "count": 31},
    {"key": "SUPP-9012", "count": 28}
  ],
  "total_found": 3
}
```

```
aggregate(pattern_id="ev_orders", group_by_line="suppliers", limit=50)
```
```json
{
  "results": [
    {"key": "SUPP-1234", "count": 120},
    {"key": "SUPP-5678", "count": 310},
    {"key": "SUPP-9012", "count": 95}
  ],
  "total_found": 3
}
```

**What to look for:** Compute `rate = anomalous_count / total_count` per entity.

| Entity | Anomalous | Total | Rate |
|--------|-----------|-------|------|
| SUPP-1234 | 47 | 120 | **39.2%** |
| SUPP-5678 | 31 | 310 | 10.0% |
| SUPP-9012 | 28 | 95 | **29.5%** |

Baseline is ~5%. Anything above 15% is suspect.

- SUPP-1234 at 39.2% — nearly 8x baseline. Strong signal.
- SUPP-9012 at 29.5% — 6x baseline. Strong signal.
- SUPP-5678 at 10.0% — elevated but below threshold. Monitor only.

**Finding:** "SUPP-1234 has 39.2% event anomaly rate (47/120 events anomalous, baseline ~5%). SUPP-9012 has 29.5% (28/95). Both warrant root cause investigation via dive_solid + find_counterparties."

---

## Composite subgroup analysis (Simpson's paradox) — Example

Catches entities that look normal in aggregate but inflate specific subgroups.

**Step 1:** Dual ranking on a composite pattern.

```
aggregate_anomalies(pattern_id="comp_line_items", group_by="supplier_key", limit=50)
```
```json
{
  "results": [
    {"key": "SUPP-3456", "count": 7, "anomaly_rate": 0.23},
    {"key": "SUPP-7890", "count": 4, "anomaly_rate": 0.18},
    {"key": "SUPP-1111", "count": 2, "anomaly_rate": 0.15}
  ]
}
```

```
find_anomalies(pattern_id="comp_line_items", rank_by_property="avg_price_dim", top_n=50)
```
```json
{
  "results": [
    {"key": "COMP-A1", "supplier_key": "SUPP-2222", "avg_price_dim": 4.82, "delta_norm": 1.21},
    {"key": "COMP-A2", "supplier_key": "SUPP-2222", "avg_price_dim": 4.67, "delta_norm": 0.98},
    {"key": "COMP-A3", "supplier_key": "SUPP-3456", "avg_price_dim": 4.51, "delta_norm": 1.45}
  ]
}
```

**What to look for:**
- From Call A: SUPP-3456 has 7 anomalous composites, SUPP-7890 has 4. Both above threshold of 2.
- From Call B: SUPP-2222 appears twice in the value-ranked list but was NOT in Call A's count ranking. This is the Simpson's paradox case — few anomalous composites but extreme per-unit values.
- Union of both rankings is the suspect set: SUPP-3456, SUPP-7890, SUPP-1111, SUPP-2222.

**Finding:** "SUPP-2222 has only 2 anomalous composites (below count threshold) but both rank in top-3 by avg_price_dim (4.82, 4.67). Subgroup price inflation pattern — invisible to count-based aggregation. SUPP-3456 flagged by both methods (7 anomalous composites + high avg_price_dim)."

---

## Temporal burst detection — Example

Detects event splitting or seasonal spikes by comparing yearly windows.

**Step 1:** Get temporal range, then aggregate per year.

```
aggregate(pattern_id="ev_orders", group_by_line="suppliers",
          time_from="2022-01-01", time_to="2023-01-01", limit=50)
```
```json
{
  "results": [
    {"key": "SUPP-1234", "count": 28},
    {"key": "SUPP-5678", "count": 45}
  ]
}
```

```
aggregate(pattern_id="ev_orders", group_by_line="suppliers",
          time_from="2023-01-01", time_to="2024-01-01", limit=50)
```
```json
{
  "results": [
    {"key": "SUPP-1234", "count": 72},
    {"key": "SUPP-5678", "count": 48}
  ]
}
```

```
aggregate(pattern_id="ev_orders", group_by_line="suppliers",
          time_from="2024-01-01", time_to="2025-01-01", limit=50)
```
```json
{
  "results": [
    {"key": "SUPP-1234", "count": 30},
    {"key": "SUPP-5678", "count": 42}
  ]
}
```

**What to look for:** Compare counts across years per entity.

| Entity | 2022 | 2023 | 2024 | Spike? |
|--------|------|------|------|--------|
| SUPP-1234 | 28 | **72** | 30 | Yes — 2.6x in 2023 |
| SUPP-5678 | 45 | 48 | 42 | No — stable |

SUPP-1234 shows a 2.6x spike in 2023 vs adjacent years. This is a burst.

**Step 2:** Drill down into quarters of the spike year.

```
aggregate(pattern_id="ev_orders", group_by_line="suppliers",
          time_from="2023-07-01", time_to="2023-10-01", limit=50)
```
```json
{
  "results": [
    {"key": "SUPP-1234", "count": 38}
  ]
}
```

38 of 72 events concentrated in Q3 2023. The burst is localized to a single quarter.

**Finding:** "SUPP-1234 shows a 2.6x event burst in 2023 (72 vs baseline ~29), concentrated in Q3 2023 (38/72 events). Consistent with event splitting or a one-time bulk activity. Cross-referenced with event_rate_divergence_alerts — entity present in alerts, confirming high-confidence signal."

---

## Neighbor contamination detection — Example

Finds normal entities surrounded by anomalous geometric neighbors.

**Step 1:** Start from a known anomalous entity and find its neighbors.

```
find_similar_entities(entity_key="SUPP-1234", pattern_id="pt_suppliers", top_n=10)
```
```json
{
  "results": [
    {"key": "SUPP-4001", "distance": 0.12, "is_anomaly": false},
    {"key": "SUPP-4002", "distance": 0.15, "is_anomaly": true},
    {"key": "SUPP-4003", "distance": 0.18, "is_anomaly": false},
    {"key": "SUPP-4004", "distance": 0.21, "is_anomaly": true},
    {"key": "SUPP-4005", "distance": 0.22, "is_anomaly": true}
  ]
}
```

**What to look for:** SUPP-4001 and SUPP-4003 are NORMAL but sitting near anomalous entities. Check their neighborhoods.

**Step 2:** Check the normal neighbor's own neighborhood.

```
find_similar_entities(entity_key="SUPP-4001", pattern_id="pt_suppliers", top_n=10)
```
```json
{
  "results": [
    {"key": "SUPP-1234", "distance": 0.12, "is_anomaly": true},
    {"key": "SUPP-4002", "distance": 0.14, "is_anomaly": true},
    {"key": "SUPP-4010", "distance": 0.16, "is_anomaly": true},
    {"key": "SUPP-4011", "distance": 0.17, "is_anomaly": true},
    {"key": "SUPP-4012", "distance": 0.19, "is_anomaly": true},
    {"key": "SUPP-4013", "distance": 0.20, "is_anomaly": false},
    {"key": "SUPP-4014", "distance": 0.22, "is_anomaly": true},
    {"key": "SUPP-4015", "distance": 0.23, "is_anomaly": true},
    {"key": "SUPP-4016", "distance": 0.25, "is_anomaly": true},
    {"key": "SUPP-4017", "distance": 0.26, "is_anomaly": false}
  ]
}
```

**What to look for:** Count anomalous neighbors: 8 out of 10 are anomalous (80%). Threshold is >50%. SUPP-4001 is a confirmed contamination target.

**Step 3:** Verify with batch check.

```
check_anomaly_batch(keys=["SUPP-1234","SUPP-4002","SUPP-4010","SUPP-4011",
    "SUPP-4012","SUPP-4014","SUPP-4015","SUPP-4016"], pattern_id="pt_suppliers")
```
```json
{
  "results": [
    {"key": "SUPP-1234", "is_anomaly": true},
    {"key": "SUPP-4002", "is_anomaly": true},
    {"key": "SUPP-4010", "is_anomaly": true},
    {"key": "SUPP-4011", "is_anomaly": true},
    {"key": "SUPP-4012", "is_anomaly": true},
    {"key": "SUPP-4014", "is_anomaly": true},
    {"key": "SUPP-4015", "is_anomaly": true},
    {"key": "SUPP-4016", "is_anomaly": true}
  ],
  "anomalous_count": 8,
  "total": 8
}
```

All 8 confirmed anomalous. SUPP-4001 (itself normal) sits in a neighborhood that is 80% anomalous.

**Finding:** "SUPP-4001 is geometrically normal (is_anomaly=false, delta_rank_pct=45) but 8 of its 10 nearest neighbors are anomalous (80%). This is a contamination target — the entity's own geometry is shifting toward the anomalous cluster but has not yet crossed the theta threshold. High monitoring priority."

---

## Trajectory shape analysis — Example

Catches non-linear temporal trajectories (arch, V-shape) invisible to displacement ranking.

**Step 1:** Find entities with moderate displacement.

```
find_drifting_entities(pattern_id="pt_suppliers", top_n=50)
```

Pick entities at rank 20-40 (moderate displacement, not top-10).

**Step 2:** Dive into temporal history.

```
dive_solid(entity_key="SUPP-2345", pattern_id="pt_suppliers")
```
```json
{
  "slices": [
    {"period": "2020", "delta_norm": 0.45, "dominant_dim": "price_dim", "delta_price_dim": +0.32},
    {"period": "2021", "delta_norm": 0.89, "dominant_dim": "price_dim", "delta_price_dim": +0.71},
    {"period": "2022", "delta_norm": 1.42, "dominant_dim": "price_dim", "delta_price_dim": +1.15},
    {"period": "2023", "delta_norm": 0.78, "dominant_dim": "price_dim", "delta_price_dim": +0.55},
    {"period": "2024", "delta_norm": 0.38, "dominant_dim": "price_dim", "delta_price_dim": +0.22}
  ],
  "displacement": 0.15
}
```

**What to look for:** The delta values rise (0.45 -> 1.42) then fall (1.42 -> 0.38). This is an **arch shape** — the entity deviated significantly in 2022 then reverted. Displacement is near-zero (0.15) because start and end are similar, masking the trajectory completely.

Compare with a linear drift entity:
```
dive_solid(entity_key="SUPP-6789", pattern_id="pt_suppliers")
```
```json
{
  "slices": [
    {"period": "2020", "delta_norm": 0.30, "delta_price_dim": +0.20},
    {"period": "2021", "delta_norm": 0.55, "delta_price_dim": +0.40},
    {"period": "2022", "delta_norm": 0.82, "delta_price_dim": +0.65},
    {"period": "2023", "delta_norm": 1.10, "delta_price_dim": +0.88},
    {"period": "2024", "delta_norm": 1.35, "delta_price_dim": +1.10}
  ],
  "displacement": 1.20
}
```

This entity has monotonically increasing deltas and high displacement (1.20). This is ordinary linear drift — find_drifting_entities already catches it. The arch-shape entity is the interesting one.

**Step 3:** Find the cohort sharing the arch trajectory.

```
find_drifting_similar(entity_key="SUPP-2345", pattern_id="pt_suppliers", top_n=20)
```
```json
{
  "results": [
    {"key": "SUPP-2345", "similarity": 1.0},
    {"key": "SUPP-2350", "similarity": 0.92},
    {"key": "SUPP-2411", "similarity": 0.88},
    {"key": "SUPP-2502", "similarity": 0.85}
  ]
}
```

4 entities share the arch trajectory shape.

**Finding:** "SUPP-2345, SUPP-2350, SUPP-2411, SUPP-2502 share an arch-shaped trajectory on price_dim — deviation peaked in 2022 (delta_norm 1.42) then reverted by 2024 (0.38). All-time displacement is near-zero (0.15), making them invisible to top-N drift ranking. This suggests a temporary pricing anomaly in 2022 that self-corrected."

---

## Population segment shift — Example

Identifies WHICH population segment is responsible for a detected regime change.

**Step 1:** Detect the regime change.

```
find_regime_changes(pattern_id="pt_suppliers", n_regimes=5)
```
```json
{
  "changepoints": [
    {"date": "2023-06-15", "magnitude": 2.31, "dimension": "cost_dim"}
  ]
}
```

**What to look for:** A single changepoint with magnitude > 1.0 is significant. The shift is on cost_dim.

**Step 2:** Segment analysis — check each property dimension.

```
find_anomalies(pattern_id="pt_suppliers",
               property_filters={"region": "EUROPE"}, limit=50)
```
```json
{
  "results": [
    {"key": "SUPP-E001", "delta_norm": 2.1},
    {"key": "SUPP-E002", "delta_norm": 1.9}
  ],
  "total_found": 2
}
```

```
find_anomalies(pattern_id="pt_suppliers",
               property_filters={"region": "ASIA"}, limit=50)
```
```json
{
  "results": [
    {"key": "SUPP-A001", "delta_norm": 3.2},
    {"key": "SUPP-A002", "delta_norm": 2.8},
    {"key": "SUPP-A003", "delta_norm": 2.5},
    {"key": "SUPP-A004", "delta_norm": 2.1},
    {"key": "SUPP-A005", "delta_norm": 1.9},
    {"key": "SUPP-A006", "delta_norm": 1.7}
  ],
  "total_found": 6
}
```

**What to look for:** ASIA has 6 anomalous suppliers vs EUROPE's 2. The regime change is concentrated in the ASIA segment.

**Step 3:** Confirm with time window comparison.

```
compare_time_windows(pattern_id="pt_suppliers",
    pre_start="2023-01-01", pre_end="2023-06-15",
    post_start="2023-06-15", post_end="2024-01-01")
```
```json
{
  "shifted_dimensions": [
    {"dimension": "cost_dim", "cohens_d": 0.85, "direction": "increase"}
  ]
}
```

Cohen's d of 0.85 on cost_dim confirms a large effect size in the expected dimension.

**Finding:** "Regime change detected on 2023-06-15 (magnitude 2.31). Concentrated in ASIA segment — 6 anomalous suppliers vs 2 in EUROPE. cost_dim shifted upward (Cohen's d = 0.85). 6 affected entities: SUPP-A001 through SUPP-A006."

---

## Collective drift — Example

Detects gradual population-wide shifts that accumulate over time.

**Step 1:** Find drifting entities.

```
find_drifting_entities(pattern_id="pt_pricing", window=365, top_n=10)
```
```json
{
  "results": [
    {"key": "PROD-101", "displacement": 2.45, "drift_score": 3.1},
    {"key": "PROD-102", "displacement": 2.38, "drift_score": 2.9},
    {"key": "PROD-103", "displacement": 2.30, "drift_score": 2.8},
    {"key": "PROD-104", "displacement": 2.25, "drift_score": 2.7}
  ]
}
```

**What to look for:** Multiple entities with similar high displacement AND similar drift_score suggests collective drift (not individual anomalies). All displacement values cluster tightly (2.25-2.45).

**Step 2:** Verify monotonic trajectory.

```
dive_solid(entity_key="PROD-101", pattern_id="pt_pricing")
```
```json
{
  "slices": [
    {"period": "2021", "delta_norm": 0.40, "delta_cost_dim": +0.30},
    {"period": "2022", "delta_norm": 0.85, "delta_cost_dim": +0.65},
    {"period": "2023", "delta_norm": 1.35, "delta_cost_dim": +1.05},
    {"period": "2024", "delta_norm": 1.90, "delta_cost_dim": +1.50}
  ],
  "displacement": 2.45
}
```

**What to look for:** Monotonically increasing delta values across ALL slices with consistent direction (+cost_dim in every period). This is gradual drift, not a step change.

**Step 3:** Confirm no regime changepoint.

```
find_regime_changes(pattern_id="pt_pricing")
```
```json
{
  "changepoints": []
}
```

No changepoint detected. This confirms gradual collective drift (not a regime step change).

Compare with a NON-drifting entity for contrast:
```
dive_solid(entity_key="PROD-500", pattern_id="pt_pricing")
```
```json
{
  "slices": [
    {"period": "2021", "delta_norm": 0.15, "delta_cost_dim": +0.05},
    {"period": "2022", "delta_norm": 0.18, "delta_cost_dim": -0.02},
    {"period": "2023", "delta_norm": 0.12, "delta_cost_dim": +0.08},
    {"period": "2024", "delta_norm": 0.20, "delta_cost_dim": -0.03}
  ],
  "displacement": 0.08
}
```

Flat, oscillating deltas with near-zero displacement. This entity is stable.

**Finding:** "Collective pricing drift detected: PROD-101 through PROD-104 show monotonically increasing cost_dim deviation over 4 years (delta_norm 0.40 -> 1.90). No regime changepoint — this is gradual accumulation. Displacement values cluster at 2.25-2.45. Non-drifting entities show flat trajectories (displacement <0.10). The drift cohort is systematically repricing upward."

---

## Multi-pattern investigation — Example

Catches entities that escape detection in their primary pattern but are flagged by other patterns.

**Step 1:** Cross-pattern profile.

```
cross_pattern_profile(key="SUPP-5678", line_id="suppliers")
```
```json
{
  "entity_key": "SUPP-5678",
  "patterns": [
    {"pattern_id": "pt_suppliers", "is_anomaly": false, "delta_norm": 0.72},
    {"pattern_id": "ev_orders", "is_anomaly": true, "anomalous_event_rate": 0.22},
    {"pattern_id": "comp_line_items", "is_anomaly": true, "anomalous_composites": 4}
  ],
  "source_count": 2
}
```

**What to look for:**
- `source_count >= 2` — entity is flagged by multiple patterns. Strong signal.
- SUPP-5678 is NORMAL in pt_suppliers (delta_norm 0.72, below theta) but ANOMALOUS in both ev_orders (22% event rate) and comp_line_items (4 anomalous composites).
- The anchor pattern misses this entity entirely. Only cross-pattern analysis catches it.

**Step 2:** Check composite risk for borderline entities.

```
composite_risk(key="SUPP-5678", line_id="suppliers")
```
```json
{
  "entity_key": "SUPP-5678",
  "combined_p": 0.003,
  "individual_p_values": [
    {"pattern_id": "pt_suppliers", "p": 0.18},
    {"pattern_id": "ev_orders", "p": 0.02},
    {"pattern_id": "comp_line_items", "p": 0.04}
  ]
}
```

**What to look for:** `combined_p < 0.05` via Fisher's method. Each individual pattern shows borderline or non-significant p-values, but combined they are highly significant (p = 0.003).

**Finding:** "SUPP-5678 is normal in anchor pattern (delta_norm 0.72) but flagged by 2 other patterns: 22% event anomaly rate + 4 anomalous composites. Fisher's combined p-value = 0.003. Multi-pattern convergence confirms this entity as anomalous despite passing the primary pattern's theta threshold."

---

## passive_scan — Example

Population-level screening for confirmation, not primary discovery.

**Step 1:** Broad scan at threshold=1.

```
passive_scan(line_id="suppliers", threshold=1)
```
```json
{
  "results": [
    {"key": "SUPP-1234", "source_count": 3, "sources": ["pt_suppliers", "ev_orders", "comp_line_items"]},
    {"key": "SUPP-5678", "source_count": 2, "sources": ["ev_orders", "comp_line_items"]},
    {"key": "SUPP-9012", "source_count": 1, "sources": ["pt_suppliers"]},
    {"key": "SUPP-3456", "source_count": 1, "sources": ["comp_line_items"]}
  ],
  "total_found": 4
}
```

**What to look for:** All entities flagged by at least 1 pattern. This is the widest net — includes single-source entities that may be false positives.

**Step 2:** Narrow to confirmed multi-source at threshold=2.

```
passive_scan(line_id="suppliers", threshold=2)
```
```json
{
  "results": [
    {"key": "SUPP-1234", "source_count": 3, "sources": ["pt_suppliers", "ev_orders", "comp_line_items"]},
    {"key": "SUPP-5678", "source_count": 2, "sources": ["ev_orders", "comp_line_items"]}
  ],
  "total_found": 2
}
```

**What to look for:**
- threshold=2 filters to entities confirmed by 2+ independent patterns. These are high-confidence anomalies.
- SUPP-1234 at source_count=3 (all patterns agree) is the strongest signal.
- SUPP-5678 at source_count=2 (event + composite) — note it was NORMAL in the anchor pattern, caught only by cross-pattern convergence.
- SUPP-9012 and SUPP-3456 dropped out — single-source only, lower confidence.

**Finding:** "passive_scan confirms 2 multi-source anomalies: SUPP-1234 (3 sources — strongest signal in the sphere) and SUPP-5678 (2 sources — invisible to anchor pattern alone). 2 additional entities flagged by single source only (lower confidence, monitor)."
