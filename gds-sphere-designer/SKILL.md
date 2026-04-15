---
name: gds-sphere-designer
description: End-to-end sphere design and construction — from data discovery through pattern strategy, YAML generation, build, calibration, and iterative tuning. Use when user asks to "design a sphere", "build a sphere", "create a sphere from my data", "optimize my sphere", "set up hypertopos", "generate sphere.yaml", "import my data into GDS", "what patterns should I build", "why is my recall low", "too many false positives", "how to split dimensions", "which dimensions should I use", "my anomaly rate is wrong", "review my sphere design", or "what's wrong with my detection". Covers both conceptual design (WHAT) and materialization (HOW).
license: Apache-2.0
compatibility: Requires hypertopos CLI (pip install hypertopos). No live MCP session needed for design phases.
metadata:
  author: Karol Kedzia
  version: 0.4.1
  mcp-server: hypertopos
---

# GDS Sphere Designer Skill

End-to-end sphere design and construction. From raw data to navigable geometry.

## Core principle

**Fewer focused dimensions on an isolated entity line outperform mega-patterns.**
A focused pattern (3-8 dims) on an isolated line consistently beats a
mega-pattern (10+ dims) on a shared line -- both in recall and precision.
Cross-line composite scoring can close the remaining gap.

**The agent writes the YAML, not the user.** Ask questions about the data, generate
`sphere.yaml`, build, verify. The user's job is to answer questions and approve.

---

## Phase 1 -- Discover

Understand the data before designing anything.

Ask the user:

1. **What data files do you have?** Format (parquet, CSV), location, approximate size.
2. **What are the main entity tables?** (customers, accounts, products, suppliers)
3. **What are the event/transaction tables?** (orders, transactions, GL entries, shipments)
4. **How do events connect to entities?** Which columns are foreign keys?
5. **What numeric columns matter?** (amount, price, quantity, balance)
6. **Is there temporal data?** (dates, timestamps -- needed for drift detection)

If user provides DDL, schema description, or sample data -- extract answers automatically.

### Schema extraction shortcuts

If user can run queries:
```sql
SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'orders';
SELECT COUNT(*) FROM orders;
SELECT * FROM orders LIMIT 5;
```

If user has parquet files:
```python
import pyarrow.parquet as pq
print(pq.read_schema("data/orders.parquet"))
```

### Decision tree

For each table:

```
Table has unique business keys + entity properties?
  -> YES -> anchor line (customers, products, accounts)
  -> NO  -> Does it have FK references to anchors + timestamps?
    -> YES -> event line (transactions, orders, shipments)
    -> NO  -> Reference/lookup table -> anchor line (small)
```

### Mapping rules

| Data concept | GDS concept |
|-------------|-------------|
| Entity table (customers, products) | anchor line |
| Transaction/fact table | event line |
| FK column on event -> entity | relation (direction: in) |
| NOT NULL FK | required: true |
| NULLABLE FK | required: false |
| Numeric column on event (amount, price) | event_dimension |
| Aggregate per entity (count, sum, avg) | derived_dimension |
| Pair of FKs on same event | candidate composite_line |
| Date/timestamp column | temporal config |

---

## Phase 2 -- Design

### Pattern strategy: one concern per pattern

Each pattern should capture ONE logical concern:

| Concern | Example dims | What it detects |
|---------|-------------|-----------------|
| Behavioral activity | tx_count, burst_daily, n_categories | How active is the entity? |
| Financial stress | penalty_interest, balance_volatility, min_balance | Is the entity in distress? |
| Exposure/risk | balance_to_loan, income_coverage | Is the entity overextended? |
| Relationships | pair patterns, composite lines | Who does the entity transact with? |

Mixing concerns in one pattern dilutes signal -- behavioral dims dominate
because they have higher variance, pushing stress/risk dims below theta.

### Dimension budget: 3-6 dims per pattern

Ask for each dim: "does this help detect THIS concern?" If no, it belongs
in a different pattern. 1 dim = too little (delta_norm rarely exceeds theta).
10+ dims = too much (signal diluted by noise). Sweet spot: 3-6 per isolated concern.

### Entity line isolation (NB-Split)

**The most impactful design decision.** Patterns on a shared entity line
inherit ALL precomputed dimensions from that line's table.

```
SHARED LINE (accounts):
  behavior_pattern -> gets 8 behavioral + 6 stress + 4 loan = 18 dims
  stress_pattern   -> gets 6 stress + 8 behavioral + 4 loan = 18 dims
  Both diluted. Neither focused.

ISOLATED LINES:
  behavior_pattern on accounts       -> gets 8 behavioral dims only
  stress_pattern on accounts_stress  -> gets 6 stress dims only
  Each focused. Signal preserved.
```

Create an isolated line: same source, different line ID. Zero code change,
only YAML:

```yaml
lines:
  accounts:
    source: accounts
    key: primary_key
  accounts_stress:
    source: accounts      # same data
    key: primary_key      # same keys
```

**Rule:** If a pattern needs exactly N dims, give it its own entity line.

**NB-Split per direction:** When the same entity line appears in multiple
relation roles (e.g. zones as both pickup and dropoff), create NB-Split
lines per direction (`zones_pickup`, `zones_dropoff`). Each direction has
different aggregate behavior — merging them dilutes both signals. Example:

```yaml
lines:
  zones_pickup:
    source: zones
    key: zone_id
  zones_dropoff:
    source: zones      # same data
    key: zone_id       # same keys
```

This applies whenever an event has two FKs pointing to the same entity type
(from/to accounts, origin/destination warehouses, buyer/seller parties).

Full NB-Split design patterns, expected impact table, and cross-line bridging
details are in [references/design-patterns.md](references/design-patterns.md).

### Derived dimension heuristics

For each anchor entity, derive from its event data:

| Feature | Metric | What it captures |
|---------|--------|------------------|
| Activity volume | `count` | How active is this entity? |
| Total monetary | `sum:<amount_col>` | Total value |
| Average transaction | `avg:<amount_col>` | Typical transaction size |
| Volatility | `std:<amount_col>` | How variable are transactions? |
| Peak value | `max:<amount_col>` | Largest single transaction |
| Counterparty diversity | `count_distinct:<fk_col>` | How many distinct partners? |
| Daily burst | `count:window=1d:agg=max` | Peak daily activity |
| Monthly burst | `count:window=30d:agg=max` | Peak monthly activity |
| Inter-event-time mean | `iet_mean` | Average gap between events |
| Inter-event-time regularity | `iet_std` | How regular are event intervals? |

**Rule of thumb:** Start with 8-12 derived dimensions. Too few = weak signal. Too many = noise.
Good starting set: count + sum + avg + std + max + 2-3 count_distinct + burst_monthly.

**Multi-FK patterns:** When an event has two FKs to the same anchor line (e.g. `from_account` / `to_account`), use `anchor_fk` to disambiguate:
```yaml
derived_dimensions:
  - from_pattern: tx_pattern
    features:
      - tx_out_count: count
  - from_pattern: transactions
    anchor_fk: to_account
    features:
      - tx_in_count: count
```

### Composite pattern dimension selection

For composite patterns (entity-pair), always include at least one
per-transaction metric (avg, max) alongside aggregate metrics (count, sum, std).
Aggregate-only dims miss targeted anomalies. See [references/design-patterns.md](references/design-patterns.md).

### Ground truth alignment

When labeled outcomes exist (default, churn, fraud), validate each pattern
independently:

1. Which pattern best separates labeled entities? (`contrast_populations` by MAX |d|)
2. Bottom-up recall: check each labeled entity for `is_anomaly` per pattern
3. The best detector may not be the pattern you expected

Match pattern concern to detection goal. A behavioral pattern will have low
recall on financial distress. A stress pattern will catch distress but miss
behavioral fraud.

### High-variance dims drown drift

If one dimension has 40x the variance of others, temporal drift ranking
will be dominated by that dimension. Check `anomaly_dimensions` on a few
top anomalies. If one dim contributes >80% of delta_norm across most
anomalies, remove or isolate it. For temporal, isolate relational dims
(count_distinct, diversity) into their own pattern.

### Dimension kind tags

The builder auto-assigns a distribution kind to each derived dimension. This
drives per-dimension Bregman divergence scoring and per-dimension theta
(hyper-ellipsoid boundary instead of hyper-sphere). In most cases the
auto-detect is correct; override only when the auto-detect is wrong:

| Auto-detect rule | Assigned kind |
|-----------------|--------------|
| Binary FK (`edge_max=1`) | `bernoulli` |
| `count`, `count_distinct`, window burst | `poisson` |
| `sum`, `avg`, `std`, `max` | `gaussian` |
| Entity property (fill indicator) | `bernoulli` |

**When to override:**
- A `count` dim that is always 0 or 1 (binary presence flag) → force `bernoulli`
- A `sum` dim that represents event occurrence (never fractional) → consider `poisson`
- A precomputed dim from an external system whose distribution you know → set explicitly

Override in YAML:
```yaml
derived_dimensions:
  - from_pattern: event_pattern
    features:
      - cross_border_flag: count_distinct:country  # auto: poisson
        kind: bernoulli                             # override: binary presence
```

**Impact on navigation:**
- `bernoulli` dims: `metric="Linf"` in `find_anomalies` catches single-flag
  deviations that L2 norm dilutes. Recommend Linf for patterns with 4+ bernoulli dims.
- `poisson` dims: anomaly = count structure deviation (structuring, burst). Bregman
  divergence is more sensitive than Euclidean for count data at low counts.
- `gaussian` dims: standard behavior. Bregman ≈ squared distance for well-calibrated
  gaussians — no practical difference from prior behavior for these dims.

For mixed-kind patterns, use `find_anomalies(metric="bregman")` to rank by
distribution-aware scoring instead of L2. Check `dimension_kinds` in
`sphere_overview` after build to verify assignments.

### Bootstrap confidence tuning

Each pattern emits `anomaly_confidence` (0–1) per entity for populations <= 50K.
This reflects bootstrap stability: how consistently the entity is flagged across
resampled populations. The number of resamples is controlled by `bootstrap_iterations`
in YAML (default 200):

```yaml
patterns:
  entity_pattern:
    type: anchor
    entity_line: entities
    bootstrap_iterations: 200   # default — good balance of stability vs build time
```

**When to change:**
- `bootstrap_iterations: 0` — disable bootstrap entirely. Use during development
  iteration (fast builds) or when the pattern uses `group_by_property` / `use_mahalanobis`
  (confidence is not emitted for those anyway).
- `bootstrap_iterations: 50` — faster builds, rougher confidence estimates. Acceptable
  for exploratory spheres where you only need coarse stable/unstable classification.
- `bootstrap_iterations: 500` — smoother confidence, recommended for production spheres
  where `min_confidence` is used as an alert threshold. Build time scales linearly.

**Note:** `anomaly_confidence` is not computed for `group_by_property` patterns (per-cohort
calibration) or `use_mahalanobis` patterns. It is also skipped when population > 50K — use
`conformal_p` for large populations instead.

### Advanced features

AML-specific features (precomputed dims, graph features, chain lines, Mahalanobis)
are documented in [references/advanced-features.md](references/advanced-features.md).

### Generalized dimension blocks (g/t/s)

Three optional dimension block types enrich the shape vector beyond relational counts:

| Block | YAML key | Normalization | Use case |
|-------|----------|---------------|----------|
| **g** (geographic) | `geo_properties: [lat, lon]` | mu/sigma | Spatial patterns |
| **t** (metric) | `metric_properties: [balance, credit_limit]` | mu/sigma | Numeric entity attributes |
| **s** (semantic) | `semantic_dim: {columns: [emb], n_components: 8}` | PCA + mu/sigma | Embedding reduction |

All optional. Dimension names are prefixed (`g:lat`, `t:balance`, `s:pc0`) in
`explain_anomaly` output. Columns must exist on the entity line's source data.

```yaml
patterns:
  account_pattern:
    type: anchor
    entity_line: accounts
    relations: auto
    metric_properties:
      - balance
      - credit_limit
    geo_properties:
      - branch_lat
      - branch_lon
```

When to use: entity has numeric attributes (balance, age, rating) or spatial
coordinates that should influence anomaly detection. Without these blocks, the
shape vector only captures relational structure (who is connected to whom).

### Edge table design

The edge table stores per-event from/to relationships, enabling runtime graph
traversal (`find_geometric_path`, `discover_chains`, `edge_stats`) without
pre-built chain lines.

**When edge table makes sense:**

- Event pattern with from/to FK structure to the same anchor line
  (account-to-account, entity-to-entity)
- Sparse graph (avg degree < 100) -- dense graphs produce too many paths
  and BFS becomes expensive
- Investigation use case: "how are these two entities connected?"
- AML/fraud domains: chain discovery, path tracing, counterparty analysis

**When edge table does NOT make sense:**

- Anchor patterns -- no from/to structure, nothing to emit
- Event patterns where relations point to different anchor lines without
  a shared entity type (e.g. order connects customer and product -- no
  entity-to-entity graph)
- Dense graphs with very high average degree per node (few unique nodes,
  many edges each) -- produces unusable path explosion
- When you only need aggregate features -- use `graph_features` instead,
  which computes in_degree/out_degree/reciprocity without storing edges

**Auto-detect vs explicit config:**

The builder auto-detects edge table candidates when an event pattern has
2+ FK relations to the same anchor line, or when `graph_features` is
configured for the same event line. This covers the common case
(transactions with from_account/to_account).

Use explicit `edge_table` config when:
- Column names don't match auto-detect heuristics
- You want specific `timestamp_col` or `amount_col` for temporal BFS
  and amount-weighted scoring

```yaml
patterns:
  tx_pattern:
    edge_table:
      from_col: sender_id
      to_col: receiver_id
      timestamp_col: tx_date    # enables temporal BFS in discover_chains
      amount_col: amount        # enables amount-weighted path scoring
```

Skip edge table emission entirely with `--no-edges` CLI flag during fast
dev builds (`hypertopos build --no-edges --no-chains --no-temporal`).

**Coexistence with chain_lines:**

Edge table and `chain_lines` serve different purposes and can coexist:
- **Edge table** -- runtime per-entity investigation (BFS from a specific
  entity, interactive path finding)
- **chain_lines** -- pre-built population-wide chain geometry (pattern
  statistics, anomaly detection on chain properties)

Use both when you need population-level chain anomaly detection AND
per-entity interactive investigation.

**Timestamp and amount columns:**

- `timestamp_col` must be Arrow timestamp or parseable string. Without it,
  temporal BFS (`discover_chains` with `time_window_hours`) will not work,
  but graph traversal (`find_geometric_path`) still works.
- `amount_col` enables amount-weighted path scoring and flow analysis.
  Without it, paths are scored by geometric coherence only.

**Anti-patterns:**

| Anti-pattern | Why it fails | Fix |
|-------------|-------------|-----|
| Edge table on anchor pattern | No from/to structure -- nothing to emit | Only configure on event patterns with 2+ FKs to same anchor |
| `find_geometric_path` on dense graph (avg_degree > 100) | Path explosion, slow BFS, too many results | Check `edge_stats` first; use `graph_features` for aggregate metrics instead |
| Relying on auto-detect for complex schemas | Auto-detect picks first two FKs to same line -- may be wrong | Use explicit `edge_table` config with correct column names |
| Skipping `edge_stats` before path queries | No awareness of graph density or data quality | Always run `edge_stats` after build to verify avg_degree and row counts |

### Common patterns by domain

Full domain-specific patterns (financial, supply chain, SaaS, GL) are in
[references/domain-patterns.md](references/domain-patterns.md).

---

## Phase 3 -- Configure

### Choose source tier

| Tier | When | Example |
|------|------|---------|
| **1: Single file** | Clean parquet/CSV | `path: data/customers.parquet` |
| **1c: File + transforms** | CSV with wrong types | `path: data/tx.csv` + `transform:` block |
| **2: Multi-file join** | Entity data across files | `path:` + `join:` with file/on/columns |
| **3: Python script** | Complex transforms, NaN, dates | `script: prepare_accounts.py` |

Use Tier 3 for: non-standard date formats, computed columns, complex joins,
NaN sentinel values. Everything else: Tier 1 or 2. Script must export
`def prepare() -> pa.Table`.

### Generate sphere.yaml

Write the YAML following this template. Include `description` fields -- they help
agents understand the sphere when navigating.

```yaml
version: "0.1.0"

sphere_id: my_sphere
name: "Human-readable name"
description: "What this sphere represents and what questions it can answer."

sources:
  # ... (from source tier decisions)

lines:
  events:
    source: events_source
    key: event_id
    role: event
    description: "What each row represents."
  entities:
    source: entities_source
    key: entity_id
    role: anchor
    fts: true
    description: "What each entity is."

patterns:
  event_pattern:
    type: event
    entity_line: events
    description: "What this geometry captures."
    relations:
      - line: entities
        direction: in
        key_on_entity: entity_id
        required: true
        # edge_max: 10  # optional: continuous mode (default: null = binary 0/1)
    event_dimensions:
      - column: amount
    anomaly_percentile: 95
    dimension_weights: kurtosis

  entity_pattern:
    type: anchor
    entity_line: entities
    description: "Behavioral profile of each entity."
    derived_dimensions:
      - from_pattern: event_pattern
        features:
          - event_count: count
          - total_amount: sum:amount
          - avg_amount: avg:amount
          - amount_std: std:amount
          - max_amount: max:amount
          - burst_monthly: "count:window=30d:agg=max"
    relations: auto
    anomaly_percentile: 95
    dimension_weights: kurtosis
    gmm_n_components: 3

temporal:
  - pattern: entity_pattern
    event_line: events
    timestamp_col: date
    window: 90d
```

### Aliases (population segments)

Aliases partition entities by a hyperplane in delta-space:

```yaml
aliases:
  high_spenders:
    base_pattern: customer_pattern
    cutting_plane:
      dimension: _d_total_spend      # delta-prefixed name
      threshold: 2.0                  # entities with delta[dim] >= threshold
    description: "Customers with high total spend signal"
```

Two specification modes:
- **Dimension sugar**: `dimension` + `threshold` -- auto-generates unit vector
- **Explicit**: `normal` + `bias` -- full hyperplane specification

---

## Phase 4 -- Build + Verify

### Build

```bash
hypertopos build --config sphere.yaml --output my_sphere/ --force --verbose
```

### Iterative build (fast feedback loop)

```bash
# Step 1: Geometry only -- fast iteration on dimensions + calibration
hypertopos build --config sphere.yaml --force --verbose --no-chains --no-temporal

# Step 2: When geometry looks good, add temporal
hypertopos build --config sphere.yaml --force --verbose --no-chains

# Step 3: Full build with chains (uses cache on 2nd+ run)
hypertopos build --config sphere.yaml --force --verbose
```

### Performance notes

- **Source caching:** Tier 3 script results cached as parquet in `.cache/`. Second build skips script execution.
- **Chain caching:** Extracted chains cached as pickle in `.cache/`. Second build loads in <1s.
- **Parallel:** Sources load in parallel; geometry + temporal build per-pattern in parallel.
- **Batched derived dims:** All simple derived dims sharing the same FK computed in a single `group_by` call.
- **IVF_FLAT indices:** Both geometry and trajectory use IVF_FLAT -- builds in seconds.

### Verify

```
open_sphere("my_sphere")
sphere_overview()                    # Check: anomaly_rate ~5%, calibration_health = good, profiling_alerts, has_temporal
get_sphere_info()                    # Check: all lines present, total_rows correct
get_line_profile("entities", "key_property")  # Check: value distribution makes sense
find_anomalies("entity_pattern", top_n=3)     # Check: anomalies have meaningful dimensions
```

### Profiling alerts as design signal

After build, `sphere_overview` emits `profiling_alerts` when a dimension
has max/p99 ratio > 1.5:

- **Extreme ratio (>3.0):** A handful of entities dominate the dimension scale. Check with `find_anomalies(rank_by_property=dim)`. If genuine, keep. If noise, cap or remove.
- **Same alert on multiple patterns:** Both patterns declare the same dim -- design smell. Each dim should belong to one concern.
- **Moderate alerts (1.5-3.0) on financial data:** Expected -- heavy-tail distributions. Focus on extreme alerts first.

**Rule:** after every rebuild, check `profiling_alerts` before evaluating recall.

---

## Phase 5 -- Iterate

After initial build, the user may want to:

1. **Add more dimensions** -- edit `derived_dimensions`, rebuild with `--no-chains --no-temporal`
2. **Add composite lines** -- add `composite_lines` section for entity pair analysis
3. **Tune thresholds** -- adjust `anomaly_percentile`, `gmm_n_components`, rebuild geometry only
4. **Add temporal** -- add `temporal` section, rebuild with `--no-chains`
5. **Track properties** -- add `tracked_properties` for data quality monitoring
6. **Add aliases** -- segment the population with cutting planes

Each change = edit YAML -> `hypertopos build --force` -> verify. Use skip flags for fast iteration.

### Per-cohort calibration

`group_by_property` gives independent mu/sigma/theta per subgroup. Use when
subgroups have different normal behavior (e.g. monthly vs weekly accounts).
Effect is marginal when the population is homogeneous.

### Composite scoring

`composite_risk` (Fisher's method) combines p-values from independent patterns.
`passive_scan` screens the full population.

- **When it helps:** patterns capture genuinely independent signals. A default may be normal in behavior but anomalous in stress.
- **Cross-line bridging:** `composite_risk` and `passive_scan` auto-bridge across sibling lines (same `source` in sphere.yaml).
- **When it doesn't help:** patterns share dims or entity line. Adding composite over correlated patterns inflates false positives.

**Rule:** test single-pattern recall first. If <95% and you have independent
patterns, add composite_risk. If already >95%, composite adds marginal value.

### Shared dims across injection types

If two anomaly types inflate the same dimension, the combined population shift
raises mu/sigma and neither stands out. Detection requires multi-pattern
triangulation -- confirm via a different pattern where only one source contributes.

---

## Calibration recommendations

| Parameter | Default | When to change |
|-----------|---------|----------------|
| `anomaly_percentile` | 95 | Lower (90) for more sensitive detection, higher (99) for fewer flags |
| `dimension_weights` | `kurtosis` | Use `kurtosis` unless all dimensions are equally important |
| `gmm_n_components` | 3 | Increase if population has distinct sub-groups (e.g. retail vs corporate) |
| `group_by_property` | null | Set when sub-populations have different normal behavior (e.g. per-country) |
| `tracked_properties` | null | Set for data quality tracking (which entities have null properties) |
| `bootstrap_iterations` | 200 | Set to 0 to skip (fast iteration); raise to 500 for production stable-anomaly confidence |

---

## Troubleshooting

### Build issues

| Signal | Fix |
|--------|-----|
| "column not found" | Check schema with `pyarrow.parquet.read_schema()` |
| "unknown pattern" in temporal | Use `{composite_line_id}_pattern` or `{chain_line_id}_pattern` |
| Sources phase >60s | Pre-export to parquet (Tier 1) or optimize script |
| Chains phase >5min | Reduce `max_chains` or increase `min_hops`. Cached after first run |
| Temporal phase >5min | Temporal rescans the event table per window — scales as O(entities x windows). Use `--no-temporal` for geometry iteration, then add temporal as a final pass. Use larger `window` (fewer snapshots) to reduce window count |
| Build hangs during Geometry | Rule: group_count x population < 10M |
| "Too many bins" / anomaly_rate=50% | Entity line has <10 entities -- too few for population statistics. Remove pattern or merge with larger line |
| `pc.strftime` timezone error (Windows) | Cast timestamp to tz-naive first: `col.cast(pa.timestamp("us"))` before date operations |
| `degree_velocity` returns flat/uniform timestamps | Edge auto-detect picked `created_at` (Lance metadata) instead of business timestamp. Set explicit `timestamp_col` in `edge_table` config |
| `discover_chains` time window has no effect | Same root cause — edge table has metadata timestamp. Set explicit `timestamp_col` |
| Amount-weighted path scoring inactive | Auto-detect only matches `amount`, `value`, `total`, `amt`. For domain-specific names (`fare_amount`, `total_amount`), set explicit `amount_col` in `edge_table` config |
| `contagion_score > 1.0` or anomalous > total counterparties | NB-Split anchor resolution bug — rebuild with hypertopos >= 0.3.0 |

### Calibration issues

| Signal | Fix |
|--------|-----|
| anomaly_rate = 0% | Lower `anomaly_percentile` to 90 |
| anomaly_rate > 20% | Raise `anomaly_percentile` to 99 |
| calibration_health = poor | Add `group_by_property` or adjust `gmm_n_components` |
| anomaly_rate higher with `group_by_property` | Expected -- per-group calibration is more sensitive |
| `sphere_overview` rate != `find_clusters` rate | Rebuild sphere with current version |

### Design issues

| Signal | Fix |
|--------|-----|
| All anomalies driven by one dim | Check for outliers; cap or remove |
| `profiling_alerts` extreme ratio (>3.0) | `find_anomalies(rank_by_property=dim)` to investigate |
| Pattern has >10 dims | Split into 2-3 focused patterns |
| Pattern has 1-2 dims | Merge with related concern or add dims |
| Entity line has <10 entities | Too few for meaningful mu/sigma/theta. Remove pattern or merge into larger line |
| Multiple patterns share entity line | Isolate with separate line |
| Recall on ground truth <30% | `contrast_populations` -- which dims actually separate? |
| One dim >80% of delta_norm | Remove or isolate the dominant dim |
| Dead dimensions (zero variance) | Remove from pattern |
| Temporal drift dominated by one dim | Isolate relational dims into own pattern |
| 0 temporal slices | Verify `timestamp_col` points to actual date column |

---

## Unsupervised ceiling

Unsupervised anomaly detection has a hard ceiling -- entities that are
geometrically normal but labeled "bad" for external reasons cannot be caught
by any individual pattern. Cross-line Fisher composite can recover a subset
(borderline in 2+ patterns), but genuine quiet outliers remain undetectable.
This is a data completeness problem, not a sphere design problem.

**Ceiling = per-pattern ceiling, not composite ceiling.** Design independent
patterns covering orthogonal concerns, then use `composite_risk` to combine.

---

## Examples

For real-world sphere config examples (Berka banking, TPC-H supply chain),
see [references/examples.md](references/examples.md).

### Quick example: Build from parquet

"I have customers.parquet and orders.parquet" -- Ask for PKs and FK columns,
generate sphere.yaml (anchor + event + derived dims), build, verify anomaly_rate ~5%.

### Quick example: Tune anomaly rate

"Anomaly rate is 25%" -- Raise `anomaly_percentile` from 95 to 99, fast rebuild
with `--no-chains --no-temporal`, verify rate drops to ~1-5%.

### Quick example: Add temporal

"Track behavior changes over time" -- Add `temporal:` section with pattern,
event_line, timestamp_col, window. Rebuild with `--no-chains`, verify slice count.
