---
type: observation
domain: [store, synthesize, discovery]
confidence: 0.95
sources: 0
entities: [TokenDF, fact_domains, fact_entities, branch_facts, BlastRadius]
refs: ['src://knomit/internal/store/token_df.go@9f8780e5', kb/decisions/store/discovery-spec-df-query-time/cfa48539.md, kb/architecture/store/240a44f5.md]
---
# Compute token document-frequency at query time, not as a materialized table

Decision for the bridge-quality `spec` (rarity) signal's document-frequency input.

**Options considered:** (a) a materialized `token_df` table maintained on upsert + rebuild, with rebuild≡ingest parity tests; (b) a query-time indexed COUNT over the existing `fact_domains`/`fact_entities` junctions joined to `branch_facts` for liveness, memoized in the consumer.

**Rationale:** `df` is needed only for the handful of candidate bridge tokens that survive the cheap coh/sep gates (bounded by the effort budget, ≤12 medium / ≤48 high per session), each an indexed point-lookup, memoized — tens of ms against the ~68s clustering the pipeline already does. A materialized table would add a migration, dual-choke-point maintenance (upsert + rebuildFacts), and parity tests to save noise-level time, AND be *less* correct (version-level, not live branch-scoped). All needed indexes (fact_domain_tokens_token, fact_entities_entity, branch_facts_fact) already exist; merge-safe because the junctions counted are maintained in upsert, which runs on merges too.

**The choice:** query-time. Live, branch-scoped COUNT(DISTINCT bf.path); domain matches canonical `fact_domains.domain`, entity matches authored `fact_entities.entity` (NOCASE). Memoize per session in the consumer.

WHERE IT LIVES (corrected 2026-07-22): the method is `TokenDF(ctx, branch, token, kind)` on the **graphStore** sub-service — reach it via `Service.GraphStore().TokenDF(...)`, NOT `SearchIndex.TokenDF`. It moved in the P1.3 read-surface split (internal/store/token_df.go; see kb/architecture/store/240a44f5.md). Calling it through Service.Search() still compiles via the composite facade, but depend on the narrowest accessor.

NOTE — DUPLICATE: kb/decisions/store/discovery-spec-df-query-time/cfa48539.md records this SAME decision with a third option considered and more rationale. Read that one first; this fact is the shorter restatement. They agree — if they ever disagree, the richer one is the survivor.
