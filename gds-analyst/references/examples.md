# Investigation Examples

Concrete examples showing the analyst thought process for targeted and
exploration investigations. Each example shows the hint → scan match →
finding chain.

---

## Example 1 — Targeted: cross-pattern discrepancy

**Instruction:** "Investigate suppliers. Focus on cross-pattern discrepancies —
entities that appear normal in one pattern but anomalous in another."

**Thought process:**

Hint: "cross-pattern discrepancy" → this is Scan 1 from the decision framework.
I need `passive_scan(threshold=1)` to find entities flagged by exactly one
pattern, then `cross_pattern_profile` to see which pattern flags them and which
does not.

```
sphere_overview()
→ patterns: suppliers_pricing (anchor), suppliers_activity (anchor)
→ both cover entity line "suppliers"
→ anomaly_rate: pricing=8%, activity=6%

passive_scan("suppliers", threshold=1)
→ 34 entities with source_count=1
→ 21 flagged by pricing only, 13 flagged by activity only

cross_pattern_profile("SUPP-0042", "suppliers")
→ pricing: is_anomaly=true, delta_norm=2.3
→ activity: is_anomaly=false, delta_rank_pct=47

explain_anomaly("SUPP-0042", "suppliers_pricing")
→ dominant: avg_item_price (+2.1σ)
→ repair: zero 1 dim → residual 0.3 (below theta=1.2)

find_counterparties("SUPP-0042", "orders", "supplier_key", "part_key")
→ 47 orders, avg_price=38.4 (population avg=24.1)
→ concentrated in BRASS parts (avg_price=52.1)
```

**Finding:** SUPP-0042 has elevated pricing anomaly isolated to BRASS part
orders (avg $52 vs $24 population), while its order frequency and volume
are completely normal. This is not a high-volume supplier charging more for
more work — it is a specific price category manipulation.

**21 suppliers flagged by pricing-only** form the full cohort. Contrast with
the 13 activity-only (different anomaly type — do not combine findings).

---

## Example 2 — Targeted: geometric neighborhood

**Instruction:** "Check if any normal customers are embedded in anomalous
neighborhoods — guilt by association."

**Thought process:**

Hint: "geometric neighborhood" → Scan 2. I need to check both directions:
from anomalous entities outward (cheaper) AND sample normal entities.
Minimum 10 `find_similar_entities` calls total.

```
find_anomalies("customers_pattern", top_n=50)
→ 50 anomalous customers, select 5 with highest delta_norm

find_similar_entities("CUST-1187", "customers_pattern", top_n=10)
→ neighbors: [CUST-0023, CUST-0091, CUST-1443, ...]

check_anomaly_batch(["CUST-0023","CUST-0091","CUST-1443",...], "customers_pattern")
→ CUST-0023: is_anomaly=false, delta_rank_pct=38
→ CUST-0091: is_anomaly=true
→ CUST-1443: is_anomaly=false, delta_rank_pct=42
→ 7/10 neighbors anomalous

find_similar_entities("CUST-0023", "customers_pattern", top_n=10)
→ check_anomaly_batch → 8/10 neighbors anomalous
→ CUST-0023 confirmed contamination target

[Repeat for 4 more anomalous seeds — 5 calls total]

[Sample 5 normal entities from delta_rank_pct 30-70]
walk_line("customers", top_n=200) → select 5 at rank_pct 35-65
find_similar_entities(each, "customers_pattern", top_n=10)
check_anomaly_batch → no additional contamination targets found
```

**Finding:** CUST-0023 (is_anomaly=false, delta_rank_pct=38) is surrounded
by 8 anomalous geometric neighbors. Its own geometry has not crossed theta
but its geometric context is saturated with anomalous entities. This is a
leading indicator — monitor for crossing the anomaly boundary.

No additional contamination targets found in normal population sample.

---

## Example 3 — Targeted: temporal trajectory shape

**Instruction:** "Look for non-linear behavioral patterns in the orders data —
I'm specifically interested in entities that spiked and then recovered."

**Thought process:**

Hint: "spiked and recovered" → this is an arch trajectory. Low displacement
(spike and recovery cancel out), non-linear shape. Scan 3 from the decision
framework — must sample LOW displacement ranges, not just top drifters.

```
find_drifting_entities("orders_pattern", top_n=200)
→ 200 entities sorted by displacement
→ identify three ranges for sampling

dive_solid("SUPP-0012", "orders_pattern")    # top-5 displacement
→ slice deltas: +1.2, +1.8, +2.4, +2.9, +3.1 → monotonic increase
→ LINEAR drift, not an arch

dive_solid("SUPP-0074", "orders_pattern")    # rank ~35
→ slice deltas: +0.3, +1.1, +2.8, +1.2, +0.2 → rise-then-fall
→ ARCH SHAPE — spike in slices 3-4, recovery in slice 5

dive_solid("SUPP-0189", "orders_pattern")    # rank ~140
→ slice deltas: +0.1, +2.9, +3.1, +0.4, -0.1 → sharp spike
→ SPIKE-RECOVERY — concentrated in slices 2-3

find_drifting_similar("SUPP-0074", "orders_pattern", top_n=20)
→ 14 entities with arch trajectory, displacement range 0.05-0.4

find_counterparties("SUPP-0074", "orders", "supplier_key", "part_key")
→ filter to slice 3 time window → concentrated orders in specific quarter
```

**Finding:** 14 suppliers show arch-shaped temporal trajectories — they
spiked in a specific quarter then returned to baseline. Total displacement
is near-zero (0.05-0.4), so they rank at positions 120-180 in the drift
ranking and are invisible to top-N drift detection. The spike period coincides
with a specific procurement window — possible bulk-order splitting behavior.

---

## Example 4 — Targeted: event anomaly rate

**Instruction:** "Check if any accounts have concentrated anomalous transaction
periods — multi-dimensional joint deviations we can't see from the account
geometry alone."

**Thought process:**

Hint: "anomalous transaction periods", "joint deviations" → event anomaly rate
recipe. Need both the geometry-filtered aggregate AND total aggregate to compute
per-entity rates.

```
sphere_overview(detail="full")
→ event_rate_divergence_alerts: [ACCT-0042 rate=0.31, ACCT-0117 rate=0.28]
→ these are below-theta accounts with high event anomaly rates

aggregate("transactions_pattern", group_by_line="accounts",
          geometry_filters={"is_anomaly": true}, limit=50)
→ 50 accounts sorted by anomalous transaction count

aggregate("transactions_pattern", group_by_line="accounts", limit=50)
→ total transaction counts

[Compute rate = anomalous/total for each]
→ ACCT-0042: 89/289 = 30.8%  ← pre-flagged in alerts ✓
→ ACCT-0117: 76/271 = 28.0%  ← pre-flagged ✓
→ ACCT-0388: 54/189 = 28.6%  ← NOT in alerts (above theta statically)
→ ACCT-0021: 12/210 = 5.7%   ← baseline, ignore

aggregate("transactions_pattern", group_by_line="accounts",
          geometry_filters={"is_anomaly": true},
          time_from="1998-01-01", time_to="1999-01-01")
→ ACCT-0042: 83/89 anomalous transactions in 1998 → burst year identified

aggregate("transactions_pattern", group_by_line="accounts",
          geometry_filters={"is_anomaly": true},
          time_from="1998-01-01", time_to="1998-04-01")
→ ACCT-0042: 71 in Q1 1998 → burst localized to Q1

dive_solid("ACCT-0042", "accounts_pattern")
→ static geometry confirms no delta shift → temporal-only anomaly
cross_pattern_profile("ACCT-0042", "accounts")
→ source_count=2 → confirmed across patterns
```

**Finding:** 3 accounts with >25% anomalous transaction rate: ACCT-0042 (31%),
ACCT-0388 (29%), ACCT-0117 (28%). ACCT-0388 was NOT in pre-computed alerts
because its static geometry is above theta — the manual rate calculation is
essential. ACCT-0042's burst is localized to Q1-1998 (71 anomalous transactions
out of 89 for the full year). These are temporal-concentration anomalies
invisible to static geometry.

---

## Example 5 — Exploration: no hints given

**Instruction:** "Run a full investigation on this sphere."

**Thought process:**

No hints → Exploration mode. Orient first, then detect, then advanced scans,
then root cause (40% budget).

```
sphere_overview(detail="full")
→ 3 patterns: customers_geo, orders_geo (event), order_items_composite
→ profiling_alerts: [orders_geo/avg_price ratio=3.2,
                     customers_geo/total_orders ratio=1.8]
→ event_rate_divergence_alerts: [CUST-1847 rate=0.22]
→ has_temporal: customers_geo=true

get_sphere_info()
→ aliases: high_value_customers (cutting plane on total_orders)
check_alerts() → no calibration issues

[Execute ALL profiling_alert calls immediately]
find_anomalies("orders_geo", rank_by_property="avg_price", top_n=50)
→ 50 orders with extreme avg_price, extract unique customer keys
find_anomalies("customers_geo", rank_by_property="total_orders", top_n=50)
→ 50 customers with extreme order counts

[Main detection]
find_anomalies("customers_geo", top_n=20) → 20 geometric anomalies
aggregate("orders_geo", group_by_line="customers",
          geometry_filters={"is_anomaly": true}, limit=50)
aggregate("orders_geo", group_by_line="customers", limit=50)
→ compute rates: 4 customers above 15% threshold

[Composite]
aggregate_anomalies("order_items_composite", group_by="customer_key", limit=50)
find_anomalies("order_items_composite", rank_by_property="avg_item_price", limit=50)

[Alias boundaries]
attract_boundary("high_value_customers", "customers_geo", direction="in")
attract_boundary("high_value_customers", "customers_geo", direction="out")

[Advanced scans — ALL 4]
passive_scan("customers", threshold=1) → cross-pattern
... [10+ find_similar_entities calls] → neighborhood
... [dive_solid at 3 displacement ranges] → trajectory
find_regime_changes("customers_geo") → segment
→ changepoint detected at 1997-03
→ property_filters by nation → FRANCE shows 12 anomalies vs 2-4 for others

[Root cause — 40% budget on top findings]
explain_anomaly("CUST-1847", "customers_geo")
dive_solid("CUST-1847", "customers_geo") → spike in 1997 slices
find_counterparties("CUST-1847", "orders", "customer_key", "supplier_key")
→ 34 orders to 2 specific suppliers in 1997
compare_entities(["CUST-1847"], ["CUST-0010","CUST-0022"], "customers_geo")

[Temporal]
find_drifting_entities("customers_geo", top_n=5)
find_regime_changes("customers_geo", n_regimes=5)
compare_time_windows("customers_geo", "1996-01-01", "1997-01-01",
                     "1997-01-01", "1998-01-01")
```

**Findings summary:**
1. FRANCE population shift (1997-03): 12 French customers anomalous post-changepoint,
   driven by total_orders +2.1σ. 12 entity keys listed.
2. 4 customers with high event anomaly rates (16-22%), including CUST-1847 with
   burst localized to 1997 Q2 (34 orders, 2 suppliers).
3. ORDER-ITEMS composite: 3 parents with >=2 anomalous subgroups AND
   avg_item_price >3x normal on specific part categories.
4. No geometric neighborhood contamination found (scan 2 negative — explicit finding).
5. No non-linear trajectories found at moderate/low displacement ranges (scan 3 negative).
