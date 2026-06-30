---
type: observation
domain: [store, synthesize, discovery]
confidence: 0.95
sources: 0
entities: [TokenDF, fact_domains, fact_entities, branch_facts, BlastRadius]
refs: ['src://knomit/internal/store/token_df.go@9af4f90']
---
# Compute token document-frequency at query time, not as a materialized table

Decision for the bridge-quality `spec` (rarity) signal's document-frequency input.

**Options considered:** (a) a materialized `token_df` table maintained on upsert + rebuild, with rebuild≡ingest parity tests; (b) a query-time indexed COUNT over the existing `fact_domains`/`fact_entities` junctions joined to `branch_facts` for liveness, memoized in the consumer.

**Rationale:** `df` is needed only for the handful of candidate bridge tokens that survive the cheap coh/sep gates (bounded by the effort budget, ≤12 medium / ≤48 high per session), each an indexed point-lookup, memoized — tens of ms against the ~68s clustering the pipeline already does. A materialized table would add a migration, dual-choke-point maintenance (upsert + rebuildFacts), and parity tests to save noise-level time, AND be *less* correct (version-level, not live branch-scoped). All needed indexes (fact_domain_tokens_token, fact_entities_entity, branch_facts_fact) already exist; merge-safe because the junctions counted are maintained in upsert, which runs on merges too.

**The choice:** query-time. Shipped as `SearchIndex.TokenDF(ctx, branch, token, kind)` — live, branch-scoped COUNT(DISTINCT bf.path); domain matches canonical `fact_domains.domain`, entity matches authored `fact_entities.entity` (NOCASE). Memoize per session in the consumer.
