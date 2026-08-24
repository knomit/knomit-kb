---
type: observation
domain: [synthesize, discovery, corpora]
confidence: 0.9
sources: 1
entities: [domain tags, subject-disjointness, agentic-engineering]
refs: ['kb://3ec012f5b4d2/kb/meta/research/motif-blueprint/0322276a.md']
---
# Umbrella tags (df > ~20% of corpus) silently zero out any pairwise disjointness gate — exclude them from the gate, not from the data

Some corpora carry umbrella tags on essentially every fact: agentic-engineering has the domain tag 'agentic-engineering' on all facts plus six more domain tags each covering >20% of the corpus, and the tokens 'agents'/'security' in nearly every subject path. Any gate of the form 'members must share no domain tag / no subject token' therefore rejects EVERY pair in such a corpus — and the failure mode is an empty result, indistinguishable from a genuinely empty candidate space, so it ships silently. Measured during the 2026-08 bridging spike: a subject-disjointness gate returned zero candidates for the entire agentic-engineering corpus until tags/segments with df > 20% were excluded from the disjointness check. Consequence: every pairwise disjointness or exclusion gate over authored tags must compute per-corpus umbrella sets (df > ~20%) and exclude them from the CHECK — while keeping them in the data, since they remain meaningful for scoping and filtering.
