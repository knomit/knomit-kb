---
kind: pragmatic
type: policy
domain: [global]
confidence: 1
sources: 0
entities: [designer]
refs: [kb/principles/philosophy/dedup-not-append/adf1c4a5.md, kb/principles/philosophy/self-describing-ontology/3668e8be.md]
---
# Knowledge has structure the author never declared

Beyond what each fact says about itself, the corpus has emergent structure. Facts cluster two ways: by meaning — embedding similarity groups facts that talk about the same thing regardless of where they were filed — and by classification — shared ontology paths, domains, and entities group facts by how they were declared. Surfacing these clusters reveals what the knowledge base is really about, where it is dense, and where it is thin.

Why: a self-aware KB must see its own shape, not just answer point queries. Semantic clusters catch facts that belong together but were filed apart (a signal for dedup and synthesis); classification clusters show declared coverage. Reading them against each other exposes drift — where meaning and filing disagree — and gaps the author never knew existed. This is the substrate for synthesis and review.

What NOT to do:
- Treat clustering as a search-ranking detail rather than a way to understand the corpus
- Rely on classification alone (it only sees what authors declared) or on embeddings alone (they miss intent)
- Let clusters that reveal duplicates or gaps go unacted-on — they are inputs to dedup and synthesis
