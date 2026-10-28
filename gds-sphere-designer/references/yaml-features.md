# Sphere YAML Feature Blocks

Optional `sphere.yaml` blocks and build-time behaviors, read from the Phase 2 design step of the sphere designer skill.

### Dimension kind tags

The builder auto-assigns a distribution kind to each derived dimension. This
drives per-dimension Bregman divergence scoring and per-dimension theta
(hyper-ellipsoid boundary instead of hyper-sphere). The kind follows from how
the dimension is declared; only `precomputed_dimensions` entries accept a `kind:` override:

| Auto-detect rule | Assigned kind |
|-----------------|--------------|
| Binary FK (`edge_max=1`) | `bernoulli` |
| `count`, `count_distinct`, window burst | `poisson` |
| `sum`, `avg`, `std`, `max` | `gaussian` |
| Entity property (fill indicator) | `bernoulli` |
| `precomputed_dimensions` entry with `edge_max: 1` | `bernoulli` |
| other `precomputed_dimensions` entry | `gaussian` |

**When the auto-detected kind is wrong** — e.g. a `count` dim that is always
0 or 1 (a binary presence flag), or a count prepared upstream that would default
to `gaussian` — compute the column on the entity table and declare it as a
precomputed dimension, with `kind:` when the default does not fit:

```yaml
precomputed_dimensions:
  - column: cross_border_flag   # 0/1 column prepared in the source script
    edge_max: 1                 # -> bernoulli
  - column: n_counterparties    # count computed upstream
    kind: poisson               # gaussian without the override
```

**Impact on navigation:**
- `bernoulli` dims: `metric="Linf"` in `find_anomalies` catches single-flag
  deviations that L2 norm dilutes. Recommend Linf for patterns with 4+ bernoulli dims.
- `poisson` dims: anomaly = count structure deviation (structuring, burst). Bregman
  divergence is more sensitive than Euclidean for count data at low counts.
- `gaussian` dims: standard behavior. Bregman ≈ squared distance for well-calibrated
  gaussians — no practical difference from prior behavior for these dims.

For mixed-kind patterns, read the `bregman_divergence` that every
`find_anomalies` result carries next to `delta_norm` (distribution-aware
scoring). Check `dimension_kinds` in `sphere_overview` after build to verify
assignments.

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
  iteration (fast builds) or when the pattern uses `group_by_property` or gets the calibrated score
  (confidence is not emitted for those anyway).
- `bootstrap_iterations: 50` — faster builds, rougher confidence estimates. Acceptable
  for exploratory spheres where you only need coarse stable/unstable classification.
- `bootstrap_iterations: 500` — smoother confidence, recommended for production spheres
  where `min_confidence` is used as an alert threshold. Build time scales linearly.

**Note:** `anomaly_confidence` is not computed for `group_by_property` patterns (per-cohort
calibration) or for patterns with the calibrated score. It is also skipped when population > 50K — use
`conformal_p` instead.

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

### Label-aware calibration (`label_audit:`)

When the entity line carries a binary outcome column (e.g. `is_laundering`,
`is_fraud`, `default_flag`), declare a top-level `label_audit:` block to
unlock class-conditioned per-dim calibration. Build with the
`--label-aware-calibration` flag to activate; the builder runs
`engine.calibration_label_aware.calibrate_label_aware` per listed pattern
and persists `{mu_pos, sigma_pos, mu_neg, sigma_neg, direction}` per dim
plus a unit Fisher LDA direction vector onto `Pattern.label_aware_calibration`.
Sphere format stamps `3.1` when the block is declared; legacy 3.0 spheres
load unchanged.

```yaml
label_audit:
  label_column: is_laundering        # binary column on the entity line
  label_positive_value: 1            # value treated as the positive class
  patterns:
    - tx_pattern                     # patterns to calibrate (event or anchor)
```

What this unlocks downstream:
- `audit_pattern_dims(pattern_id, top_k=10)` returns the full-field path —
  per-dim Cohen's d separation + Fisher direction component +
  `recommended_action` ∈ {keep, split, drop_low_separation,
  investigate_drift, kind_mismatch_review}. Without the block the tool
  returns a fallback shape with raw mu/sigma only.
- `delta_norm_signed` Lance column populates with the per-polygon
  projection onto the Fisher LDA direction (positive ⟹ pushed toward
  positive-labelled centroid, negative ⟹ toward the other class).
  All-null on patterns without label-aware calibration.

The label column must be a column on the patterns' entity_line (for
event patterns: the event line itself; for anchor patterns: a column on
the anchor line). The block validates pattern names against the
`patterns:` registry; an unknown pattern fails the build.

Use when: the sphere has a ground-truth label that an investigator
would want to "show me dims the positive class deviates on". Skip when
no label column exists — the standard mu/sigma calibration path is
already sufficient.

### Declarative compliance rules (`conformance_rules:`)

Attach human-authored rules to a pattern in `sphere.yaml` so the builder
materializes a per-pattern violations table at build time. Independent
from `delta_norm` anomaly flags — an entity can be one, the other, or
both. Query the result via `find_conformance_violations(pattern_id,
rule_id, severity_min, top_n)` at runtime. AML adoption surface for
SAR-narrative composition: cite a broken rule by `rule_id`, attach
geometric explanation as supporting evidence.

```yaml
patterns:
  account_pattern:
    type: anchor
    entity_line: accounts
    conformance_rules:
      - rule_id: high_risk_kyc_missing
        severity: high
        description: "High-risk client without KYC complete"
        violates_when:
          op: and
          terms:
            - op: "=="
              prop: risk_band
              value: high
            - op: "!="
              prop: kyc_status
              value: complete
      - rule_id: cross_border_no_sanctions_check
        severity: critical
        violates_when:
          op: and
          terms:
            - op: "=="
              prop: cross_border_flag
              value: true
            - op: "in"
              prop: sanctions_check_status
              value: ["not_run", "expired"]
```

Predicate AST language: logical `and` / `or` / `not` over comparison
leaves `==` / `!=` / `<` / `<=` / `>` / `>=` / `in`. `prop` resolves to
a column on the pattern's points table. Severity is one of `low` /
`medium` / `high` / `critical`. Builder compiles each predicate to a
PyArrow `compute.Expression` (no `eval()`), evaluates row-wise, and
persists `(primary_key, rule_id, severity)` triples to
`_gds_meta/conformance/violations/{pattern_id}/v={N}.lance` alongside
`rule_set_hash` (rule changes invalidate the sidecar on next rebuild).

### Edge table design

The edge table stores per-event from/to relationships, enabling runtime graph
traversal (`find_geometric_path`, `extract_chains`, `edge_stats`) without
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
      timestamp_col: tx_date    # enables temporal ordering of hops in extract_chains
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
  temporal ordering in `extract_chains` (`time_window_hours`) will not work,
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
