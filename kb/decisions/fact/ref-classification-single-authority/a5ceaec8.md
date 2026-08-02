---
type: observation
domain: [fact, refs, store, mcp, web]
confidence: 0.95
sources: 1
entities: [fact.ClassifyRef, fact.Ref, fact.RefKind, internal/fact/ref.go, internal/store/search_index.go, internal/mcp/explain.go, web/src/FactBody.tsx, internal/federate/federate.go, federate.ParseQualifiedPath]
refs: ['src://knomit/internal/store/search_index.go@4154e92c8ff333435fd00c442489e855e4c3331e', 'src://knomit/internal/mcp/explain.go@4154e92c8ff333435fd00c442489e855e4c3331e', 'src://knomit/web/src/FactBody.tsx@4154e92c8ff333435fd00c442489e855e4c3331e', 'src://knomit/internal/federate/federate.go@4154e92c8ff333435fd00c442489e855e4c3331e', 'src://knomit/web/src/useTimeTravel.ts@4154e92c8ff333435fd00c442489e855e4c3331e', kb/principles/philosophy/historical-not-current/6c745bf4.md, kb/decisions/lens/qualified-path-repo-identity/10a3bcc0.md, kb/invariants/store/refs-to-derived-from-edges/eb438c74.md]
---
# One ref classifier in internal/fact returns KIND only — resolution status is commit-dependent and never server-computed

Ref classification was implemented three times with three different rules: the edge builder (search_index.go, "anything not http(s):// is a local candidate"), knomit_explain (explain.go, "kb:// prefix OR not .md means External"), and the web UI (FactBody.tsx, "any URI scheme means plain text, schemeless means clickable"). For a typo'd local path all three disagreed — dropped, reported healthy, rendered as a live link. Diagnosing 22 citation orphans required opening the edges table in SQLite directly.

**Options considered:** (a) fix each of the three call sites independently; (b) extract one classifier returning kind AND resolution status, consumed everywhere; (c) extract one classifier returning KIND ONLY, with resolution left to existing commit-anchored machinery.

**Rationale:** (a) reproduces the drift. (b) was drafted and is WRONG: it violates kb/principles/philosophy/historical-not-current — a server-computed status is evaluated at HEAD, so a fact opened at an older commit whose ref target was later retracted would be labelled broken even though its ref resolves correctly at the referrer's commit. The split that makes (c) work: KIND (local fact / foreign fact / source / external URL / malformed) is a function of syntax plus repo identity, both HEAD-independent, so the server can state it once and be right forever. RESOLUTION is a function of the referrer's commit, so the server must not state it at all.

**The choice:** `fact.ClassifyRef(raw, localRepoID) -> fact.Ref` in internal/fact — a pure, I/O-free function returning kind and parsed components. All three call sites consume it. internal/fact is a leaf (federate imports repos imports store imports fact), so ClassifyRef owns the kb:// parsing and federate.ParseQualifiedPath delegates DOWN to it. A completeness test modelled on TestFactSchema_DescriptionsAreComplete fails the build if any file outside internal/fact re-derives ref classification from raw scheme checks.

**Non-scope:** this does NOT authorize the fact API or any client to report whether a ref resolves. The UI gates clickability on kind alone and hands the target to the existing commit-anchored hop (resolveHopAnchor / qualifyHopTarget in useTimeTravel.ts), which already pins the version the referrer reasoned over. Adding a `status` field back to the API re-introduces the HEAD-time bug this decision removed.
