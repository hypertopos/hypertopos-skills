# GDS Explorer — Output Examples

Concrete output examples for each exploration recipe. Shows what the tool
returns and how to interpret it.

## sphere_overview — Example

**Call:** `sphere_overview(detail="summary")`

**Output (key fields):**
```json
{
  "sphere_id": "gds_procurement",
  "patterns": [
    {
      "pattern_id": "supplier_pattern",
      "entity_line": "suppliers",
      "total_entities": 1842,
      "anomaly_count": 168,
      "anomaly_rate": 0.091,
      "geometry_mode": "continuous",
      "calibration_health": "good",
      "has_temporal": true
    },
    {
      "pattern_id": "invoice_pattern",
      "entity_line": "invoices",
      "total_entities": 48210,
      "anomaly_count": 2891,
      "anomaly_rate": 0.060,
      "geometry_mode": "continuous",
      "calibration_health": "good",
      "has_temporal": false
    }
  ],
  "profiling_alerts": [
    {
      "pattern_id": "supplier_pattern",
      "alert": "dimension 'avg_late_days' has kurtosis 12.4 — heavy tail, extreme values may be ranked low by delta_norm",
      "suggested_action": "find_anomalies(pattern_id='supplier_pattern', rank_by_property='avg_late_days')"
    }
  ]
}
```

**What to look for:**
- `anomaly_rate` in 5-15% range = healthy calibration. Below 3% = theta too
  strict (missing anomalies). Above 20% = theta too permissive (too many
  false positives). Either way, `recalibrate` before trusting results.
- `calibration_health = poor` = recalibrate immediately, anomaly results
  are unreliable until fixed.
- `geometry_mode = binary` = skip delta-based anomaly tools entirely, use
  `find_hubs` and `get_centroid_map` instead.
- `has_temporal = true` = drift/regime detection available for this pattern.
- `profiling_alerts` = EXECUTE EVERY suggested action. These catch signals
  that default ranking misses (heavy tails, bimodal distributions).

**Finding:** "Sphere has 2 patterns. supplier_pattern: 9.1% anomaly rate,
good calibration, temporal data available. Profiling alert on avg_late_days
(heavy tail) — need rank_by_property check. invoice_pattern: 6.0% rate,
good calibration, no temporal data."

---

## get_sphere_info — Example

**Call:** `get_sphere_info()`

**Output (key fields):**
```json
{
  "sphere_id": "gds_procurement",
  "lines": [
    {
      "line_id": "suppliers",
      "entity_count": 1842,
      "key_column": "supplier_id",
      "properties": ["supplier_id", "name", "country", "category", "status", "risk_tier"],
      "fts_available": true
    },
    {
      "line_id": "invoices",
      "entity_count": 48210,
      "key_column": "invoice_id",
      "properties": ["invoice_id", "supplier_id", "amount", "date", "status", "days_late"]
    }
  ],
  "patterns": ["supplier_pattern", "invoice_pattern"],
  "aliases": [
    {"alias_id": "high_risk_suppliers", "pattern_id": "supplier_pattern", "has_cutting_plane": true}
  ]
}
```

**What to look for:**
- `properties` list: look for ground truth columns (status, is_default,
  churn_flag, fraud_flag, risk_tier). If found, STOP exploring and hand
  off to gds-investigator for ground truth validation.
- `fts_available = true` = full-text search enabled, use
  `search_entities_fts` for name/text lookups.
- `aliases` with `has_cutting_plane = true` = MANDATORY: run
  `attract_boundary` in BOTH directions (in/out) on every such alias.
- Multiple patterns on the same entity line = check each independently,
  signal dilution risk.
- Lines sharing the same `key_column` type = potential for cross-line
  analysis via `cross_pattern_profile`.

**Finding:** "Sphere has 2 lines: suppliers (1842, FTS enabled) and invoices
(48210). Risk_tier property on suppliers = ground truth available, hand off
to investigator. 1 alias (high_risk_suppliers) with cutting plane — run
boundary analysis both directions."

---

## find_clusters — Example

**Call:** `find_clusters(pattern_id="supplier_pattern", n_clusters=5, sample_size=5000)`

**Output (key fields):**
```json
{
  "pattern_id": "supplier_pattern",
  "n_clusters": 5,
  "total_clustered": 1842,
  "clusters": [
    {
      "cluster_id": 0,
      "size": 892,
      "pct": 0.484,
      "centroid": {"total_spend": -0.21, "n_invoices": -0.34, "avg_late_days": -0.08, "n_categories": -0.15},
      "anomaly_rate": 0.032,
      "label": "low-activity baseline"
    },
    {
      "cluster_id": 1,
      "size": 412,
      "pct": 0.224,
      "centroid": {"total_spend": 0.88, "n_invoices": 1.12, "avg_late_days": -0.31, "n_categories": 0.67},
      "anomaly_rate": 0.078,
      "label": "high-volume punctual"
    },
    {
      "cluster_id": 2,
      "size": 287,
      "pct": 0.156,
      "centroid": {"total_spend": 0.34, "n_invoices": 0.45, "avg_late_days": 2.41, "n_categories": 0.12},
      "anomaly_rate": 0.314,
      "label": "late-payment cluster"
    },
    {
      "cluster_id": 3,
      "size": 156,
      "pct": 0.085,
      "centroid": {"total_spend": 2.67, "n_invoices": 2.89, "avg_late_days": -0.42, "n_categories": 1.94},
      "anomaly_rate": 0.423,
      "label": "mega-suppliers"
    },
    {
      "cluster_id": 4,
      "size": 95,
      "pct": 0.052,
      "centroid": {"total_spend": -0.89, "n_invoices": -0.91, "avg_late_days": -0.03, "n_categories": -0.78},
      "anomaly_rate": 0.168,
      "label": "minimal-activity"
    }
  ]
}
```

**What to look for:**
- Centroid values are in delta space (standard deviations from population
  mean). Values > 1.0 or < -1.0 = cluster is significantly different from
  population on that dimension.
- Cluster with high `anomaly_rate` (cluster 2: 31.4%, cluster 3: 42.3%)
  = archetypes that concentrate anomalies. Investigate these first.
- Largest cluster (cluster 0: 48.4%) = population baseline. Low anomaly
  rate confirms it represents "normal" behavior.
- Small cluster with extreme centroids (cluster 3: 8.5% of population,
  total_spend=2.67) = niche archetype, often the most interesting.
- Compare anomaly rates across clusters — if only 1 cluster concentrates
  anomalies, the signal is clean. If anomalies are spread evenly,
  the pattern may not discriminate well.

**Finding:** "5 archetypes identified. Cluster 2 (late-payment, 15.6%
of population) concentrates 31.4% anomaly rate — avg_late_days centroid
2.41 sigma above mean. Cluster 3 (mega-suppliers, 8.5%) has 42.3% anomaly
rate but these are large legitimate suppliers. Cluster 0 (48.4%) is the
healthy baseline at 3.2% anomaly rate."

---

## get_centroid_map — Example

**Call:** `get_centroid_map(pattern_id="supplier_pattern", group_by_line="categories")`

**Output (key fields):**
```json
{
  "pattern_id": "supplier_pattern",
  "group_by_line": "categories",
  "groups": [
    {
      "group_key": "raw_materials",
      "count": 612,
      "centroid": {"total_spend": 1.21, "n_invoices": 0.89, "avg_late_days": -0.34},
      "anomaly_rate": 0.082
    },
    {
      "group_key": "services",
      "count": 534,
      "centroid": {"total_spend": -0.45, "n_invoices": 0.12, "avg_late_days": 0.67},
      "anomaly_rate": 0.112
    },
    {
      "group_key": "equipment",
      "count": 389,
      "centroid": {"total_spend": 0.78, "n_invoices": -0.56, "avg_late_days": -0.21},
      "anomaly_rate": 0.054
    },
    {
      "group_key": "consulting",
      "count": 307,
      "centroid": {"total_spend": -0.89, "n_invoices": -0.67, "avg_late_days": 1.45},
      "anomaly_rate": 0.195
    }
  ]
}
```

**What to look for:**
- Groups with anomaly_rate significantly above population average (9.1%)
  = subpopulations where the pattern fits poorly or genuine risk
  concentration.
- Consulting (19.5% anomaly rate) vs equipment (5.4%) = pattern treats
  consulting suppliers as more anomalous — could be real risk or
  subpopulation mismatch (consulting has different geometry).
- Centroid differences reveal group structure: raw_materials = high spend,
  many invoices, paid on time. Consulting = low spend, few invoices,
  chronically late. Equipment = high spend, few invoices, on time.
- If one group's centroid is extreme on multiple dimensions = candidate
  for entity line isolation (separate pattern for that subpopulation).

**Finding:** "Supplier population segments by category. Consulting suppliers
(16.7% of population) have 19.5% anomaly rate — their centroid shows
low volume but high late-payment tendency (avg_late_days=1.45 sigma).
Equipment suppliers have lowest anomaly rate (5.4%). Consider isolated
pattern for consulting subpopulation if recall is poor."

---

## attract_boundary — Example

**Call:** `attract_boundary(alias_id="high_risk_suppliers", pattern_id="supplier_pattern", direction="in", top_n=10)`

**Output (key fields):**
```json
{
  "alias_id": "high_risk_suppliers",
  "direction": "in",
  "total_inside": 234,
  "results": [
    {"key": "SUPP-3456", "signed_distance": 0.08,  "is_anomaly": false},
    {"key": "SUPP-7890", "signed_distance": 0.12,  "is_anomaly": false},
    {"key": "SUPP-2345", "signed_distance": 0.19,  "is_anomaly": true}
  ]
}
```

**Call:** `attract_boundary(alias_id="high_risk_suppliers", pattern_id="supplier_pattern", direction="out", top_n=10)`

**Output (key fields):**
```json
{
  "alias_id": "high_risk_suppliers",
  "direction": "out",
  "total_outside": 1608,
  "results": [
    {"key": "SUPP-4567", "signed_distance": -0.05, "is_anomaly": false},
    {"key": "SUPP-6789", "signed_distance": -0.11, "is_anomaly": false},
    {"key": "SUPP-8901", "signed_distance": -0.14, "is_anomaly": false}
  ]
}
```

**What to look for:**
- `signed_distance` near 0 = entity is right at the boundary. Positive
  = inside the alias segment, negative = outside.
- `direction="in"` results: entities INSIDE the alias closest to the
  boundary — these could fall out with small changes. Watchlist candidates.
- `direction="out"` results: entities OUTSIDE the alias closest to the
  boundary — these could enter with small changes. Early warning.
- SUPP-3456 (distance=0.08) is barely inside — a small behavioral change
  could move it outside the high-risk segment.
- SUPP-4567 (distance=-0.05) is barely outside — trending toward high-risk.
- Always run BOTH directions on every alias with `has_cutting_plane=true`.

**Finding:** "High-risk boundary analysis: 234 suppliers inside alias.
SUPP-3456 is the most marginal insider (distance=0.08) — minor behavioral
change could reclassify. SUPP-4567 is closest outsider (distance=-0.05) —
nearly high-risk, early warning candidate. 3 entities within 0.15 of
boundary in each direction."
