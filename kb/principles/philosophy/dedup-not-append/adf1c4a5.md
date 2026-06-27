---
kind: pragmatic
type: policy
domain: [global]
confidence: 1
sources: 0
entities: [designer]
refs: []
---
# Learning the same thing twice subsumes; it never duplicates

A claim is asserted once. When an agent learns something the corpus already knows, the operation merges into the existing fact — update, subsume, or raise confidence — rather than appending a near-duplicate. The corpus converges on a single best statement of each fact; it does not accrete redundant copies.

Why: a knowledge base whose size grows with every re-encounter of the same truth is a log, not knowledge. Deduplication is what separates "what is known" from "everything that was ever said." The prune pass in knomit_review enforces this after the fact, but the intent is upstream — learn should recognize a known claim and refine it, not restate it.

What NOT to do:
- Write a new fact for a claim a query would already surface — update the existing one instead
- Treat confidence/source accumulation as a reason to keep duplicates around
- Let near-identical facts coexist across categories merely because their paths differ
