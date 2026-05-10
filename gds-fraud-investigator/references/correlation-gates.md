# Feature Curation via Stratified Correlation Gates

> Loaded on demand when curating which features should anchor a SAR rationale or a triage rule, given the recurring "correlated with confounder X" failure mode for raw account / chain features. Documents the three-lens approach to feature evidence (all-population, stratified per-bucket, partial point-biserial) and the runtime knob that wires gate verdicts into ranking.

Before relying on a chain or account feature in a SAR narrative or a triage rule, sanity-check whether the feature carries laundering signal independent of the obvious confounders (chain length, account transaction volume). The hypertopos benchmark/ harnesses surface a per-feature verdict on labelled spheres along three lenses, in increasing methodological strictness:

1. **All-population correlation gate.** Welch t-test + Mann-Whitney U per feature against the laundering label across the whole pattern. Cheapest. Verdict: PASS / MARGINAL / FAIL. Limitation: confounded with length / volume — a feature that's just correlated with chain length will get a mechanical PASS because long chains have higher base-rate laundering exposure. Mark length-correlated PASSes as suspect by default.

2. **Stratified correlation gate.** Bucket the population by the confounder (hop_count for chains, tx_out_count + tx_in_count for accounts) and re-run the gate within each bucket. Cross-bucket verdict:
   - **ROBUST**: PASSes in every bucket AND Cohen d direction is consistent across passing buckets. Real signal independent of the confounder.
   - **DIRECTION-INCONSISTENT**: PASSes everywhere but Cohen d sign flips across buckets. Statistical separation real, but interpretation depends on the confounder regime — different bucket, different mechanism.
   - **LENGTH-MEDIATED / VOLUME-MEDIATED**: PASSes in some buckets, FAILs in others. Signal exists at one end of the confounder spectrum only.
   - **NOISE**: FAIL or MARGINAL in every bucket. No real signal — the all-population PASS (if any) was an artefact.

3. **Partial point-biserial correlation.** Across the full population, residualise the feature on the confounder via linear regression and compute the partial r against the laundering label. Significant partial r with non-trivial magnitude (>0.05 typically) is the strongest evidence of confounder-independent signal. Per-bucket ROBUST + low partial r means "consistently weak signal across strata" — real but small.

**When to use which.** All-population gate is the right first-pass screen. Stratified gate is the bias-controlled follow-up — run it when an all-population PASS is on a feature that's correlated with the natural confounder. Partial r is the headline number to put in a SAR rationale when the question is "is this feature driving the alert independent of the entity's size."

**Verdict interpretation in SAR / triage context.** A NOISE verdict is a hard signal that the feature shouldn't anchor a SAR rationale — even if `find_anomalies` flagged the entity high on this feature, the underlying separation from clean entities isn't real at population scale once confounders are controlled. A DIRECTION-INCONSISTENT verdict means the feature behaves differently in different regimes — investigators should match the entity's regime (short / long chain, low / high volume) to the per-bucket Cohen d sign before claiming directional rationale. ROBUST + high partial r is the cleanest evidence; lean on those features.

**Empirical sparseness caveat.** The intersection (ROBUST per-bucket AND |partial r| > 0.05) is often small — typically 1–3 features per pattern on real-world spheres. When the intersection is empty or near-empty for the pattern in question, fall back to ROBUST-partial-r-only as second-tier evidence (volume-/length-mediated per-bucket, but confounder-residualised). Treat that fallback as "best available evidence given confounder shape" rather than confirmed signal — and note in the SAR rationale that the feature's per-bucket behaviour was mixed.

**Heterogeneous label hypothesis.** When multiple features classify as DIRECTION-INCONSISTENT on the same population, the underlying issue may be that the laundering label aggregates several typologies — different rings behave differently, and direction-inconsistency reflects that heterogeneity rather than per-feature noise. Stratifying further by typology (when sub-labels are available) before per-feature analysis can resolve the inconsistency.

## Wiring gate verdicts into runtime ranking

Once a feature is classified as NOISE, DIRECTION-INCONSISTENT, or VOLUME-MEDIATED, surface the verdict in `find_anomalies` ranking via the `dimension_weights` parameter — it accepts a `{dim_name: float}` mapping that scales each dim's contribution to the rank score before computing `delta_norm`. Missing dims default to `1.0`; explicit `0.0` silences a dim. Requires `metric` in `"L2"` or `"Linf"`.

- **NOISE → `0.0`.** The dim has no real signal once confounders are controlled — silence it.
- **DIRECTION-INCONSISTENT or VOLUME-MEDIATED → `0.5`.** Real but interpretation depends on the regime — keep at half weight as soft demotion.
- **ROBUST + high partial r → leave at `1.0` (default).**

```python
# Account pattern triage that respects gate verdicts:
nav.find_anomalies(
    "account_pattern",
    top_n=100,
    dimension_weights={
        "return_ratio": 0.0,        # NOISE — fail every bucket
        "amount_out_std": 0.5,      # VOLUME-MEDIATED — soft demote
    },
)
```

The default `find_anomalies` call (no weights) is unchanged — opt in only when the gate verdict warrants it. Useful for SAR rationale: "we ranked entities discounting features that fail the bias-controlled correlation gate."
