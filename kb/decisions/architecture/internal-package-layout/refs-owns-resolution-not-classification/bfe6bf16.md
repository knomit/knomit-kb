---
type: observation
domain: [architecture, refs, fact, packaging, store]
confidence: 0.95
sources: 1
entities: [internal/refs, refs.Gate, refs.New, refs.FromFactQuery, refs.Canonicalize, fact.ClassifyRef, internal/fact, TestFactStaysPure, internal/refgate]
refs: ['src://7b4887ce51d9/internal/refs/refs.go@8cba29ff2e1c0556c90b1c9b21574772303b28cf:c451fd992c42a2f30f0db62108259c0647b773dc', 'src://7b4887ce51d9/internal/fact/ref.go@8cba29ff2e1c0556c90b1c9b21574772303b28cf:48d1932edb298fc889e24c1aa7bffac5ae1aa5b1', 'src://7b4887ce51d9/test/archtest/layering_test.go@8cba29ff2e1c0556c90b1c9b21574772303b28cf:9a7667d237b127e4fdb7384115a59bb1bd0fbd0e', 'https://github.com/knomit/knomit/pull/57', 'kb://3ec012f5b4d2/kb/decisions/fact/ref-classification-single-authority/a5ceaec8.md']
---
# internal/refgate became internal/refs, and the broader name is bounded: refs owns ref RESOLUTION, internal/fact keeps CLASSIFICATION

`internal/refgate` was renamed to `internal/refs` (`refs.Gate`, `refs.New`, `refs.Canonicalize`), and simultaneously fenced so the broader name does not become a dumping ground.

**Options considered**

1. Keep `refgate` as a top-level package.
2. Rename to `refs` — more generic, reads better at call sites. **Chosen.**
3. Nest under `internal/fact` as `fact/refgate` — the intuitive parent, since it is all about fact references.
4. Nest under `internal/store` as `store/refgate` — dependency-coherent, since it already imports store.

**Rationale**

Option 2 won on call-site readability: `refs.Gate` and `refs.New(...)` read better than `refgate.Gate`.

Option 3 was rejected despite being mechanically legal (fact does not import refs, so `fact/refs → store → fact` is no cycle). It INVERTS dependency weight: `internal/fact` is 154 deps / 28 external, while refs is 388 / 189 — essentially store's entire closure (sqlite, go-git, golang-migrate, ProtonMail crypto). Go directory nesting reads as "a more specific version of the parent", and readers assume dependency weight is monotone as you descend. Eleven packages import `fact` today, several precisely because it is the cheap pure leaf; once `fact/<anything-heavy>` exists, "I'll grab something from fact/" links SQLite and nothing fails — the binary just gets fatter.

Option 4 is dependency-coherent (child ≥ parent) but was rejected because refs is a corpus RULE, not storage mechanics: having mcp, web and synthesize import `store/refgate` reads as three front-ends reaching into store internals, which is the impression the package was extracted to avoid.

**The choice**

`internal/refgate` → `internal/refs`, staying top-level.

**Consequence to act on**

`refs` is a much stronger magnet than `refgate` — it invites "all ref things". The one thing that must NOT migrate into it is ref CLASSIFICATION. `fact.ClassifyRef` answers what KIND a ref is and stays in the pure leaf; `refs` answers whether a ref RESOLVES, which is commit-dependent and needs corpus access plus repo identity. That is why refs may import `internal/store` and `internal/fact` may not. The package doc carries a `# Scope: resolution, NOT classification` section saying this, and `TestFactStaysPure` in `test/archtest` enforces the other direction by failing if `internal/fact` ever transitively imports store, repos, embeddings, refs, web, mcp or synthesize.

**Named misreading**

This does NOT mean "put anything ref-related in internal/refs". A pure ref helper — parsing, shape validation, kind classification — belongs in `internal/fact`, and moving one into `refs` would force every caller that only wanted to parse a string to link the whole store closure. The test catches the fact→heavy direction; nothing catches gratuitous migration INTO refs, so that one is on the author.
