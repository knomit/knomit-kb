---
type: observation
domain: [materialize, frames, refs, mcp]
confidence: 0.95
sources: 1
entities: [knomit_materialize, internal/corpus/materialize, cross_kb, unpin, frame, Manager.ByRootID]
refs: ['kb://3ec012f5b4d2/kb/decisions/fact-types/frame/type-only-scope/85d04f3b.md', 'kb://3ec012f5b4d2/kb/gotchas/corpus/refs/superseded-flag/a4f230f9.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/identity/uid-vs-root-commit-id/ae7a7d41.md', 'kb://3ec012f5b4d2/kb/principles/philosophy/historical-not-current/6c745bf4.md']
---
# materialize rulings (2026-10-28): pinned by default, unpin silently drops gone targets, cross-KB follows ANY registered repo at its latest version, frame ancestors always included with their own data only, depth 4/200 facts, MCP+REST only

The user's final rulings for `materialize` (knomit_materialize over MCP, `/materialize` over REST; package internal/corpus/materialize). They override rev 2 of the design.

1. CROSS-KB: with the optional `cross_kb` parameter set, a `kb://<id12>/…` ref is followed when the target repo is ANY repo registered on the server (not only a mount of the caller's binding). The id is the root-commit id (not the registry uid). The foreign target is read at that repo's LATEST version (its branch tip, frozen at the first call) and marked as latest-version in the output, because a foreign ref has no DERIVED_FROM edge and so no cited version. When `cross_kb` is off, or no registered repo has the id, the ref is a "not included" entry WITH a reason.
2. FRAME ANCESTORS: always included whatever their distance, bounded only by the fact cap. Each ancestor is rendered with its own fact data ONLY: none of its other references is expanded. Example: frame A has frame children B and C; materializing B gives "A | B" and never C. Order: topological (each ancestor before any frame that cites it, a shared ancestor once, ties by refs order), then the starting frame, then the ordinary depth-first refs walk from it. A parent is ANY ref of a frame whose (resolved) target is a frame (see the frame-parent ruling below); several parents are allowed; nothing validates them.
3. UNPIN (off by default): every ref resolves to the target's latest version. A gone target (retracted/deleted) is NOT rendered and NOT mentioned anywhere — no "not included" line, no count — and anything reachable only through it drops out too. Frame ancestry uses latest versions too. Resolved versions are frozen in the cursor snapshot.
4. Defaults depth 4 (max 10), 200 facts (max 500).
5. No CLI command.
6. No preamble or skill: the human formatting instructions are a server constant on page 1 with a version on every page.
7. materialize ignores `context.status` entirely; a retracted frame is the abandoned one.
8. "Changed since cited" is decided by comparing BODIES, never commit inequality.

CONSEQUENCE: the default output is a pure function of (root, anchor, options) — historical, never HEAD-relative; only `unpin` and cross-KB nodes read latest versions, and the output says so.

WHAT THIS DOES NOT MEAN: cross-KB does NOT pin the foreign target to any cited version (none exists), and `unpin` silence is deliberate — a shorter document under unpin is not a bug. Ancestors are NOT depth-limited, but they are NOT expanded either: an ancestor's refs to sibling frames never appear.

FRAME-PARENT RULING (user, 2026-10-28, round 3 of the PR #675 review; it REPLACES an earlier creation-commit rule, which was built and then removed): frame order uses COMMIT time in the temporal graph, never CREATION time; nothing computes when a frame was created. The frame you are at is the CHILD and every frame it refs is a PARENT. Each ref is read at the commit its DERIVED_FROM edge pins, so a cited version precedes, or shares the commit of, the citing one: walking parents moves backward through the graph on its own.
- Two frames that ref each other (at the same commit, or through a stale pin): the one you are at, i.e. the one reached FIRST from the starting frame, is the child, the other its parent. There is NO cycle detection: the 'each path written once' dedup ends the walk (a frame already in the set is not added again), and the partner's ref back is a back-reference.
- Frames in other repos follow the cross-KB rules of regular facts: resolved only with cross_kb on and the caller's reach; otherwise a 'not included' entry. When followed, the frame is read at that repo's latest version (frozen on the first call) and the ancestor walk continues there under the same rules.
- A retracted-and-relearned frame, or a frame on a concurrent branch: no special handling; the version at the pinned commit is used.
- Under unpin, refs resolve to latest and path dedup still ends the walk.
- Order: each ancestor before any frame that cites it, ties by refs (first-reached) order.

As built (after review B3): ancestors are ordered by a depth-first POST-ORDER from the starting frame over its frame refs in refs order (walk back to the top, write top-down); a ref to a frame already visited is a back-reference, which is how 'the frame you are at is the child' settles a mutual pair and why the walk cannot loop. A breadth-first pass only decides inclusion and the cap. An earlier discovery-number edge filter with a Kahn sort was removed: it wrote a child before a parent reached earlier (U; T->U; C->T; F->T,C must give U, T, C, F).

WHAT THIS DOES NOT MEAN: not 'the older frame is the parent' and not 'the newer timestamp is the child': only the walk from the starting frame (pinned refs plus first-reached order) decides.
