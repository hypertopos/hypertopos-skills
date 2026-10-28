# Population Analyses

Detector composition, cross-pattern lead-lag, hidden influencers and density gaps for the investigator skill.

## Lower-level p-value composition

**Lower-level p-value composition** (when detectors outside `cross_pattern_profile` matter
— external rules engines, typology scores, chain-coherent run p-values):

```
combine_anomaly_pvalues(pattern_id, detectors=["delta_norm", "neighbor_contamination"])
-> Runs each named built-in detector, combines per-entity p-values via Wilson HMP
   (robust under positive dependence). Returns entities ranked ascending by combined HMP.
-> Available detectors: delta_norm, neighbor_contamination, segment_shift,
   trajectory_continuous (opt-in: pass it in detectors; reads the temporal history of the
   sampled entities). density_gap is aggregate-only, no per-entity p.
classify_detector_consensus(pattern_id, detectors=["delta_norm", "neighbor_contamination"])
-> Categorical label per entity: anomalous_consensus / mixed_signal / normal_consensus /
   single_detector_signal. "mixed_signal" is itself a signal — entity at boundary of two typologies.
```

## Cross-pattern lead-lag

When investigating drift across multiple anchor patterns over the SAME entity
line (e.g. `account_behavior_pattern` × `account_stress_pattern`), use
`find_lead_lag(pattern_a, pattern_b)` to detect the temporal ordering of
population-level shifts.

**Default workflow:**

```
find_lead_lag(
    pattern_a="account_behavior_pattern",
    pattern_b="account_stress_pattern",
    cohort="fixed",            # panel-clean centroid signal
    fdr_method="storey",       # default — recovers power on rich-signal regimes
)
```

**Read the response in this order:**

1. `degenerate_signal` — if `true`, either centroid drift series has zero
   variance (population is constant per epoch). Treat as no signal and stop.
2. `agreement` — `"strong"` means centroid lag and volatility lag both
   agree on direction and magnitude (most trustworthy); `"weak"` means
   both channels show movement but disagree (population is heterogeneous);
   `"divergent"` means no coherent lead-lag — do not report a `lag` claim.
3. `is_significant` — `True` when `abs(correlation) > max_corr_threshold`
   (peak Bonferroni-adjusted threshold). Reliable headline gate.
4. `lag` — peak lag in epochs (positive = `pattern_a` leads `pattern_b`).
   Multiply by the temporal `window` (sphere config) for wall-clock units.
5. `top_dim_pairs` — even when `is_significant=False`, the top entries
   ranked by ascending `q_value` (then |corr|) surface the strongest
   leading dim pairs for hypothesis generation. The displayed pairs may
   not be FDR-significant on small `N` because Bonferroni-over-lags + BH
   over `D_A * D_B` pairs is intentionally conservative.

**Drill-down:** pass `primary_key=...` to replace the population centroid
with that entity's own delta trajectory. Useful for case storytelling
("for customer X, behavior_change preceded stress_change by 2 epochs"),
but per-entity reliability is `"low"` at most realistic N.

**Verbose mode:** `verbose=True` returns the full `D_A × D_B` matrix in
`per_dim_pairs` (sorted by ascending q-value); use when you need to scan
the full breakdown rather than the top 10.

**Limit reminder:** `find_lead_lag` requires both patterns to share the
underlying entity space — `cohort="fixed"` raises empty-cohort otherwise.
On disjoint entity spaces (e.g. accounts vs chains in AML) `cohort="all"`
typically lands in `degenerate_signal=true`.

## Find hidden influencers — entities defining what "normal" means

Standard anomaly detection asks "how far is this entity from normal?". The inverse question is "how much does this entity SHAPE what 'normal' means?". A hidden influencer has high impact on coordinate system calibration but LOW anomaly score — invisible to anomaly scans, yet removing it shifts μ/σ enough to flip other entities' classifications.

**Note:** `find_calibration_influencers` applies only to patterns scored by `delta_norm` (no `score_dims`). On calibrated patterns the anomaly verdict comes from a rank-based reference no single entity can mask itself in — use `explain_anomaly` to understand why a specific entity is flagged.

```
mcp__hypertopos__find_calibration_influencers(
    pattern_id="<pattern>",
    classify="hidden",
    top_n=10,
)
```

Returns entries with `total_impact` (high), `delta_norm` (low), `classification="hidden"`, `top_dim_contributions[]` (which dims drive the impact). For coordinated detection: collect 2-5 candidates and pass as one group to `find_group_influence` — `reinforcing_factor > 1.5` confirms coordinated pull.

Common operational triggers:
- **Data quality audit** — hidden influencer may be a duplicated record warping the entire coordinate system. Find → fix data → rebuild → previously-masked anomalies surface.
- **Adversarial AML** — coordinated account injection to shift coordinates and mask fraud is the canonical hidden-influencer signature.
- **Explaining anomaly flux** — when `compare_calibrations` shows population shifted between epochs, hidden influencer analysis explains WHY.

`verbose=True` adds `cascading_flip_count` per entry — count of OTHER entities that flip is_anomaly classification after this entity's removal. Use when the candidate has high impact but you need a quantitative "blast radius" before recommending exclusion.

## Anomaly by absence — `find_density_gaps`

`find_anomalies` surfaces entities at unusual positions in the geometry. `find_density_gaps` answers the inverse: which combinations of dim values **should** be populated under the independence null but are not? Useful when the investigation shifts from "who is anomalous" to "what's structurally missing".

```python
result = mcp__hypertopos__find_density_gaps(
    pattern_id="account_pattern", top_n=5,
)

for gap in result["gaps"]:
    print(
        f'{gap["dim_i"]} ∈ [{gap["delta_range_i"][0]:.2g}, {gap["delta_range_i"][1]:.2g}] '
        f'AND {gap["dim_j"]} ∈ [{gap["delta_range_j"][0]:.2g}, {gap["delta_range_j"][1]:.2g}] '
        f'— observed {gap["observed"]}, expected {gap["expected"]:.1f}, '
        f'q={gap["q_value"]:.2e}'
    )
```

**Range space note:** `delta_range_*` is in **z-score (delta) space**, NOT raw property units. Dim labels with the `_d_` prefix mean "delta of <property>"; e.g. `_d_tx_count ∈ [-0.6, -0.4]` is "tx_count z-scored against the population sits between -0.6 σ and -0.4 σ". The tool does not map ranges back to raw property units; inverse-transform from the points table if you want raw units in the narrative.

**When the gap is real:** independence + uniform-marginal expectation says the bin should hold ~N entities, observed is far below, q ≤ α after BH. Interpretation: there's structural avoidance of this combination — possibly business logic, regulatory cutoff, or a class of entities that exits when both features cross specific thresholds together.

**When the gap is a binning artefact (FP):** check the bin's entity count in the `find_density_gaps` output (`bin_count` field). If non-zero the cell was mis-binned at the histogram edge; deprioritise it. To drill further, use `find_anomalies(pattern_id, property_filters={"<prop_i>": {"gt": lo_i, "lt": hi_i}}, top_n=50)` on the suspect dimension range. The probe report and tests cover the boundary cases, but real-data gaps benefit from one-shot manual TP-verify before acting on them.

**Excluded dims:** bernoulli, degenerate, and very sparse (<30 finite values) dims are auto-excluded and reported in `excluded_dims` with `reason`. If an expected dim is missing from the gap report, look here first.
