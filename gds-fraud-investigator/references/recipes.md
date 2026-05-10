# Edge Table Investigation Recipes — Per-Recipe Detail

> Detailed recipe playbooks loaded on demand. The R-numbered family is summarised in `SKILL.md` (Edge Table Investigation Recipes section + Recipe input + capability matrix). Read this file when you need the full per-recipe detail: tool sequences, interpretation rules, false-positive guards, and uniqueness rationale.

Each recipe lists: **Pattern** (what triggers it), **Tool sequence** (which primitives to call in order), **Interpretation** (how to read the result), and where applicable **False positive guard** + **Why unique vs other recipes**.

For the recipe summary table and the input/capability matrix see `SKILL.md`.

---

## R1 — Mirror Transaction Detection

**Pattern:** Entity A sends to B, and B sends back to A the same (or similar) amount within the same day. Classic circular flow indicator.

**Tool sequence:**
1. `discover_chains(primary_key, pattern_id, direction="both", time_window_hours=24, min_hops=2)` — find short loops
2. Filter chains where `keys[0] == keys[-1]` (cyclic) or where the chain returns to a known counterparty
3. `anomalous_edges(from_key, to_key, pattern_id)` — inspect individual transactions between the mirror pair
4. Compare amounts: if `abs(edge_a.amount - edge_b.amount) / max(amounts) < 0.05` → strong mirror signal

**Interpretation:** Mirror ratio > 0.95 with same-day timing is a strong indicator. Check if both edges are individually anomalous (`is_anomaly=true` in event geometry).

---

## R2 — Pass-Through / Rapid Layering

**Pattern:** Entity receives funds and sends within 2 hours. The entity is a conduit, not a destination.

**Tool sequence:**
1. `discover_chains(primary_key, pattern_id, time_window_hours=2, min_hops=2, max_chains=50)` — find rapid chains
2. `entity_flow(primary_key, pattern_id)` — check if `net_flow ≈ 0` (pass-through entities have balanced flow)
3. `anomalous_edges(from_key, to_key, pattern_id)` — inspect individual transactions at the bottleneck hop

**Interpretation:** net_flow near zero + chains with tight time windows = layering. `degree_velocity` showing acceleration confirms recent ramp-up.

---

## R3 — Burst Detection (Structuring / Smurfing)

**Pattern:** Many transactions to the same target within 24 hours, each below a reporting threshold.

**Tool sequence:**
1. `discover_chains(primary_key, pattern_id, time_window_hours=24, max_chains=100)` — find all outgoing activity
2. Group chains by terminal entity — look for repeated targets
3. `anomalous_edges(from_key, target_key, pattern_id, top_n=50)` — get all edges to the repeated target
4. Check if individual amounts are below a threshold but sum exceeds it

**Interpretation:** 5+ transactions to same target in 24h with amounts clustered just below a round reporting threshold is a strong structuring signal.

---

## R4 — Weighted Reciprocity

**Pattern:** Balanced bidirectional flow between two entities — min(out, in) / max(out, in) close to 1.0.

**Tool sequence:**
1. `entity_flow(primary_key, pattern_id)` — get per-counterparty net flow
2. For each counterparty with both outgoing AND incoming: `reciprocity = min(out, in) / max(out, in)`
3. `anomalous_edges(from_key, counterparty_key, pattern_id)` — inspect the transactions

**Interpretation:** Reciprocity > 0.8 between two entities = suspicious round-tripping. Cross-reference with `contagion_score` — if the counterparty is also contagious, the pair is high-priority.

---

## R5 — Financial Profile (Mule Detection)

**Pattern:** Entity's total flow reveals its role: source (high out, low in), sink (high in, low out), or mule (high both, near-zero net).

**Tool sequence:**
1. `entity_flow(primary_key, pattern_id)` — get totals
2. `cross_pattern_profile(primary_key, line_id)` — anomaly status across all patterns
3. Classify: `net_flow > 0.7 * outgoing_total` → source; `net_flow < -0.7 * incoming_total` → sink; else → mule candidate

**Interpretation:** Mule candidates (balanced flow, multiple patterns flagged) warrant `propagate_influence` to map the network they serve.

---

## R6 — Concentration Risk

**Pattern:** A single counterparty dominates an entity's flow — potential control relationship.

**Tool sequence:**
1. `entity_flow(primary_key, pattern_id, top_n=5)` — get top counterparties by abs(net_flow)
2. Compute `concentration = abs(top_1_net_flow) / (outgoing_total + incoming_total)`
3. `contagion_score(primary_key, pattern_id)` — check if the concentrated counterparty is anomalous

**Interpretation:** Concentration > 0.6 means one counterparty controls >60% of flow. If that counterparty is anomalous (contagion), the entity is at high risk.

---

## R7 — Benford's Law (Amount Distribution)

**Pattern:** Natural financial data follows Benford's Law for first digits. Fabricated transactions often don't.

**Tool sequence:**
1. `find_counterparties(primary_key, line_id, from_col, to_col, pattern_id)` — get top counterparties
2. For each counterparty pair: `anomalous_edges(from_key, to_key, pattern_id, top_n=50)` — collect per-transaction amounts
3. Aggregate all `edge.amount` values across counterparties, compute first-digit distribution
4. Compare with expected Benford distribution: 1→30.1%, 2→17.6%, 3→12.5%, etc.

**Note:** `discover_chains` returns only `total_amount` per chain (aggregate sum), not individual transaction amounts. Use `anomalous_edges` to get per-transaction amounts needed for Benford analysis.

**Interpretation:** Chi-squared test against Benford expected frequencies. p-value < 0.05 = amounts are likely not organic. Most effective with 100+ transactions.

---

## R8 — Witness Cohort Discovery (Fraud Cohort Expansion)

**Pattern:** A confirmed launderer X has a ring of accomplices. Some are already in X's counterparty network (visible via `find_counterparties`). Others share X's anomaly signature — same witness dimensions, drifting in the same geometric direction — but are NOT yet connected to X via transactions. `find_witness_cohort` surfaces these geometric peers, ranked by composite witness/delta/trajectory/anomaly score, with already-connected entities filtered out.

**Honest scope:** This is **investigative cohort expansion**, NOT edge forecasting. The function does NOT predict that X and the cohort members will transact in the future. It surfaces existing peers worth investigating, not future connections.

**Tool sequence:**
1. Identify a confirmed or suspected launderer `X` (from typology recipes R1–R7 or external intel)
2. `find_witness_cohort(X, anchor_pattern_id, top_n=10)` — top peers excluding existing counterparties
3. Inspect `members[]` — focus on entries with `witness_overlap >= 0.5` AND `is_anomaly == true`
4. For each candidate `Y`: `cross_pattern_profile(Y, line_id)` to verify multi-pattern confirmation
5. For high-confidence candidates: `find_witness_cohort(Y, anchor_pattern_id)` — recursive expansion to map the cohort

**Interpretation:** A cohort member with `witness_overlap = 1.0` and `trajectory_alignment > 0.95` is strong — shares the SAME structural anomaly signature and the same geometric drift direction as X. The lack of an existing edge is the agent-guidance value: existing counterparties are skipped (often legitimate), so the cohort is denser in unknown peers worth investigating.

**False positive guard:** Two competitors or two unrelated entities can also share witness profiles. Use cohort members as INVESTIGATIVE RANKING, not as evidence. Combine with domain context before escalation. Many cohort members will not be laundering even when the seed is — the function narrows the search space, it does not eliminate the need for human verification.

**Why this is unique vs other tools:**
- `find_similar_entities + is_anomaly = true`: returns shape twins via plain ANN. `find_similar_entities` does not exclude existing counterparties (often legitimate), does not score witness overlap, and does not weight trajectory alignment
- Neo4j GDS link prediction: topological features only (Adamic-Adar, common neighbors), no witness sets, no population-relative geometry
- ML link prediction (Node2Vec, GraphSAGE): requires training, no interpretability, no labeled data needed for hypertopos
- Vector DB ANN: nearest neighbors but no graph awareness, no edge exclusion

---

## R9 — Chain-Coherent Cascade (Composition-based Layering)

**Pattern:** A stored chain (anchor pattern built from `chain_lines:` BFS extraction OR ingested from an external system per the [external-chains convention](../../../hypertopos-py/docs/external-chains-as-anchor-line.md)) hops through ≥ N consecutive entity-anchor positions that are individually anomalous in the entity-anchor pattern AND share the same dominant delta dimension. The chain-level shape (hop count, time span, amount decay) may look unremarkable, but the *composition* of the path — every consecutive entity flagged for the same structural reason — is the signal.

**External chain source.** When chains come from upstream (SAR typology engine, ERP workflow audit, EHR pathway, customer-journey platform), declare the chain table as an anchor line and populate the `chain_keys` column per the convention (comma-joined member primary_keys in chain order). The full R9 loop — `find_chains_with_coherent_anomaly`, `anomaly_propagation_in_chain`, `classify_chain_typology`, `extend_chain`, `chain_investigation_summary`, `investigate_chain` — works identically on those chains. No code change, just the schema convention. See [external-chains-as-anchor-line.md](../../../hypertopos-py/docs/external-chains-as-anchor-line.md) for the worked example.

**Tool sequence (full investigative loop — triage → flag → trace → label → extend, with one-shot orchestrator shortcut):**

0. **Triage (optional, recommended on unfamiliar spheres)** — `chain_investigation_summary(chain_pattern_id="<chain_pattern>", anchor_pattern_id="<entity_anchor>")` returns one-shot population aggregates: `coherent_run_rate`, `cross_pattern_overlap.jaccard` with chain-shape anomalies, `top_dims_in_coherent_runs`, `recommended_min_hops`, `run_length_distribution`. Triage rules: `coherent_run_rate < 0.005` AND low jaccard → skip the deep R9 loop, fall back to `find_anomalies(<chain_pattern>)`; `coherent_run_rate > 0.05` → expect a productive loop, proceed; `recommended_min_hops > 3` → use the recommended threshold in step 1 to focus on the strongest cases. Cost is one coherent-anomaly sweep — exactly what step 1 would pay anyway, with the aggregates surfaced for free.

**Per-chain shortcut (after step 1 surfaces a target chain_id):** `investigate_chain(chain_id, "<chain_pattern>", anchor_pattern_id="<entity_anchor>")` runs steps 2-5 (trace + typology + shape-anomaly lookup + extension forward + extension backward) server-side in a single call and returns the aggregated report with a `summary` block. Strength scoring uses **four chain-composition signals** (coherent run length >= 3, typology position not "no-run", forward extension has an anomalous candidate, backward extension has an anomalous candidate) — chain-shape anomaly is reported as evidence but NOT scored, so the R9 sweet spot (composition anomalous, chain shape normal) reaches `strong` without needing chain-shape agreement. Buckets: `score >= 3` → `strong` → `escalate to SAR`; `score == 2` → `moderate` → `continue investigation`; `score 0-1` → `weak` → `false-positive candidate`. Rationale = concatenated SAR-ready paragraph. Use when the investigator already knows which chain to drill into and wants the full R9 read in one round-trip; the per-step granular tools (steps 2-5 below) remain available when finer per-step control is needed.

**SAR narrative draft (final step in the per-chain shortcut path):** `generate_sar_rationale(chain_id, "<chain_pattern>", anchor_pattern_id="<entity_anchor>", evidence=<investigate_chain output>)` composes a 3-5 paragraph SAR-ready narrative from the R9 evidence — chain identification + typology, per-hop trace, boundary extensions, chain-shape corroboration, aggregated strength + recommended action. No LLM call; pure template composition. Returns `sar_narrative` (paragraph-separated string), `evidence_anchors` (structured pointers per claim — investigator can audit every line against source data), `confidence` (`high` / `moderate` / `low` derived from strength + evidence completeness), and a `regulatory_template_hint` passthrough (`"FinCEN SAR"` default; tag with `"EU AMLR Annex II"` or internal template name for downstream filing systems). Pass the `investigate_chain` return verbatim as `evidence` to skip re-running the R9 loop. Honesty discipline: narrative uses "evidence indicates" / "the per-hop trace shows" / "corroborating evidence" — never "confirms" — and is positioned as a starting draft for the investigator, NOT a final verdict.

1. **Flag** — `find_chains_with_coherent_anomaly(pattern_id="<chain_pattern>", anchor_pattern_id="<entity_anchor>", min_hops=3, max_results=100)` sweeps all chains in the chain pattern, returns ranked runs (chain_id, run_start_idx, run_length, top_dim, run_keys, max_delta_norm).
2. **Trace** — for each top flagged chain: `anomaly_propagation_in_chain(chain_id, "<chain_pattern>", anchor_pattern_id="<entity_anchor>")` returns the full per-hop progression — see WHERE the anomaly intensity peaks and WHERE it breaks. Hops carry `is_anomaly`, `delta_norm`, `top_dim`, `delta_rank_pct`. Run end with low `delta_rank_pct` = clean exit; with elevated rank = soft boundary worth extending.
3. **Label** — `classify_chain_typology(chain_id, ...)` wraps the trace and returns a five-axis operational tag: `shape` (rising / falling / peak-position), `position_in_chain` (leading / transit / terminal / full-chain), `extension_signals` (forward / backward booleans), plus `dominant_top_dim` across the whole chain. Lets investigator triage chains by typology without re-reading hop sequences.
4. **Cross-check** — compare with `find_anomalies(<chain_pattern>)`: chains in BOTH sets are highest-confidence (chain shape AND chain composition agree); chains in coherent-anomaly-only are missed by chain-shape scoring; chains in baseline-only are flagged for shape (e.g. amount decay) but not composition. The two are orthogonal detectors.
5. **Extend** — when `extension_signals.forward` or `.backward` is True (or the trace's breakpoint hop is in `delta_rank_pct >= 80` band): `extend_chain(chain_id, ..., direction="forward"|"backward")` returns ranked candidate entities that follow / precede the boundary in OTHER chains. Anomalous candidates with high `delta_norm` are prime targets for widening the investigation into the surrounding ring.
6. **Deep-dive per candidate** — for each high-rank extension target: `find_chains_for_entity(candidate_key, <chain_pattern>)` to enumerate which chains contain it, then standard Phase 2 (Entity 360) on those chains' anchor entities.

**Interpretation:** A run of length 4 with `top_dim="find_motif_structuring_max"` is textbook structuring — every account on the path is individually flagged for high motif-structuring activity. A run on `top_dim="pair_edge_count_count_above_threshold"` indicates burst-pair behaviour clustering. Mixed top_dim across the population (3-5 distinct drivers in the top 100 results) signals diverse layering vectors, not a single typology.

**Complementary, not replacement:** This primitive and `find_anomalies(<chain_pattern>)` are orthogonal detectors — they catch different laundering chains. Use both for full coverage.

**False positive guard:** Hubs that appear as terminal nodes of many chains will inflate the result count (multiple chain_ids pointing to the same run pattern). The `run_keys` field reveals duplication — when multiple matched chains share the same `run_keys` tail, treat as one cluster. Always verify with `find_counterparties` and `cross_pattern_profile` on the run entities before escalation.

**Why this is unique vs other tools:**
- `find_anomalies(<chain_pattern>)`: scores chain SHAPE features (hop_count, amount_decay, time_span) — the chain looks unusual as a whole. Different axis from chain composition.
- `trace_root_cause(entity_key, anchor_pattern_id)`: branching DAG from one root via counterparties. Different structure: tree from a single source, not a stored linear chain.
- `decompose_drift(entity_key, anchor_pattern_id)`: temporal slice diff for one entity. Different axis: time, single entity.
- `find_motif_by_hops` with `require_anomalous_entity` per hop: runtime motif enumeration with anchor-anomaly check, requires explicit motif declaration. Operates on the edge table, not on persisted chain anchor patterns.

**Agent-friendly query forms via `detect_pattern`.** Each step of the R9 loop has a natural-language form that the smart-mode router recognises and routes to the right primitive — agents don't need to remember primitive names, just the investigation intent. Chain anchor pattern + entity anchor pattern are auto-detected from sphere context; chain-id tokens of the form `CHAIN-<digits>` are extracted from the query and threaded through.

| step | natural-language query | routes to |
|---|---|---|
| Flag (population sweep) | `find chains where consecutive accounts are individually anomalous` | `find_chains_with_coherent_anomaly` |
| Trace (per-chain hop-by-hop) | `trace chain CHAIN-XXXXXX hop by hop` | `anomaly_propagation_in_chain` |
| Label (typology) | `classify chain CHAIN-XXXXXX typology` | `classify_chain_typology` |
| Extend (boundary candidates) | `extend chain CHAIN-XXXXXX forward` | `extend_chain` |

The Flag query phrasing matches the suggestion `open_sphere` surfaces in `suggested_queries` when the sphere has both a chain anchor pattern and a non-chain entity anchor pattern, so agents discover the loop entry point without drilling into `sphere_overview`. Equivalent phrasings like "find chains where consecutive accounts cascade through structuring" or "anomaly cascade in chains" hit the same routing. After flagging, follow the loop sequentially: Flag → Trace → Label → cross-check (`find_anomalies(<chain_pattern>)` for chain SHAPE) → Extend. Each step's natural-language form is independent.

**Step 6 deep-dive — direct call only.** `find_chains_for_entity(<account_key>, <chain_pattern>)` is **not** routable via natural language in the current implementation: the smart-mode router doesn't extract free-form entity keys from queries (chain-id tokens have a fixed `CHAIN-<digits>` shape; entity keys vary per sphere). After Extend surfaces candidates, the agent must call `find_chains_for_entity` directly with each candidate's primary_key to enumerate the chains they participate in. The R9 narrative still ends with this deep-dive step; only the natural-language entry point is missing.

When the smart-mode router doesn't have an LLM available it falls back to keyword matching on the same intent set. The keyword fallback skips chain-id-required steps when no `CHAIN-<digits>` token is present in the query, so a query like "classify chain shape" without a specific chain_id won't crash — just no per-chain step lands in the plan.

---

## R10 — Trade-Based Money Laundering (TBM)

**Pattern:** Multi-currency cross-bank flows with amount mismatches typical for over/under-invoicing. Funds traverse motifs whose hops cross both currency and bank boundaries, and the per-hop amounts diverge in a way that does not match plain FX rate movement (i.e. the "invoice" leg and the "payment" leg disagree by more than expected currency-conversion noise).

**Tool sequence:**
1. `find_high_potential_motifs(pattern_id="<event_pattern>", motif_type="fan_out", seeds=<suspects>, top_n=20, time_window_hours=24)` — rank short fan-outs by event-aware potential. Seeds typically come from `find_anomalies` on `account_pattern` filtered to `is_anomaly=true`. (`split_recombine` and `bipartite_burst` are alternative motif types worth combining when surface is sparse.)
2. `anomalous_edges(from_key, to_key, pattern_id="<event_pattern>")` along each surfaced hop — read the edge dimensions to get currency, bank, amount per leg
3. Filter: keep paths where `currency_diversity >= 2` (multi-currency along the path) AND `cross_bank_count >= 2` (multi-bank along the path)
4. Score each surviving path: `currency_diversity × cross_bank_count × max(delta_norm along the path)` — entities on high-scoring paths are the candidates

**Interpretation:** TBM signature is structural: the *combination* of multi-currency + multi-bank + amount distortion is what flags it. A fan-out that crosses one currency boundary or one bank boundary is too noisy a signal alone. Once flagged, drill in with `cross_pattern_profile` on the seed and `find_counterparties` on the seed's bottleneck hop to verify the trade-finance context (importer/exporter pair vs unrelated parties).

**False positive guard:** legitimate trade finance also produces multi-currency cross-border flows. The discriminator is the `is_anomaly=true` requirement on at least one path entity AND a non-trivial `delta_norm` on the dominant hop. Without that, you're scoring legitimate import/export businesses.

**Why unique vs other recipes:** R1 (mirror) and R2 (rapid layering) operate on single-currency same-day patterns and explicitly do not require currency or bank diversity. R3 (structuring) is amount-distribution-based, not flow-topology-based. R10 specifically requires currency_diversity ≥ 2 AND cross_bank_count ≥ 2 along the surfaced path, which is the structural signature of TBM as opposed to other layering typologies.

**Status:** Exploratory — empirical lift TBD. Recipe captures architectural completeness (composes existing primitives into a TBM-shaped query); investigators should treat output as candidate set, not verdict, until the validation report lifts the exploratory tag.

---

## R11 — Mule Network Discovery

**Pattern:** A *group* of low-volume accounts forming a dense subgraph with coordinated reciprocal flows. Distinguished from R5 (single-mule profiling) by group cohesion: the network is detected as a cluster, not as individual flagged entities.

**Tool sequence:**
1. `find_clusters(pattern_id="account_pattern", n_clusters=0, top_n=10, sample_size=5000)` — silhouette-based auto-`k` clustering (`n_clusters=0` searches `k=2..15`, picks highest-mean-silhouette via internal subsample). On populations larger than ~100K explicitly subsample to 5000 first to keep cold-call latency bounded. Mule networks present as low-volume clusters distinct from the bulk population
2. For each candidate cluster: aggregate `contagion_score(member_key, anchor_pattern_id="account_pattern")` over members — high mean indicates group-level contagion (anomaly propagates symmetrically across the cluster, not radiating from one seed)
3. For top candidates: `cross_pattern_profile(member_key, line_id)` per member — verify multi-pattern flagging (`account_pattern` + `account_pairs_pattern` etc. both flag the same set)

**Score:** `cluster_size × mean_contagion_score × multi_pattern_hit_rate` (where `multi_pattern_hit_rate` is the fraction of cluster members flagged on ≥ 2 patterns).

**Interpretation:** A mule network looks like a coordinated tight cluster of accounts whose flow signatures resemble each other, whose contagion scores are similar (no clear "ringleader" signature), and whose membership shows up across multiple anchor patterns. Single-pattern clusters or clusters with one dominant contagion-source are NOT mule networks — those are R5 (mule) or R8 (witness cohort) territory.

**False positive guard:** small-business clusters (e.g. local service providers, franchised branches) share density signature without being laundering. Require ≥ 2 cluster members flagged on ≥ 2 patterns AND `mean_contagion_score` above the population median before escalating. Below that, treat as exploratory cohort, not a network.

**Why unique vs other recipes:** R5 classifies one entity at a time (source / sink / mule); R8 expands a cohort *from a confirmed seed* via witness overlap. R11 discovers the network *without* a seed, via cluster density on `account_pattern` geometry — the entry point is the cluster itself, not a known suspect.

**Status:** Exploratory — empirical lift TBD. Same caveat as R10.
