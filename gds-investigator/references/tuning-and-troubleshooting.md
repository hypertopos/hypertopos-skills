# Tuning and Troubleshooting

FDR tuning for the suspect list and what to do when tools return nothing useful.

## Alarm budget and diverse selection

When tracing root causes, remember what the suspect list promises: On a calibrated pattern (non-empty `score_dims`), `is_anomaly` means `conformal_p ≤ 1 − anomaly_percentile/100` — the alarm budget: at the 95th percentile about 5 % of a clean population is flagged. Chasing false positives through the full root-cause chain wastes the entire investigation budget, so start from the smallest `conformal_p` (strongest evidence) and confirm with a second detector before going deep. Use `select="diverse"` when requesting K>10 results to surface anomalies driven by different dimensions rather than clustering on one dominant failure mode; this ensures the investigation covers distinct root-cause categories. `select` also applies to `attract_boundary`, `find_hubs` and `find_drifting_entities`.

**Stale calibration.** The budget holds against the stored calibration reference. When `check_alerts` reports `calibration_age` (reference older than 60 days), `recalibrate(pattern_id)` before relying on `conformal_p` for a decision.

**Prioritise deteriorating drift.** When `drift_direction` is `"deteriorating"`, the entity is structurally moving away from the null centre — investigation priority is higher than for equally-displaced entities with `"normalizing"` direction, which are already self-correcting. Use this to rank multiple drifting entities by risk rather than raw displacement.

**Decompose drift into intrinsic vs extrinsic.** `decompose_drift(entity, pattern)` (default args) splits the entity's drift between its first and last temporal slice into intrinsic (entity-driven shape change, σ from oldest retained calibration) and extrinsic (residual, population recalibration). The aggregate `intrinsic_fraction` answers: "Of the squared change, what proportion was the entity itself?" Use when `find_drifting_entities` flags an entity and you need to discriminate "this entity actually moved" from "the population calibrated around it". An entity at `intrinsic_fraction ≈ 0.05` didn't really change — recalibration shifted the coordinate system around it. An entity at `intrinsic_fraction ≈ 0.95` is the real signal. `find_drifting_entities` already attaches the same three scalars per entry for batch triage.

## When things don't work

- **Tool returns empty results** — try different parameters, wider sample,
  or a different pattern. Adjust `top_n` or filters.
- **Tool errors** — check pattern_id and version, verify sphere is open.
  An unknown parameter name is rejected; the error suggests the closest
  declared name when there is one ("did you mean 'top_n'?") — fix the
  name instead of dropping the argument.
  If the error persists, report it rather than retrying in a loop.
- **No anomalies found** — not every sphere has every anomaly type.
  Report "no signal detected" as a valid finding.
- **check_anomaly_batch returns all normal for "bad" keys** — the pattern
  may not capture the relevant signal. Try other patterns or composite_risk.
- **dive_solid returns no temporal slices** — temporal data may not exist
  for this pattern. Skip temporal root cause and note it in the report.
- **contrast_populations shows small effect sizes** — the anomaly may be
  spread across many dimensions rather than concentrated in one. Check
  individual dimensions rather than relying on the top-1.
