---
type: observation
domain: [okf, store, architecture, git]
confidence: 0.95
sources: 1
entities: [internal/okfsource, okfsource.Load, okfsource.Snapshot, internal/store/okf.go, internal/domain/okf]
refs: ['src://7b4887ce51d9/internal/domain/okf/source/source.go@48d32e95ccc2bb2b439f975479ae15d165e5049e:18f73c167fc6c98ba9d45e0f47124a243d5592d5', 'src://7b4887ce51d9/internal/domain/okf/source/history.go@48d32e95ccc2bb2b439f975479ae15d165e5049e:4825ee15f80ff79d53bb90fcd3a73c2e319c709f']
---
# internal/okfsource reads a KB through a go-git storer ONLY — no SQL — so the exporter runs against a plain git clone

internal/okfsource turns a knowledge base at one commit into everything the OKF mapper needs: facts, changelog events, per-fact revisions, retirements, and the ontology, via `Load(storer, head) Snapshot`.

It reads git objects and NOTHING else — no SQLite, no knomit Service, no repo instance. That is what makes the export portable: the same code path serves knomit's SQLite-backed store, a filesystem `git clone`, and an in-memory test fixture. The CLI depends on it, so the exporter needs no knomit server at all.

This package is a lift of the read half of the former internal/store/okf.go, which was verified to touch no SQL and was therefore already storer-agnostic in substance. The logic was moved deliberately unchanged; only the plumbing was re-pointed (methods on *Service became functions taking a storer, and the logging call became a Warnings slice on Snapshot, because the package must not log — a CLI and a server want to surface degradation differently).

The consequence to preserve: any new read the exporter needs must be expressible against a storer. Reaching back into internal/engine/store for it would silently re-couple the tool to a running server and break the plain-clone case.

Note the module constraint that forces the layout: Go forbids importing knomit/internal/... from outside the module, so the CLI must live in-tree at tools/okf rather than in its own repository.
