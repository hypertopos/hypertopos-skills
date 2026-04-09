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

**Mahalanobis distance** — for correlated dimensions:
```yaml
patterns:
  account_pattern:
    use_mahalanobis: true
```
