# Troubleshooting

Symptom → fix tables for sphere builds, calibration, and design.

## Troubleshooting

### Build issues

| Signal | Fix |
|--------|-----|
| "column not found" | Check schema with `pyarrow.parquet.read_schema()` |
| "unknown pattern" in temporal | Use `{composite_line_id}_pattern` or `{chain_line_id}_pattern` |
| Sources phase >60s | Pre-export to parquet (Tier 1) or optimize script |
| Chains phase >5min | Reduce `max_chains` or increase `min_hops`. |
| Temporal phase >5min | Temporal rescans the event table per window — scales as O(entities x windows). Use `--no-temporal` for geometry iteration, then add temporal as a final pass. Use larger `window` (fewer snapshots) to reduce window count |
| Build hangs during Geometry | Rule: group_count x population < 10M |
| "Too many bins" / anomaly_rate=50% | Entity line has <10 entities -- too few for population statistics. Remove pattern or merge with larger line |
| `pc.strftime` timezone error (Windows) | Cast timestamp to tz-naive first: `col.cast(pa.timestamp("us"))` before date operations |
| `degree_velocity` returns flat/uniform timestamps | Edge auto-detect picked `created_at` (Lance metadata) instead of business timestamp. Set explicit `timestamp_col` in `edge_table` config |
| `extract_chains` time window has no effect | Same root cause — edge table has metadata timestamp. Set explicit `timestamp_col` |
| Amount-weighted path scoring inactive | Auto-detect only matches `amount`, `value`, `total`, `amt`. For domain-specific names (`fare_amount`, `total_amount`), set explicit `amount_col` in `edge_table` config |
| `contagion_score > 1.0` or anomalous > total counterparties | NB-Split anchor resolution bug in an old build — rebuild the sphere |
| "output directory has branches" | A build never overwrites branches, even with `--force`. `hypertopos sphere branch list <sphere>/`, then promote or delete each branch before rebuilding |

### Calibration issues

| Signal | Fix |
|--------|-----|
| anomaly_rate = 0% | Lower `anomaly_percentile` to 90 |
| anomaly_rate > 20% | Raise `anomaly_percentile` to 99 |
| calibration_health = poor | Add `group_by_property` or adjust `gmm_n_components` |
| anomaly_rate higher with `group_by_property` | Expected -- per-group calibration is more sensitive |
| `sphere_overview` rate != `find_clusters` rate | Rebuild sphere with current version |
| A few entities distort μ/σ (`find_calibration_influencers` names them) | Try the fit without them on a branch: `recalibrate(pattern_id, branch="trial", exclude_keys=[...])` scores every entity against a calibration that leaves those keys out; `compare_branches(pattern_id, branch="trial")` shows flips and drift; `manage_branch(action="promote", name="trial")` keeps it, `manage_branch(action="delete", name="trial")` discards it |

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
