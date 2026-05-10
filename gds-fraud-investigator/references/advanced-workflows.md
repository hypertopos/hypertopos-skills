# Advanced Operational Workflows

> Loaded on demand for adversarial-AML, group-coordination, lead-lag, and edge-feature workflows that go beyond the per-suspect Phase 2 playbook in `SKILL.md`. These compose the standard primitives into specific operational scenarios.

## Detect coordinated population-shift attack

Adversarial AML scenario: attackers inject N coordinated accounts each looking near-normal individually, but together they shift μ/σ enough to mask fraudulent outliers as "within population norm". Single-account anomaly detection misses them — they pass under the threshold by design. The signal is at the GROUP level: multiple accounts pulling μ/σ in the same direction.

```python
# Step 1: identify candidate set via existing fraud heuristics
candidates = find_witness_cohort(...)  # or anomalous_edges, cluster_bridges
member_keys = [...]  # 2-N entity_keys

# Step 2: check group-level coordination
mcp__hypertopos__find_group_influence(
    pattern_id="<account_pattern>",
    groups=[member_keys],
)
```

Interpret `reinforcing_factor`:
- `> 1.5` → reinforcing — group together moves μ/σ MORE than sum of individual impacts. Signature of coordinated pull (collusion ring, mule network, duplicate records).
- `< 0.5` → canceling — members offset each other. Likely uncoordinated outliers.
- `[0.5, 1.5]` → ambiguous, run other typology checks.

Companion: `find_calibration_influencers(classify="hidden", verbose=True)` — finds individual hidden influencers + their `cascading_flip_count` showing how many other entities flip is_anomaly upon their removal. High cascading flip count is independent evidence of strong calibration influence.

## AML hidden-influencer triage → SAR candidate workflow

When investigating a financial sphere (account-level pattern with adversarial-typology dimensions), hidden influencers right below the anomaly threshold are the highest-value SAR (Suspicious Activity Report) candidates: they look "normal" to anomaly detection BUT calibration of typology-relevant dims rests on them. Workflow:

```python
# Step 1: surface candidates — top 10 by total_impact among non-anomalous accounts
result = mcp__hypertopos__find_calibration_influencers(
    pattern_id="<account_pattern>",
    classify="hidden",
    top_n=10,
    verbose=True,  # adds cascading_flip_count
)
```

```python
# Step 2: per-entity triage — flag candidates whose top_dim_contributions
# include >=3 of the AML adversarial-typology atoms:
ATOMS = {
    "_d_amount_uniformity",       # equal-amount structuring signature
    "_d_return_ratio",            # round-trip / return chain signature
    "_d_fan_asymmetry",           # in-degree vs out-degree imbalance
    "_d_structuring_pct",         # below-CTR-threshold transaction share
    "_d_counterpart_overlap",     # cluster overlap with known anomalous
    "_d_intermediary_score",      # pass-through layering signature
}

flagged = []
for entry in result["entries"]:
    top_dim_labels = {d["dim_label"] for d in entry["top_dim_contributions"]}
    aml_signature_count = len(top_dim_labels & ATOMS)
    if aml_signature_count >= 3 and entry["delta_norm"] >= 0.7 * result["theta_norm"]:
        flagged.append({
            "entity_key": entry["entity_key"],
            "aml_signature_count": aml_signature_count,
            "matched_atoms": list(top_dim_labels & ATOMS),
            "cascading_flip_count": entry["cascading_flip_count"],
            "delta_norm": entry["delta_norm"],
        })
```

The `delta_norm >= 0.7 * theta_norm` filter selects accounts AT (or near) the anomaly threshold — they're the ones the typology-relevant dims actually support. Accounts FAR below threshold don't tell you much about typology calibration even if they have high `total_impact`.

```python
# Step 3: group test on flagged set — is this a collusion ring?
if len(flagged) >= 2:
    group_result = mcp__hypertopos__find_group_influence(
        pattern_id="<account_pattern>",
        groups=[[e["entity_key"] for e in flagged]],
    )
    if group_result[0]["reinforcing_factor"] > 1.5:
        # Coordinated — escalate as ring, not individual SARs
        ...
```

Single-entity SARs go to individual review; ring-detection (reinforcing > 1.5) goes to escalation. Real production AML data will mostly show `reinforcing_factor ≈ 1.0` (similar profiles, not coordinated) — the >1.5 case is the high-value adversarial signal.

## Cross-pattern lead-lag for AML rapid-escalation

`find_lead_lag` detects whether population-level shifts in pattern A precede shifts in pattern B. For AML this is the "rapid-escalation" signature — behavior shifts before structuring/layering signatures ramp up.

**Hard prerequisite:** both patterns must observe the SAME entity line. Calling `find_lead_lag(pattern_a="account_pattern", pattern_b="tx_chains_pattern")` raises `cohort='fixed' produced empty cohort` because account_pattern is over `accounts` and tx_chains_pattern is over `chains`. Spheres typically ship with one anchor pattern per entity space, so cross-pattern lead-lag is **not directly usable** without rebuilding the sphere with two anchor patterns over the same entity line (e.g. `account_behavior` + `account_stress` analogue).

**If the sphere has two account-level anchor patterns, the workflow is:**

```python
result = mcp__hypertopos__find_lead_lag(
    pattern_a="<account_behavior_pattern>",
    pattern_b="<account_stress_pattern>",
    cohort="fixed",
    fdr_method="storey",
)
```

**Reading the response on AML cadence (`window=2d`, N=9):**

- `reliability` = `"low"` (N - 1 < 12) — calibration of significance is wide.
- `bartlett_ci_95` ≈ 0.65; `max_corr_threshold` ≈ 0.86. Only very strong peaks survive.
- `is_significant=True` with `agreement="strong"` is the green light for SAR narrative; otherwise log the headline lag/correlation as exploratory only and do not claim lead-lag in the SAR body.
- `degenerate_signal=True` means at least one centroid drift series has zero variance — typical when temporal data is near-constant per (entity, epoch). Stop and rebuild the sphere with finer `window` or richer per-epoch features.

**Cross-validate with hidden-influencer triage:** entities flagged hidden-influencer with `delta_norm ≥ 0.7 * theta_norm` AND appearing in the leading dim of `top_dim_pairs` are SAR-priority candidates — they sit near θ on a behaviorally-leading dimension whose centroid shifts before the stress-pattern centroid does.

**For production-grade lead-lag:** the blocker is structural, not cadence. Today's typical AML config has one anchor pattern per entity space — `cohort="fixed"` raises empty-cohort because there is no shared entity space. Rebuild the sphere with two anchor patterns over the SAME entity line and lead-lag becomes well-defined. Window-only rework (`window=12h` for finer N) does not help: the population centroid is dominated by `√N` averaging over hundreds of thousands of entities, so its drift series is near-constant in absolute units — the limit is the architecture, not the temporal resolution.

If `find_lead_lag` raises with `cohort upper bound would build at least X.XX GB of shape tensors`, the patterns are over disjoint entity_lines and cross-pattern lead-lag is undefined. Fall back to `cohort='fixed'` (will raise empty-cohort with the same diagnostic) or pass `entity_key=<id>` for per-entity drill-down.

## Edge-derived dimensions for AML detection

Spheres can declare an `edge_dimensions:` block on event patterns that emit transactions as edges (any pattern with an `edge_table:` block). Five build-time per-edge dim functions land on every event polygon:

| Dim | Type | Signal |
|---|---|---|
| `pair_edge_count` | poisson | counts edges in (from, to) directed pair across the full sphere span — high values flag concentration / pair-locking |
| `position_in_chain` | poisson | depth in the longest reverse-temporal chain ending at this edge — high values flag the entity is N hops downstream from a structuring-likely source |
| `time_since_pair_last_edge` | gaussian | seconds since the previous edge in the same pair; first edge in a pair gets a sentinel = sphere span |
| `pair_amount_zscore` (LOW_VAR pairs only) | gaussian | signed z-score of amount within (from, to) pairs whose CV(amount) < `cv_threshold`; HIGH_VAR pairs and pairs below `min_count` return zero |
| `find_motif_structuring` | bernoulli | 1.0 if this edge participates in any A→B→C→D structuring motif within `time_window_hours` with hop1 ≥ `amt1_min` and hops 2/3 ≤ `amt2_max` |

YAML surface — drop into the event pattern's stanza:

```yaml
patterns:
  tx_pattern:
    type: event
    entity_line: transactions
    edge_table:
      from_col: from_account
      to_col: to_account
      timestamp_col: ts
      amount_col: amount
    edge_dimensions:
      - pair_edge_count
      - position_in_chain:
          min_position: 5            # MUST be >= 3 — pos2+ flags 56% of population
      - time_since_pair_last_edge:
          burst_seconds: 60
          dormant_seconds: auto      # resolves to sphere span at build time
      - pair_amount_zscore:
          cv_threshold: 0.05
          min_count: 3
      - find_motif_structuring:
          time_window_hours: 1.0
          amt1_min: 10000
          amt2_max: 10000
```

Validation rejects: `min_position < 3`, `cv_threshold` outside (0, 1], `min_count < 2`, non-positive `amt1_min` / `amt2_max` / `time_window_hours`, negative `burst_seconds`, duplicate dim entries, edge_dimensions on anchor patterns. Each entry can be either a bare string (use defaults) or a single-key dict with overrides.

Reading the dims at investigation time: `find_anomalies` (or any other primitive) returns each event polygon's `delta` vector with the new dims appended. Their order matches the order in `pattern.dimension_kinds` — read both side-by-side. The new dims contribute to `delta_norm`, anomaly classification, similarity search, clustering, and per-dim Bregman ranking exactly the same way relations and `event_dimensions` do.

A persisted per-edge sidecar Lance dataset lives at `_gds_meta/edge_features/{pid}/data.lance` with one row per `event_key` and one column per declared dim. No current primitive reads it on the navigator hot path; it is forward-compat for a future HopPredicate query API that would let agents express motif queries with predicates on these dim values.

**When NOT to enable:** anchor patterns, patterns with no `edge_table` (the dims have nothing to compute over), and patterns where the entity_line is not 1:1 with edges (only event patterns satisfy `primary_key == event_key`).
