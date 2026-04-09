# Design Patterns

## Entity line isolation

When multiple patterns share the same entity line, the builder auto-discovers
ALL precomputed dimensions from that line's table. A stress pattern on the
`accounts` line inherits behavioral dims (tx_count, burst_monthly) from other
patterns — diluting its signal.

**Why it matters:**
- Stress pattern on shared line: many dims (intended + inherited) → recall drops significantly
- Same pattern on isolated line: only intended dims → recall improves dramatically
- Same data, same columns — only the entity line isolation changed

**Fix:** Give focused patterns their own entity line (same source, separate namespace):

```yaml
lines:
  accounts:
    source: accounts
    key: primary_key
    role: anchor
  accounts_stress:       # ← separate line, same data
    source: accounts
    key: primary_key
    role: anchor

patterns:
  account_behavior_pattern:
    entity_line: accounts           # shares line with other behavioral patterns
    derived_dimensions: [tx_count, n_banks, burst_monthly, ...]

  account_stress_pattern:
    entity_line: accounts_stress    # ← isolated, gets ONLY its declared dims
    precomputed_dimensions: [penalty_interest_count, balance_volatility, min_balance, balance_trend]
    derived_dimensions:
      - from_pattern: tx_pattern
        anchor_fk: account_id       # needed when entity line ≠ event FK target
        features: [amount_std, mean_balance]
    relations: []                   # no FK relations, pure continuous
```

**Rule:** If a pattern needs exactly N dims, give it its own entity line.
Shared entity lines cause dim inheritance. 6 focused dims > 18 diluted dims.

---

## NB-Split pattern design

**Problem:** One pattern with 17 dimensions averages ALL signals. If 12 dims
are behavioral (normal for defaults) and 5 are credit risk (discriminative),
the 12 dilute the 5. Recall drops because defaults are not behavioral outliers.

**Solution (HPL-2009-225):** Split dimensions into orthogonal groups — each
group captures an independent aspect. Then combine signals via Fisher's method
(`composite_risk` / `passive_scan`). This is equivalent to Naive Bayes
assumption between tables (inter-table independence) from the paper.

```yaml
# BAD: one mega-pattern with mixed concerns
account_mega_pattern:
  derived_dimensions: [tx_count, n_banks, burst_daily, ..., penalty_interest, balance_volatility, ...]
  # 17 dims, behavioral noise dilutes risk signal

# GOOD: orthogonal patterns, combined via composite_risk
account_behavior_pattern:    # activity, diversity, burst
  derived_dimensions: [tx_count, n_banks, n_categories, burst_daily, burst_monthly]

account_stress_pattern:      # financial distress signals
  precomputed_dimensions: [penalty_interest_count, balance_volatility, min_balance, balance_trend]

account_loan_pattern:        # loan exposure
  precomputed_dimensions: [balance_to_loan, income_coverage]
```

**How it works:**
- Each pattern independently flags anomalies from its own perspective
- `composite_risk(entity_key)` combines p-values via Fisher's method
- `passive_scan(line_id, threshold=2)` finds entities flagged in 2+ patterns
- A default may be normal in behavior but anomalous in stress + loan →
  `source_count=2`, `combined_p` is low → detected

**Expected impact (order of magnitude):**

| Configuration | Typical recall |
|--------------|---------------|
| Single mega-pattern (10+ dims, shared line) | Low (~30%) |
| NB-Split patterns (shared line) | Medium (~60-70%) |
| Isolated pattern (own line, 3-8 dims) | High (~80-90%) |
| Isolated + cross-line Fisher composite | Highest (~85-95%) |

Key insight: fewer focused dims on an isolated entity line outperforms both
mega-patterns and multi-pattern composite scoring. Cross-line Fisher bridging
recovers additional borderline entities that are elevated in 2+ patterns.

**Cross-line bridging:** Lines sharing the same `source` in sphere.yaml are
recognized as sibling lines at runtime. `composite_risk` and `passive_scan`
automatically bridge across sibling lines — combining p-values via Fisher's
method. This requires no YAML changes beyond using the same `source:` value.

**Design rules:**
- Each pattern should capture ONE logical concern (behavior vs stress vs exposure)
- Dimensions within one pattern CAN be correlated (intra-pattern dependency OK)
- Give focused patterns their own entity line (see Entity line isolation above)
- Same `source:` value on sibling lines enables automatic cross-line composite scoring
- 3-6 dims per pattern is the sweet spot — enough signal, no dilution
- Use different `anomaly_percentile` per pattern if needed (e.g. 95 for behavior, 90 for stress)

**NB-Split tradeoff — cross-pattern detection tools:**

NB-Split puts each pattern on its own isolated entity line. This is optimal for
calibration and recall, but creates a visibility gap for `detect_cross_pattern_discrepancy`
and `passive_scan` — these tools discover patterns covering the SAME entity line.
If two patterns are on sibling lines (same source, different entity_line), cross-pattern
discrepancy detection cannot compare them directly.

| Design | Calibration | cross_pattern_discrepancy | passive_scan | composite_risk |
|--------|-------------|--------------------------|--------------|----------------|
| Shared line (2+ patterns) | Diluted (dims inherited) | Works | Works (per-line) | Works |
| NB-Split (isolated lines, same source) | Optimal (focused dims) | Does NOT work | Works (cross-line bridging) | Works (cross-line bridging) |

**Recommendation:** Use NB-Split for recall. Agents that need cross-pattern discrepancy
detection should use `passive_scan(line, threshold=1)` + `composite_risk` on sibling
lines instead — these bridge across lines automatically via shared `source:`. The
`detect_cross_pattern_discrepancy` tool documents this limitation in its docstring.

---

## NB-Split vs. multi-source pipeline: when to use which

NB-Split isolates orthogonal concerns into separate patterns and combines them via
Fisher's method. This works when each concern group has a **distinct geometric
signature** — different top dimensions, different entity populations.

Some domains require a different primary strategy: **multi-source pipeline** —
independent geometry types (account + pair + chain + heuristics) each adding unique TPs.

**Observed on IBM AML HI-Small (5M tx, 515K accounts, 7 laundering types):**

`n_currencies_out` drove detection across all 7 laundering pattern types (2.8-9.7x
separation ratio). Per-typology detectors on isolated lines yielded 45.4% recall vs
75.2% for the full multi-source pipeline — worse despite fewer dims. The right isolation
was at the **geometry level** (account / pair / chain / heuristics), not within the
account pattern itself.

| Geometry | What it captures | Unique TP |
|----------|-----------------|-----------|
| Account pattern | Behavioral profile per entity | core |
| Composite (A→B pairs) | Relationship-level anomalies | +significant |
| Chain lines | Multi-hop money flows | +significant |
| Heuristic sources | Cross-currency, structuring, borderline | +significant |

**Before applying NB-Split**, verify with `contrast_populations` that the concern groups
you want to isolate have distinct top dimensions. If the same 1-2 dimensions dominate
ALL concerns, NB-Split will produce correlated patterns — use multi-source pipeline
instead.

**Note:** This is an early observation from a single AML benchmark. The general
principle for complex multi-signal domains is still being developed.

---

## Dimension budgeting

3-6 dims per pattern. More dims = more noise. Each dim should answer a
distinct question about the entity from the pattern's concern.

Ask for each dim: "does this help detect THIS concern?" If no, it belongs
in a different pattern.

1 dim per pattern = too little. delta_norm = single z-score, rarely
exceeds theta alone. A 1-dim pattern typically achieves ~10% recall.
Same dim in a 5-dim pattern -> ~80% recall.

10+ dims per pattern = too much. Signal diluted by noise dims.

**Sweet spot: 3-6 dims per isolated concern.**
Each dim must answer a question about THAT concern. If a dim answers
a different question, it belongs in a different pattern.

---

## NB-Split for temporal: isolate drift dimensions

The same NB-Split principle that works for static detection also applies
to drift detection. If a pattern has 5 dimensions and one (burst_monthly)
has 40x the variance of another (n_suppliers), overall drift displacement
is dominated by burst — a real relational change (8 new suppliers) is
invisible.

**Fix:** create a separate pattern on an isolated entity line with ONLY
the relational dimensions (n_suppliers, n_parts). Temporal drift on
this 2-dim pattern is purely relational — no activity noise.

This works because drift displacement = L2 norm of per-dimension diffs.
With 2 focused dims, a spike on one dim dominates the norm. With 5 mixed
dims, the spike is one component among many.

**Rule:** if you need to detect relational changes over time, isolate
relational dimensions (count_distinct, diversity) into their own pattern.
Activity dimensions (count, sum, burst) belong in the activity pattern.

**When to apply:** any sphere where:
- Temporal drift detection is required
- The activity pattern mixes volume dims with relationship dims
- Expected anomalies include "new relationships" (new suppliers,
  new counterparties, new categories)

---

## Composite pattern dimension selection

Composite lines (entity-pair patterns) need dimensions that directly
capture the anomaly signal, not just aggregates that dilute it.

**Aggregate-only dims miss targeted anomalies.** If a supplier-part pair
has 20 normal transactions and 3 inflated ones, `sum` and `std` show
moderate elevation — lost among organically large pairs. But `avg` and
`max` directly surface the per-transaction inflation.

**Rule:** for composite patterns, always include at least one
per-transaction metric (avg, max) alongside aggregate metrics (count,
sum, std). The aggregate captures volume anomalies; the per-transaction
metric captures value anomalies.

Typical composite dimension set:
- `count` — relationship intensity
- `sum:<amount>` — total exposure
- `std:<amount>` — pricing variance
- `avg:<amount>` — per-unit value (catches targeted inflation)
- `max:<amount>` — peak value (catches one-off extremes)

---

## High-variance dims drown drift

If one dimension has 40x the variance of others, temporal drift ranking
will be dominated by that dimension. Other dims' real changes become
invisible.

Example: a low-cardinality dimension (7 discrete values, high kurtosis)
can dominate drift ranking, hiding real changes on other dimensions.
Removing or isolating it preserves static recall and unblocks drift.

**Rule:** check `anomaly_dimensions` on a few top anomalies. If one dim
contributes >80% of delta_norm across most anomalies, it's drowning the
rest. Either remove it or isolate it.

---

## Design review checklist

When reviewing an existing sphere:

| Check | Red flag | Fix |
|-------|----------|-----|
| Pattern has >10 dims | Signal dilution | Split into 2-3 focused patterns |
| Pattern has 1-2 dims | Insufficient delta_norm | Merge with related concern or add 1-2 more dims |
| Multiple patterns share entity line | Dim inheritance | Isolate with separate line |
| `profiling_alerts` same dims on 2+ patterns | Same dim declared in multiple patterns | Each dim should belong to one concern — move to the pattern it best serves |
| Extreme profiling_alert (ratio >3.0) | Outlier entities dominate scale | Investigate: genuine signal or noise? Cap/remove if noise |
| Recall on ground truth <30% | Wrong dims or wrong concern | Check contrast_populations — which dims actually separate? |
| One dim >80% of delta_norm | Variance domination | Remove or isolate the dominant dim |
| All anomalies are same archetype | Pattern captures one signal only | Check if the detected archetype is the one you care about |
| anomaly_rate >20% | Theta too low or population heterogeneity | Add group_by_property or raise anomaly_percentile |
| Dead dimensions (zero variance) | Dim contributes no signal | Remove from pattern |
| Two anomaly types share a dimension | mu/sigma shift masks both | Ensure alternate detection path via different pattern |
| Temporal drift dominated by one dim | Relational changes invisible | Isolate relational dims into own pattern (NB-Split for temporal) |
| Composite anomalies uniform by count | Cannot distinguish sub-populations | Add per-transaction dims (avg, max) not just aggregates (sum, std) |
