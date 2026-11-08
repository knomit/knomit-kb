---
type: observation
domain: [web, ui, corpus, refs]
confidence: 0.85
sources: 1
entities: [FactBody, fact.ClassifyRef, buildRefViews, wikilink]
motifs: [one-notation-many-meanings]
refs: ['kb://3ec012f5b4d2/kb/invariants/ui/factbody/ref-scheme-branching/29780d09.md', 'src://7b4887ce51d9/web/src/FactBody.tsx@031640eef8406ffcf1672e38ebbca8db86c9cbc9:7935c0de5417be8f1a12090f6d0c44ba399bd73d']
---
# `[[...]]` in fact bodies is not one syntax: authors write an 8-hex id, a kb/ path with or without .md, a kb/ category, or a bare category-leaf slug, and some `[[...]]` are not links at all; the server parses none of them

Survey of the CURRENT version of every live fact in the local corpora (2026-11-08, read-only over the repo DBs): 465 `[[...]]` occurrences outside fenced code in 241 facts, almost all in two repos (knomit-kb 52, one AI-agents corpus 412).

Forms, by count:
- 8-hex id `[[f6db3a49]]`: 391. 387 resolve uniquely to a live fact whose file name is that id (no id collisions in any corpus); 4 resolve to nothing.
- bare slug `[[cte-over-commit-parents]]`: 52. It names the LAST CATEGORY SEGMENT of the intended fact's path (kb/decisions/store/resolver/cte-over-commit-parents/189c424e.md). 40 match exactly one live fact that way, 1 matches two (local-origin-gate: a gotchas fact and an invariants fact), 11 match none (stale or never-written slugs such as two-table-commit-index, okf-type-from-topic).
- kb/<path>/<id8> without .md: 10, all resolve.
- kb/<path>.md: 8, all resolve.
- kb/<category> with no id `[[kb/decisions/desktop/icons/per-platform-strategy]]`: 4. Each names a category holding one fact today, but a category can hold several.
- NOT links: `[[oauth.client]]` (a TOML array-of-tables header, 3 times), `[[:space:]]` (a POSIX regex class). A naive renderer would turn these into broken links.

The server has no wikilink handling: grep of internal/ finds no `[[` parsing, fact.ClassifyRef never sees body text, and no ref_edges or DERIVED_FROM edge comes from a body `[[...]]`. Today FactBody renders them as literal brackets, and nothing in web/src handles `[[`.

Consequences for anyone building wikilink rendering: (1) the forms are ambiguous (a slug is a category name, not an id or a title), so the resolution rule is a product decision, not an implementation detail; (2) per kb/invariants/ui/factbody/ref-scheme-branching/29780d09.md the client must not classify or resolve refs itself: existence is anchor-dependent and only the server knows it, so resolution belongs server-side (e.g. alongside buildRefViews); (3) the non-link uses mean `[[` alone is not a reliable marker.
