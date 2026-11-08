---
type: observation
domain: [web, ui, corpus, refs]
confidence: 0.85
sources: 1
entities: [FactBody, fact.ClassifyRef, buildRefViews, wikilink]
motifs: [one-notation-many-meanings]
refs: ['kb://3ec012f5b4d2/kb/invariants/ui/factbody/ref-scheme-branching/29780d09.md', 'src://7b4887ce51d9/web/src/FactBody.tsx@29e3187c6c5140162ceb8f44958668115e57fd91:7935c0de5417be8f1a12090f6d0c44ba399bd73d', 'src://7b4887ce51d9/internal/domain/fact/ref.go@29e3187c6c5140162ceb8f44958668115e57fd91:15a393c01dab70fb96d8622044bd458c86d87c35#L111-L151', 'src://7b4887ce51d9/internal/domain/fact/markdown.go@29e3187c6c5140162ceb8f44958668115e57fd91:910144ace628d1f7e5f4735657f1c38cd2c501bf', 'kb://3ec012f5b4d2/kb/decisions/fact/body/no-wikilinks/e89cd9f1.md']
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

The server resolves no `[[...]]`: fact.ClassifyRef never sees body text, and no ref_edges or DERIVED_FROM edge comes from a body `[[...]]`. Since #797 (29e3187c) internal/domain/fact/markdown.go does scan bodies for `[[...]]`, but only to REFUSE them on write (fact.WikiLinks, CheckWikiLinks, CheckWikiLinksUpdate), never to link anything. FactBody renders them as literal brackets, and nothing in web/src handles `[[` (FactBody.tsx is byte-identical at 29e3187c to the #793 merge b7a34667 where this was first checked).

**DECIDED (user, 2026-11-08): never rendered, and writing them is now refused.** The user decided against rendering `[[...]]` because of prompt-injection risk; refs are the only link channel. PR #797 (merged as 29e3187c) refuses `[[...]]` outside code on every fact write, judged on the RESULTING body: a create may hold none, and an update may leave none, including ones the fact already had. The existing occurrences counted above are deliberately left in place: the user decided (2026-11-08) not to clean them up, and no cleanup job runs. So each of those facts cannot be updated, on any field, until the update removes its `[[...]]` or moves them to refs. Rule, scope and rationale: kb/decisions/fact/body/no-wikilinks/e89cd9f1.md. The three findings that bore on the question: (1) the forms are ambiguous (a slug is a category name, not an id or a title), so any resolution rule would be a product decision, not an implementation detail; (2) per kb/invariants/ui/factbody/ref-scheme-branching/29780d09.md the client does not classify or resolve refs itself, because existence is anchor-dependent and only the server knows it; (3) the non-link uses mean `[[` alone is not a reliable marker.

What this does NOT mean: the counts are not a backlog of broken links to resolve or remove: they stay as they are, and the absence of rendering is not a bug: it is the decision.
