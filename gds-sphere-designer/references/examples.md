# Real-World Sphere Config Examples

Examples drawn from two production benchmark spheres:
- **Berka** — Czech banking dataset (4,500 accounts, 1M transactions)
- **TPC-H** — Supply chain benchmark (6M lineitems, 10K suppliers)

---

## NB-Split: Isolated Entity Lines for Orthogonal Patterns — Example from Berka

**sphere.yaml excerpt:**
```yaml
lines:
  accounts:
    source: accounts
    key: primary_key
    role: anchor
    fts: true
    description: "Bank accounts enriched with owner demographics, loan status, card type, and standing orders."
  accounts_stress:
    source: accounts
    key: primary_key
    role: anchor
    description: "Account stress view — separate entity line for isolated stress pattern (avoids inheriting behavioral/loan dims from shared accounts line)."

patterns:
  account_behavior_pattern:
    type: anchor
    entity_line: accounts
    description: "Account behavioral profile — transaction activity, diversity, and burst patterns. Orthogonal to stress/loan patterns for NB-Split-style composite scoring via Fisher's method."
    derived_dimensions:
      - from_pattern: tx_pattern
        features:
          - tx_count: count
          - n_distinct_banks: count_distinct:bank
          - n_categories: count_distinct:k_symbol
          - n_operations: count_distinct:operation
          - burst_daily: "count:window=1d:agg=max"
          - burst_weekly: "count:window=7d:agg=max"
          - burst_monthly: "count:window=30d:agg=max"
      - from_pattern: orders
        features:
          - n_standing_orders_derived: count
    relations: auto
    anomaly_percentile: 95
    dimension_weights: kurtosis
    tracked_properties: [owner_gender, frequency]

  account_stress_pattern:
    type: anchor
    entity_line: accounts_stress
    description: "Financial stress signals with context — penalty interest, balance volatility, minimum balance, balance trend + frequency and district context for per-cohort calibration."
    precomputed_dimensions:
      - column: penalty_interest_count
        display_name: "Penalty Interest Count"
      - column: balance_volatility
        display_name: "Balance Volatility (CV)"
      - column: min_balance
        display_name: "Minimum Balance"
      - column: balance_trend
        display_name: "Balance Trend"
    derived_dimensions:
      - from_pattern: tx_pattern
        anchor_fk: account_id
        features:
          - amount_std: std:amount
          - mean_balance: avg:balance
    relations: []
    anomaly_percentile: 90
    group_by_property: frequency
    dimension_weights: kurtosis
    tracked_properties: [loan_status, has_loan]
```

**What this does:** Both patterns point to the same source data (`accounts`), but they use separate entity lines (`accounts` vs `accounts_stress`). The behavioral pattern captures activity signals; the stress pattern captures financial distress signals. Because they live on different lines, their geometry is completely independent — mu/sigma/theta for stress is calibrated against stress dimensions only, never contaminated by activity counts. You can then combine the two anomaly scores via Fisher's method at query time.

**When to use:** Any time an entity has two or more conceptually orthogonal risk dimensions that would distort each other if mixed into a single delta vector. Classic cases: activity vs. stress in banking, volume vs. pricing in supply chain, operational vs. financial in GL.

**Common mistake:** Putting behavioral and stress dimensions into a single pattern. This causes the theta calibration to be dominated by whichever dimension set has higher variance, and the other dimensions have reduced detection power. Separate lines fix this.

---

## Temporal Config on Anchor Pattern — Example from Berka

**sphere.yaml excerpt:**
```yaml
temporal:
  - pattern: account_behavior_pattern
    event_line: transactions
    timestamp_col: date
    window: 90d
  - pattern: account_stress_pattern
    event_line: transactions
    timestamp_col: date
    anchor_fk: account_id
    window: 90d
```

**What this does:** Both patterns get quarterly (90-day) temporal snapshots. The `account_stress_pattern` adds `anchor_fk: account_id` because it lives on the `accounts_stress` isolated line, not the default `accounts` line — the builder needs to know which FK on the transactions event line to use when reconstructing per-account history.

**When to use:** 90d windows work well for monthly-cycle financial data (salary, rent, recurring transfers). Use shorter windows (30d) for high-frequency data where behavioral shifts are fast; use longer windows (180d–365d) for slower-moving datasets like supply chain shipments or annual purchasing cycles.

**Common mistake:** Omitting `anchor_fk` when the pattern entity line is not the one directly referenced by the event pattern's primary relation. The builder will fail with "unknown pattern" or produce incorrect temporal slices.

---

## Alias Definition — Example from Berka

**sphere.yaml excerpt:**
```yaml
aliases:
  high_activity_accounts:
    base_pattern: account_behavior_pattern
    cutting_plane:
      dimension: _d_tx_count
      threshold: 1.5
    description: "Accounts with tx_count > 1.5σ above mean — high-activity segment"
  low_engagement_accounts:
    base_pattern: account_behavior_pattern
    cutting_plane:
      dimension: _d_burst_monthly
      threshold: -1.0
    description: "Accounts with burst_monthly < -1σ — low engagement segment"
```

**What this does:** Each alias defines a population segment via a cutting plane in delta-space. `_d_tx_count` is the delta coordinate for the `tx_count` dimension — prefix `_d_` always. A threshold of `1.5` means "entities whose tx_count delta is at least 1.5 standard deviations above the population mean". The low-engagement alias uses a negative threshold to capture entities below the mean. Each alias gets its own sub-population mu/sigma/theta at build time.

**When to use:** When you want to run `attract_boundary` or `contrast_populations` on a specific segment without manually filtering every query. Also used to detect anomalies within a sub-population rather than against the global baseline.

**Common mistake:** Using raw column names instead of delta-prefixed names (e.g. `tx_count` instead of `_d_tx_count`). The cutting plane operates in delta-space, not feature-space.

---

## Derived Dimensions with anchor_fk — Example from Berka

**sphere.yaml excerpt:**
```yaml
  account_stress_pattern:
    type: anchor
    entity_line: accounts_stress
    derived_dimensions:
      - from_pattern: tx_pattern
        anchor_fk: account_id
        features:
          - amount_std: std:amount
          - mean_balance: avg:balance
```

**What this does:** `anchor_fk: account_id` tells the builder which FK column on the event line (`transactions`) links back to this entity line when the entity line is not the primary relation target of `tx_pattern`. Without it, the builder would look for the default FK and fail to resolve the join, because `account_stress_pattern` lives on `accounts_stress`, a secondary line with no direct FK in the event pattern.

**When to use:** Required whenever you use the NB-Split pattern — a derived dimension that pulls from an event pattern on behalf of an isolated entity line. Also required on composite patterns where the event has multiple FKs to the same anchor type (e.g. `from_account` / `to_account`).

**Common mistake:** Defining derived dimensions on an isolated entity line without `anchor_fk`. The build succeeds but all derived dims are zero because the join resolves to 0 rows.

---

## Source Script Reference — Example from Berka

**sphere.yaml excerpt:**
```yaml
sources:
  accounts:
    script: prepare_accounts.py
  transactions:
    script: prepare_transactions.py
  orders:
    script: prepare_orders.py
  districts:
    script: prepare_districts.py
```

**What this does:** Each source delegates to a Python script that returns a `pa.Table`. The scripts handle date parsing (YYMMDD integers), computed columns (`penalty_interest_count`, `balance_trend`), multi-table joins, and NaN coercion. The builder caches script output as parquet in `.cache/` — the script runs once; subsequent builds load the cached file in milliseconds.

**When to use:** Use Tier 3 scripts when the source data requires any of: non-standard date formats, computed columns (e.g. balance trend requires windowed aggregation), multi-table joins with aggregations, or NaN sentinel handling. For clean parquet files, prefer Tier 1 (`path: data/file.parquet`) to keep the YAML simple.

**Common mistake:** Running heavy computation (igraph, windowed z-scores, Python loops over millions of rows) inside the prepare script without caching. The cache only kicks in on the second build — if you rebuild with `--force` on every iteration, you pay the script cost every time. Pre-export expensive computations to parquet and reference them as Tier 1 sources.

---

## Event Pattern with Multiple Dimensions — Example from TPC-H

**sphere.yaml excerpt:**
```yaml
  lineitem_pattern:
    type: event
    entity_line: lineitems
    description: "Lineitem geometry — structural shape of each shipment."
    relations:
      - line: customers
        direction: in
        key_on_entity: custkey
        required: true
      - line: suppliers
        direction: in
        key_on_entity: suppkey
        required: true
      - line: shipmodes
        direction: in
        key_on_entity: shipmode
        required: false
    event_dimensions:
      - column: extendedprice
      - column: discount
      - column: quantity
      - column: late_days
    anomaly_percentile: 95
    dimension_weights: kurtosis
```

**What this does:** The event pattern combines 3 relational dimensions (customer, supplier, shipmode) with 4 continuous event dimensions (price, discount, quantity, delivery lateness). Each relation adds a directional edge to the event polygon. The `late_days` column is a computed column from the prepare script (`receipt_date - commit_date`) — it is not in the raw TPC-H schema and must be created in `prepare_lineitems.py`. `required: false` on `shipmodes` means events without a shipmode value are still included in the geometry.

**When to use:** Use multiple `event_dimensions` when each dimension captures a distinct aspect of the event's shape. Four is a reasonable maximum — beyond 7-8 dimensions, delta vector distances become less discriminative (curse of dimensionality). `kurtosis` weighting automatically down-weights low-variance dimensions, so adding a low-signal column rarely hurts as long as signal columns are still present.

**Common mistake:** Including highly correlated columns as separate event dimensions (e.g. both `extendedprice` and `net_price = extendedprice * (1 - discount)`). Correlated dims double-count the same signal and inflate delta distances for that direction. Include either the raw price or the derived price, not both.

---

## Composite Lines — Example from TPC-H

**sphere.yaml excerpt:**
```yaml
composite_lines:
  supplier_part_pairs:
    event_line: lineitems
    key_cols: [suppkey, partkey]
    description: "Supplier-part pair geometry — count, revenue, price variance per (supplier, part) pair."
    derived_dimensions:
      - features:
          - pair_count: count
          - pair_revenue: sum:extendedprice
          - pair_price_std: std:extendedprice
          - pair_avg_price: avg:extendedprice
          - pair_max_price: max:extendedprice
    anomaly_percentile: 99

  supplier_parttype_pairs:
    event_line: lineitems
    key_cols: [suppkey, parttype]
    description: "Supplier-part-type pair geometry — per-type pricing profile. Detects subgroup-level pricing anomalies invisible at aggregate supplier level."
    derived_dimensions:
      - features:
          - spt_count: count
          - spt_avg_price: avg:extendedprice
          - spt_max_price: max:extendedprice
          - spt_price_std: std:extendedprice
    anomaly_percentile: 95
```

**What this does:** Composite lines create a new entity type from every unique combination of the `key_cols`. `supplier_part_pairs` creates ~800K entities (one per supplier-part combination seen in lineitems). Each composite entity gets a delta vector aggregated from all events matching that key pair. This surfaces anomalies that are invisible at the individual supplier or individual part level — a supplier may look normal overall but be an outlier when supplying a specific part.

`supplier_parttype_pairs` uses a coarser grouping (`parttype` instead of `partkey`) to catch broader pricing anomalies across part categories. Higher `anomaly_percentile` (99) on `supplier_part_pairs` because the population is large and many pairs have sparse data — a conservative threshold reduces noise from low-count pairs.

**When to use:** Add composite lines when you suspect that the relationship between two entities matters, not just each entity in isolation. Classic cases: (supplier, part), (account, counterparty), (customer, product_category), (merchant, mcc_code).

**Common mistake:** Using a high-cardinality key for one of `key_cols` without checking how many unique pairs result. If `key_cols` produces 10M+ pairs from 6M events, most pairs have 1 event each — the geometry is meaningless and the build will be slow. Check expected pair count before adding a composite line.

---

## Multiple Anchor Patterns on Isolated Lines — Example from TPC-H

**sphere.yaml excerpt:**
```yaml
lines:
  suppliers:
    source: suppliers
    key: primary_key
    role: anchor
    fts: true
    description: "TPC-H suppliers with nation and account balance."
  suppliers_pricing:
    source: suppliers
    key: primary_key
    role: anchor
    description: "Isolated supplier line for pricing pattern — avoids inheriting activity dims."

patterns:
  supplier_activity_pattern:
    type: anchor
    entity_line: suppliers
    description: "Supplier activity profile — volume, revenue, customer and part diversity. General supplier health indicator."
    derived_dimensions:
      - from_pattern: lineitem_pattern
        features:
          - lineitem_count: count
          - total_revenue: sum:extendedprice
          - n_customers: count_distinct:custkey
          - n_parts_supplied: count_distinct:partkey
          - n_ship_modes: count_distinct:shipmode
    relations: auto
    anomaly_percentile: 95
    dimension_weights: kurtosis
    gmm_n_components: 2
    tracked_properties: [nation]

  supplier_pricing_pattern:
    type: anchor
    entity_line: suppliers_pricing
    description: "Supplier pricing profile — price variance, discount, peak price. Isolated line."
    derived_dimensions:
      - from_pattern: lineitem_pattern
        anchor_fk: suppkey
        features:
          - price_std: std:extendedprice
          - avg_discount_given: avg:discount
          - max_single_price: max:extendedprice
    relations: auto
    anomaly_percentile: 95
    dimension_weights: kurtosis
    gmm_n_components: 2
    tracked_properties: [nation]
```

**What this does:** The same supplier entities get two completely independent behavioral profiles. `supplier_activity_pattern` captures volume and diversity (how much work does this supplier do?). `supplier_pricing_pattern` captures pricing behavior (what prices does this supplier charge?). Both have `gmm_n_components: 2` because TPC-H suppliers split into large and small tiers — a single Gaussian would poorly calibrate the threshold. `tracked_properties: [nation]` on both allows the agent to detect which nations have data quality gaps.

**When to use:** When an entity type has two business questions that require different dimensions and different calibration baselines. Here: "is this supplier unusually active?" and "is this supplier charging unusual prices?" are independent questions — merging them into one pattern would let high-activity suppliers mask pricing anomalies.

**Common mistake:** Giving the isolated line `fts: true` when it is only used as an internal pattern target. Full-text search is only meaningful on lines the agent will search by name or property. Setting `fts: true` on an internal line wastes build time without benefit.

---

## group_by_property on Customer Pattern — Example from TPC-H

**sphere.yaml excerpt:**
```yaml
  customer_activity_pattern:
    type: anchor
    entity_line: customers
    description: "Customer purchasing activity — order volume, spend, supplier diversity, burst."
    derived_dimensions:
      - from_pattern: lineitem_pattern
        features:
          - order_count: count
          - total_spend: sum:extendedprice
          - n_suppliers: count_distinct:suppkey
          - n_parts: count_distinct:partkey
          - burst_monthly: "count:window=30d:agg=max"
    relations: auto
    anomaly_percentile: 95
    dimension_weights: kurtosis
    gmm_n_components: 3
    group_by_property: mktsegment
    tracked_properties: [nation]
```

**What this does:** `group_by_property: mktsegment` splits the customer population into five sub-groups (AUTOMOBILE, BUILDING, FURNITURE, HOUSEHOLD, MACHINERY) and calibrates mu/sigma/theta independently per group. An AUTOMOBILE customer is anomalous relative to other AUTOMOBILE customers, not relative to all 100K customers. This is essential when sub-populations have structurally different normal behavior — a MACHINERY customer naturally spends more on large parts than a FURNITURE customer.

`gmm_n_components: 3` models each group as a mixture of three Gaussians, capturing small/medium/large customer tiers within each segment. `tracked_properties: [nation]` tracks which nations have missing or incomplete customer data — useful for data quality monitoring.

**When to use:** Set `group_by_property` when you have a categorical column that strongly predicts spending patterns, transaction volume, or other behavioral dimensions. Rule of thumb: if removing the group label reduces the R² of a regression by >10%, it belongs in `group_by_property`. Maximum cardinality: `group_count × population_size < 10M`. At 5 segments × 100K customers = 500K — well within limits.

**Common mistake:** Using a high-cardinality property like `nation` (25 values × 100K customers = 2.5M — acceptable) for `group_by_property` on a small population. If any group has fewer than ~200 entities, the mu/sigma estimate for that group is unstable and theta will be miscalibrated. Check minimum group size before setting `group_by_property`.

---

## tracked_properties — Example from TPC-H

**sphere.yaml excerpt:**
```yaml
  supplier_activity_pattern:
    tracked_properties: [nation]

  supplier_pricing_pattern:
    tracked_properties: [nation]

  customer_activity_pattern:
    tracked_properties: [nation]
```

**What this does:** `tracked_properties` instructs the builder to record which entities have null or missing values for the listed properties. The agent can then call `detect_data_quality_issues` to find entities missing `nation`, or use `get_line_profile` to see the nation distribution. In TPC-H, `nation` is always present — but tracking it makes data quality monitoring explicit and repeatable when the sphere is rebuilt against updated source data.

**When to use:** Track any property that should be non-null for every entity (nationality, segment, status, category). Do not track high-cardinality properties like free-text fields — the tracking overhead is proportional to cardinality.

**Common mistake:** Tracking `tracked_properties` on event lines instead of anchor lines. The feature is designed for entity-level data quality — tracking null rates on event columns is better handled via source-level validation in the prepare script.
