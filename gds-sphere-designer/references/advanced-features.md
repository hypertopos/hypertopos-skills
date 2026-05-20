# Advanced features (AML / graph-heavy domains)

**Precomputed dimensions** — when you have ratio/score columns already on the entity table:
```yaml
precomputed_dimensions:
  - column: intermediary_score
    edge_max: 1
  - column: structuring_pct
    edge_max: 1
```

**Graph features** — auto-compute from event from/to columns:
```yaml
graph_features:
  event_line: transactions
  from_col: from_account
  to_col: to_account
  features: [in_degree, out_degree, reciprocity, counterpart_overlap]
```

**Chain lines** — extract multi-hop transaction chains:
```yaml
chain_lines:
  tx_chains:
    event_line: transactions
    from_col: from_account
    to_col: to_account
    features: [hop_count, is_cyclic, time_span_hours, amount_decay]
    max_chains: 300000
```

**Edge table** — auto-emitted for event patterns with 2+ FK relations to same anchor line. Optional explicit config:
```yaml
# Edge table enables find_geometric_path, discover_chains, edge_stats
# MCP tools at navigation time.
# Build cost: near zero (data already in memory during geometry build).
# Skip with --no-edges CLI flag.
patterns:
  tx_pattern:
    edge_table:
      from_col: sender_id
      to_col: receiver_id
      timestamp_col: tx_date    # optional
      amount_col: amount        # optional
```

**Edge dimensions** — opt-in build-time per-edge dims that feed the polygon shape and the per-edge predicate filter. Six available:

```yaml
patterns:
  tx_pattern:
    edge_dimensions:
      - pair_edge_count            # Poisson — count of edges in this (from, to) pair
      - position_in_chain:         # Poisson — ordinal position in a detected chain
          min_position: 5
      - time_since_pair_last_edge  # Gaussian — seconds since pair last had an edge
      - pair_amount_zscore         # Gaussian — per-pair amount z-score (LOW_VAR pairs only)
      - find_motif_structuring     # Bernoulli — 1 if edge sits in a structuring motif
      - edge_curvature_frc         # Gaussian — combinatorial Forman-Ricci curvature
```

`edge_curvature_frc` is the only purely structural dim — it's `4 − deg(u) − deg(v) + #triangles(u, v)` on the undirected projection of the multigraph (presence not multiplicity; canonical for both directions). Negative values mark sparse-hub topology (lookalike mule signature); near-zero marks tight rings (K-clique sub-structures); positive marks over-clustered local neighbourhoods. Composes through the existing population-relative anomaly stack via aggregated `*_mean` / `*_max` on the anchor pattern.

**Mahalanobis distance** — for correlated dimensions:
```yaml
patterns:
  account_pattern:
    use_mahalanobis: true
```
