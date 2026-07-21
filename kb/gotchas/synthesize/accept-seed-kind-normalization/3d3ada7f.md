---
type: observation
domain: [synthesize, store, seeding]
confidence: 0.9
sources: 0
entities: [Strategy.AcceptSeed, Strategy.SeedQuery, factFromSearchResult, fact.DefaultKind, ParseFact, Pipeline.dirtyFacts]
refs: ['src://knomit/internal/synthesize/pipeline.go@6356eab8', 'src://knomit/internal/synthesize/strategy.go@6356eab8', kb/decisions/architecture/synthesize/scope-filter/72ee2c9a.md, kb/conventions/synthesize/dirty-facts-excludes-pragmatic/8b26a25d.md]
---
# AcceptSeed runs on BOTH scan paths, so a raw empty Kind from Search would silently make the seed pool depend on watermark state

The engine's seed predicate `Strategy.AcceptSeed(fact.Fact)` is applied on BOTH seeding paths: post-Search on the full-scan path, and post-ParseFact on the incremental path. Kind arrives from Search as a RAW STRING off the facts table. If it were left empty, AcceptSeed would reject the entire full-scan pool while accepting the incremental one — meaning the seed pool would depend on whether a watermark exists.

That is the exact divergence class decisions/architecture/synthesize/scope-filter exists to prevent: read and write sides must agree, and the same arguments must yield the same seed pool regardless of watermark state. `factFromSearchResult` therefore normalizes Kind == "" to fact.DefaultKind, matching what ParseFact does on the other path.

CONSEQUENCE that looks like a bug and is not: AcceptSeed DUPLICATES the IncludeKinds/IncludeTypes filter already in SeedQuery. This is intentional. SQL cannot run on the incremental path, so the Go predicate is the AUTHORITATIVE statement of the filter (e.g. conventions/synthesize/dirty-facts-excludes-pragmatic) and the SQL is only an efficiency measure. Do not delete the Go predicate as redundant — that silently changes incremental-run behaviour only, which is the hardest kind of drift to notice.
