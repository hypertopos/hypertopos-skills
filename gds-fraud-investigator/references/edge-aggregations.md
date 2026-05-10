# Edge-Dim Aggregation Cheatsheets

> Loaded on demand when designing or reading aggregated edge-dim columns on anchor or chain anchor patterns. The `edge_dim_aggregations:` YAML block is opt-in — these cheatsheets document the per-aggregate semantic and how to read it on a suspect.

## Account-level recall via aggregated edge dims

An anchor pattern can pull the per-edge sidecar dims up to per-anchor-entity columns by declaring an `edge_dim_aggregations:` block:

```yaml
patterns:
  account_pattern:
    type: anchor
    entity_line: accounts
    edge_dim_aggregations:
      from: tx_pattern              # event pattern that emitted the sidecar
      dims: [pair_edge_count, find_motif_structuring]   # all five aggregates per dim
    relations:
      ...
```

For each declared source dim, the builder bakes the five canonical aggregates into the anchor polygon: `<dim>_mean`, `<dim>_max`, `<dim>_std`, `<dim>_p95`, `<dim>_count_above_threshold` (count of edges whose source-dim value exceeds the population p95 cutoff persisted in the calibration epoch). Account-level investigation reads them off `find_anomalies` and `explain_anomaly` exactly the same way as any other dim — no new MCP tool, no new flag.

When five aggregates per dim is more than you need, switch `dims:` from list to mapping and pick a per-dim subset:

```yaml
    edge_dim_aggregations:
      from: tx_pattern
      dims:
        pair_edge_count: [count_above_threshold]
        find_motif_structuring: [mean, max]
```

This emits only three aggregated columns (`pair_edge_count_count_above_threshold`, `find_motif_structuring_mean`, `find_motif_structuring_max`) instead of ten — useful when narrow signals dominate the investigation and the extra columns dilute `delta_norm`.

**Reading aggregates on a suspicious account:**
- `pair_edge_count_max` very high → account participates in at least one heavily-recurring pair (counter-party concentration risk)
- `find_motif_structuring_mean` materially > 0 → meaningful share of the account's transactions sit inside a structuring motif chain
- `position_in_chain_max` high → account sits near the deep end of multi-hop chains (downstream sink in long layering)
- `time_since_pair_last_edge_mean` low + `pair_edge_count_max` high → bursty re-activation of dormant pair edges (classic flash-burst signature on the account level)

These lift account-level recall on workflows where the per-tx geometry signal is faint (transaction polygons calibrate against a multi-million-row population so a single anomalous tx is hard to surface alone) but accumulates clearly at the entity level (10 anomalous transactions out of 30 hop into a single account's `find_motif_structuring_mean`).

**Anchor regimes supported:** `single` (account-style, anchor PK matches edge `from_key` OR `to_key`), `pair` (composite k=2 anchor like `account_pairs`, PK encoded as `<from><separator><to>` per the `composite_lines:` block — separator defaults to `→`), `chain` (auto-emitted from `chain_lines:`), and k>2 composite anchors (tripartite and beyond, where `key_cols[0:2]` map positionally to edge endpoints and remaining `key_cols` join on the event_table).

## Chain-level recall via aggregated edge dims

A chain-anchor pattern (auto-emitted from a `chain_lines:` YAML block) can pull the per-edge sidecar dims onto per-chain columns by adding the `edge_dim_aggregations:` block inside `chain_lines:`:

```yaml
chain_lines:
  tx_chains:
    event_line: transactions
    from_col: from_account
    to_col: to_account
    features: [hop_count, is_cyclic, time_span_hours]
    edge_dim_aggregations:
      from: tx_pattern                  # event pattern that emitted the sidecar
      dims: [find_motif_structuring, position_in_chain]
```

For each declared source dim, the builder bakes the five canonical aggregates into the chain anchor polygon: `<dim>_mean`, `<dim>_max`, `<dim>_std`, `<dim>_p95`, `<dim>_count_above_threshold`. Chain-level investigation reads them off `find_anomalies` / `explain_anomaly` exactly the same way as any other dim — no new MCP tool, no new flag. Switch `dims:` to mapping form to pick a per-dim subset and avoid the polygon-dim balloon when only narrow signals matter.

**Reading aggregates on a suspicious chain:**
- `find_motif_structuring_mean` materially > 0 → meaningful share of the chain's hops sit inside a structuring motif; the chain itself is a layering candidate
- `position_in_chain_max` very high → at least one hop in the chain is embedded deep inside a longer chain (chain-of-chains topology, classic layering through intermediate accounts)
- `pair_edge_count_mean` very high → all hops cluster on already-recurring pairs (chain re-uses known counter-party flows; counter-party concentration signal at the chain level)
- `find_motif_structuring_max == 1.0` AND `find_motif_structuring_mean` low → only one or two hops are structuring-like; chain is a partial-overlap case rather than a clean structuring chain

**Mechanism — when chain-level aggregation is hypothesised to help:** chain-level signal accumulates over *paths*, not over *entities*. An account that hosts five anomalous transactions out of 30 has 5/30 ≈ 17% account-level structuring rate. A chain that strings those five transactions together has 5/5 = 100% chain-level rate. The chain regime is therefore *hypothesised* to surface the layering pattern that would otherwise dilute into the account-level noise floor.

**Empirical lift on real labels — TBD.** Treat this entry as architectural completeness for the agent vocabulary, not as a confirmed-lift recall booster. The aggregated edge dims expose a structurally consistent feature surface, but no public benchmark has yet shown a measurable AUROC delta from these aggregations alone against ground-truth laundering labels. Prior on signal lift is correspondingly low until the upstream `find_motif_structuring` / `position_in_chain` predicates are validated on a labelled dataset where they discriminate above the population base rate. See [`correlation-gates.md`](correlation-gates.md) for the gate machinery that closes this evidence gap on labelled spheres.

**Anchor regimes supported:** chain anchor patterns auto-emitted via `chain_lines:` block (BFS extraction at build time), AND external chain tables ingested as anchor lines per the convention. Membership lookup via the `chain_keys` property column on the chain anchor line (comma-joined member primary_keys in chain order, populated at chain extraction time OR by the upstream system that emitted the chains). See `packages/hypertopos-py/docs/external-chains-as-anchor-line.md` for the external-chain workflow (SAR typology engines → anchor line → R9 loop, with the `chain_keys` column as the convention column that unlocks the chain-coherent primitives on externally-curated chains).
