# Decision Framework — Anomaly Type to Tool Mapping

Full mapping used by gds-analyst Step 2T. For each anomaly category, which
tools to run, in which order, and what to look for.

---

## 1. Cross-pattern discrepancy

**Signal:** entity anomalous in ONE pattern but normal in another.
Opposite of multi-source confirmation.

**When to use:** instruction mentions "cross-pattern", "discrepancy",
"anomalous in pricing but normal in activity", or similar.

```
Step 1: passive_scan(line_id, threshold=1)
        → all entities flagged by at least one pattern
        → filter to source_count == 1

Step 2: For each source_count=1 entity:
        cross_pattern_profile(key, line_id)
        → which pattern flags it? which says normal?

Step 3: explain_anomaly(key, flagging_pattern_id)
        → which dimension drives it?

Step 4: For borderline entities (elevated but not flagged):
        composite_risk(key, line_id)
        → Fisher's method p-value — combined_p < 0.05 = significant
```

**What you're looking for:** supplier anomalous in pricing but NORMAL in
order_frequency means pricing anomaly is not driven by volume — isolated
deviation. Different finding type from multi-source.

**Common mistake:** using `threshold=2` (finds agreement). You need
`threshold=1` for discrepancy.

---

## 2. Geometric neighborhood contamination

**Signal:** normal entities whose geometric neighbors are systematically
anomalous. The target is NORMAL — invisible to anomaly scans.

**When to use:** instruction mentions "neighborhood", "contamination",
"guilt by association", or similar.

```
Approach A — from anomalous outward (cheaper, do first):
Step 1: find_anomalies(pattern, top_n=50) → top anomalous entities
Step 2: For 5-10 anomalous entities:
        find_similar_entities(anom_key, pattern, top_n=10)
Step 3: check_anomaly_batch(neighbor_keys, pattern)
        → normal neighbor with >50% anomalous neighbors = candidate
Step 4: For each normal candidate:
        find_similar_entities(normal_key, pattern, top_n=10)
        check_anomaly_batch → confirm >50% threshold

Approach B — from population sample (if A finds nothing):
Step 1: Walk line, sample 30-50 normal entities (delta_rank_pct 30-70)
Step 2: For each normal entity:
        find_similar_entities(key, pattern, top_n=10)
        check_anomaly_batch → same >50% threshold
```

**What you're looking for:** CUST-X is_anomaly=false, delta_rank_pct=45,
but 7 of its 10 nearest geometric neighbors are anomalous. The entity is
being dragged toward anomalous territory without crossing the threshold yet.

**Minimum calls:** 10 `find_similar_entities` calls across both approaches.

---

## 3. Temporal trajectory shape

**Signal:** non-linear temporal trajectory (arch, V-shape, spike-recovery)
with near-zero displacement — invisible to drift ranking.

**When to use:** instruction mentions "trajectory", "arch", "V-shape",
"behavioral change pattern", "spike then recovery", or "temporal shape".

```
Step 1: find_drifting_entities(pattern, top_n=50)
        → sort by displacement, identify 3 displacement ranges:
          - Top 5 (high displacement) → likely linear drift
          - Rank 20-50 (moderate)    → may be non-linear
          - Rank 100-200 (low)       → arch/V candidates

Step 2: For at least one entity from EACH range:
        dive_solid(key, pattern)
        → inspect delta values across slices:
          Monotonic increase   = linear drift (caught by drift ranking)
          Up-then-down (arch)  = trajectory anomaly — LOW displacement
          Down-then-up (V)     = trajectory anomaly — LOW displacement
          Flat                 = no signal, move to next range

Step 3: If arch or V found:
        find_drifting_similar(arch_key, pattern, top_n=20)
        → full cohort sharing the trajectory shape

Step 4: For the cohort:
        get_event_polygons(key) or find_counterparties
        → what happened at the peak/trough of the arch?
```

**Key insight:** arch/V trajectories cancel out over time → near-zero
all-time displacement → they rank LAST in drift ranking. Must actively
search low-displacement range. Never stop at top-10.

**What you're looking for:** entity has displacement=0.1 but dive_solid
shows +3, +4, +3, +1, -3, -4, -3 across slices — arch shape. The entity
changed dramatically but returned to baseline.

---

## 4. Population segment shift

**Signal:** a specific categorical segment (nation, region, category)
shifted collectively — individual entities below theta, group centroid moved.

**When to use:** instruction mentions "segment shift", "population change",
"which group changed", "regime change", or similar.

```
Step 1: find_regime_changes(pattern, n_regimes=5)
        → changepoint at date X with magnitude Y
        → If ALL patterns shift at same timestamp → data boundary artifact
          Do NOT dismiss without segment analysis (step 2 still required)

Step 2: MANDATORY even for suspected artifacts:
        get_line_schema(entity_line) → categorical properties
        For each categorical property with <50 distinct values:
          find_anomalies(pattern,
                         property_filters={"<prop>": "<value>"},
                         limit=50)
        → which segment has disproportionately more anomalies?

Step 3: contrast_populations(pattern,
                              time_window_pre, time_window_post)
        → which dimensions shifted? |d| > 0.8 = large effect.

Step 4: For the affected segment:
        search_entities(entity_line, "<prop>", "<segment_value>")
        → get full key list — list ALL in report
```

**What you're looking for:** regime_changes detects a changepoint in
2020-03. Segment analysis shows GERMANY has 38 anomalies, all other
nations <8. contrast_populations confirms total_purchases dimension
shifted by d=1.4 for GERMANY specifically.

**Common mistake:** detect regime change → attribute to "data boundary"
→ move on. You MUST decompose by segment first.

---

## 5. Event anomaly rate (copula / joint deviation)

**Signal:** entity has elevated event anomaly rate from multi-dimensional
joint deviations — each column individually normal, combination abnormal.

**When to use:** instruction mentions "event rate", "joint deviation",
"copula", "multi-dimensional", or `event_rate_divergence_alerts` are present.

```
Step 1: sphere_overview(detail="full")
        → check event_rate_divergence_alerts
        → pre-computed for top 20 by rate, entities below theta only

Step 2: ALWAYS run manual check too (catches entities above theta):
        CALL A: aggregate(event_pattern, group_by_line=anchor,
                          geometry_filters={"is_anomaly": true}, limit=50)
        CALL B: aggregate(event_pattern, group_by_line=anchor, limit=50)
        → rate = call_a_count / call_b_count per entity
        → >15% = suspect (baseline ~5%), >30% = strong signal

Step 3: For entities with high rate AND low static delta_norm:
        → anomaly concentrated in specific time windows, not aggregate
        → run windowed aggregate to localize:
          aggregate(event_pattern, group_by_line=anchor,
                    geometry_filters={"is_anomaly": true},
                    time_from="YYYY-01-01", time_to="YYYY+1-01-01")
          → identify burst year, then drill to quarter

Step 4: For burst confirmed:
        dive_solid(entity_key, anchor_pattern)
        → temporal history of the anchor during burst period
        cross_pattern_profile(entity_key, line_id)
        → is the high event rate consistent across patterns?
```

**What you're looking for:** SUPPLIER-X has 45 anomalous events out of
180 total = 25% rate. Static delta_norm=0.3 (below theta). Windowed
aggregate shows all anomalous events in Q3-2019 only — burst, not drift.

---

## 6. Composite subgroup (Simpson's paradox)

**Signal:** parent entity has NORMAL aggregate stats but inflates specific
subgroup categories — detected only via composite pattern.

**When to use:** instruction mentions "composite", "subgroup", "Simpson's",
"per-category inflation", or sphere has composite patterns.

```
Step 1: get_sphere_info() → identify composite patterns and their key columns

Step 2: For EACH composite pattern — DUAL ranking (both calls required):
        CALL A: aggregate_anomalies(composite_pattern,
                                    group_by="parent_key_col", limit=50)
                → ranked by COUNT of anomalous composites (concentration)

        CALL B: find_anomalies(composite_pattern,
                               rank_by_property="avg_dim", limit=50)
                → ranked by VALUE metric (per-unit inflation)

        Parents appearing in EITHER ranking = suspect.
        Threshold for CALL A: >=2 anomalous composites (not >=5).
        VALUE ranking catches subgroup inflation that count ranking misses.

Step 3: For top suspects from both rankings:
        find_counterparties(key, event_line, from_col, to_col)
        → which specific subgroup relationships are anomalous?
        → is inflation concentrated in 1-2 categories or spread wide?

Step 4: compare_entities(suspect_key, normal_parent_key, composite_pattern)
        → which subgroup categories differ?
```

**What you're looking for:** SUPPLIER-X has avg_price anomaly in only
2 product categories (BRASS and COPPER) but total volume is normal.
count ranking missed it (only 2 composites), value ranking flagged it
because avg_price is 3x normal for those categories.

---

## 7. No specific hints — exploration sequence

When the instruction gives no specific anomaly categories, run the full
exploration sequence from gds-analyst Exploration mode (Steps 1E-5E).

Exploration budget split: orient 15% → detect 20% → scans 20%
→ root cause 35% → temporal 10%.

**Decision during exploration:**

| sphere_overview signal | Next action |
|---|---|
| profiling_alerts present | Execute suggested call for EACH alert immediately |
| event_rate_divergence_alerts | Flag entities for Step 5E temporal investigation |
| anomaly_rate = 0% on a pattern | Skip geometric scan, use find_hubs + get_centroid_map |
| anomaly_rate > 20% on a pattern | Binary or broad geometry — use aggregate counts |
| has_temporal = true | Phase 5E drift + regime required (not optional) |
| aliases with cutting planes | attract_boundary both directions |

---

## Tool quick reference

| Goal | Tool | Key parameters |
|---|---|---|
| All single-pattern anomalies | `passive_scan` | `threshold=1` |
| Multi-pattern confirmed | `passive_scan` | `threshold=2` |
| Combined p-value | `composite_risk` | default |
| Entity multi-pattern overview | `cross_pattern_profile` | `key, line_id` |
| Geometric neighbors | `find_similar_entities` | `top_n=10` |
| Batch anomaly check | `check_anomaly_batch` | list of keys |
| Temporal slice history | `dive_solid` | `key, pattern` |
| Trajectory cohort | `find_drifting_similar` | `key, pattern, top_n=20` |
| Drift ranking | `find_drifting_entities` | `top_n=10, rank_by_dimension=<dim>` |
| Changepoint | `find_regime_changes` | `n_regimes=5` |
| Pre/post comparison | `compare_time_windows` | `pre_start, pre_end, post_start, post_end` |
| Event count by entity | `aggregate` | `metric="count", time_from, time_to` |
| Anomalous event count | `aggregate` | `geometry_filters={"is_anomaly": true}` |
| Composite concentration | `aggregate_anomalies` | `group_by="parent_key_col"` |
| Property distribution | `find_anomalies` | `rank_by_property="<dim>", limit=50` |
| Segment filtering | `find_anomalies` | `property_filters={"<prop>": "<val>"}` |
| Population contrast | `contrast_populations` | look at max single |d|, not average |
| Root cause dimensions | `explain_anomaly` | note repair set + residual |
| Event relationships | `find_counterparties` | `entity_key, event_line, from_col, to_col` |
| Polygon geometry | `get_polygon` | after `goto(key, line_id)` |
| Segment full list | `search_entities` | `line_id, property, value` |
