---
type: observation
domain: [fact, taxonomy]
confidence: 0.95
sources: 0
entities: [Kind, Type, EpistemicTypes, PragmaticTypes, AllEpistemicTypes, AllPragmaticTypes, DefaultKind, DefaultEpistemicType, factTypeDocs, typeStyles, TypeIcon, spec/mbekg.md, fact.Frame]
motifs: [undetected-derivation-gap]
refs: ['kb://3ec012f5b4d2/kb/decisions/fact/pragmatic-types/signal/e48a7a32.md', 'kb://3ec012f5b4d2/kb/decisions/fact/epistemic-types/insight/f9640cdc.md', 'kb://3ec012f5b4d2/kb/decisions/fact-types/frame/type-only-scope/85d04f3b.md', 'src://7b4887ce51d9/internal/domain/fact/kind.go@1eb481b61f036c61969858179862732a20149d99:e9fedce857eaa7b804658b10aef47665436e4d5f', 'src://7b4887ce51d9/internal/domain/fact/epistemic_type.go@1eb481b61f036c61969858179862732a20149d99:967a848d86a6d4968d8a21b55d182ed940d3c6ec', 'src://7b4887ce51d9/internal/domain/fact/pragmatic_type.go@1eb481b61f036c61969858179862732a20149d99:ab12895627834da1ffa7eda4e0bcc400c6d7031c', 'src://7b4887ce51d9/internal/edge/mcp/factschema.go@1eb481b61f036c61969858179862732a20149d99:5185d013ac06a86db44d542ea512cde85458c747', 'src://7b4887ce51d9/internal/edge/mcp/factschema_test.go@1eb481b61f036c61969858179862732a20149d99:31e5829edf90c952fea0edf67bf057db59550fef', 'src://7b4887ce51d9/internal/domain/fact/pragmatic_test.go@1eb481b61f036c61969858179862732a20149d99:cc56ceb9b709eeb98f1143a04891974139594f73', 'src://7b4887ce51d9/internal/engine/store/index/index.go@1eb481b61f036c61969858179862732a20149d99:dddc6718d96140c93676dd03a463a462cad687b6', 'src://7b4887ce51d9/internal/edge/mcp/instructions.go@1eb481b61f036c61969858179862732a20149d99:38e8967960305b7666fec02f1dc37c7824513d4e', 'src://7b4887ce51d9/web/src/utils.ts@1eb481b61f036c61969858179862732a20149d99:2f0f291eb7a0a02f0bcf99c21fc98a1889d942e1', 'src://7b4887ce51d9/web/src/icons.tsx@1eb481b61f036c61969858179862732a20149d99:a4b29d235e605bb58c30c746565bb4192fb9f760', 'src://7b4887ce51d9/web/src/icons.test.tsx@1eb481b61f036c61969858179862732a20149d99:0811d36a2e179216f5fdfad05a8f1e73673725fd', 'src://7b4887ce51d9/web/src/utils.test.ts@1eb481b61f036c61969858179862732a20149d99:05298a9ff415a20e392cdecd3e860f7e660d3844', 'src://7b4887ce51d9/spec/mbekg.md@1eb481b61f036c61969858179862732a20149d99:d6245bda7ae6b304246721abd3f2510fa75a382f', 'https://github.com/knomit/knomit/pull/247', 'https://github.com/knomit/knomit/pull/668']
---
# Kind has 2 members (Epistemic, Pragmatic); Type has 14 leaves; authoritative sets live in maps

Kind (kind.go) is a string-typed enum with two members: Epistemic ('epistemic') and Pragmatic ('pragmatic'). Type (epistemic_type.go, pragmatic_type.go) is also string-typed and carries 14 leaf values: 10 epistemic (Observation, Concept, Process, Principle, Pattern, Reference, Synthesis, Insight, Hypothesis, Methodology) + 4 pragmatic (Policy, Heuristic, Signal, Frame).

Insight was added in PR #70 (ordered between Synthesis and Hypothesis) — see [[insight]]. Signal was added by F01 in PR #247, ordered last among the pragmatic types — see [[signal]]; it types coordination content that is consumed rather than believed. Frame was added by PR #668, ordered last among the pragmatic types: a node of a mission's thinking process, worked with rather than believed; the type adds only the pragmatic-kind admission and the same-subject candidate skip, see kb/decisions/fact-types/frame/type-only-scope/85d04f3b.md.

Authoritative sets: EpistemicTypes map[Type]bool and PragmaticTypes map[Type]bool — the SINGLE source of truth used by Kind.AllowsType(t). Stable ordering via AllEpistemicTypes() / AllPragmaticTypes() for code that needs deterministic iteration. DefaultKind = Epistemic; DefaultEpistemicType = Observation — pragmatic facts have NO default type (policy ≠ heuristic ≠ signal ≠ frame; the author must declare which). Type itself is the umbrella — the previously-separate EpistemicType was collapsed into Type to remove call-site casts.

## Adding a new Type: the full checklist

An earlier revision of this fact said that after the Go helpers "nothing else needs editing", because the consumers it knew about all derive their lists. That was wrong, and F01 proved it: two hand-maintained tables in the web UI have no derivation, so a new type reached the Go enum, the MCP schema and the spec while the UI still could not draw it.

DERIVED — no edit needed, verified at the F01 merge commit and re-read at the #668 merge commit (1eb481b6), where the frame type reached all three without touching them:
- internal/edge/mcp/factschema.go builds the `type` enum from allTypes() = AllEpistemicTypes()+AllPragmaticTypes() and the description's gloss lists from the same helpers; only the per-type factTypeDocs entry (item 2) is hand-written.
- internal/edge/mcp/instructions.go renders both type lists via instructionTypeLines(fact.All*Types(), ...).
- internal/engine/store/index/index.go builds the `type` completion list from AllEpistemicTypes()+AllPragmaticTypes().

MUST BE EDITED BY HAND:
1. The EpistemicTypes or PragmaticTypes map AND the matching AllXxxTypes() helper (internal/domain/fact).
2. internal/edge/mcp/factschema.go — a factTypeDocs entry (gloss + aside). Keep the gloss short: it is what lands in the one-line JSON schema description.
3. web/src/utils.ts — a `typeStyles` entry (color, bg, label, icon). The new colour must sit ≥ 60 weighted-RGB from EVERY existing type colour, and 4- or 8-digit hex is rejected.
4. web/src/icons.tsx — an SVG component plus a `case` in the TypeIcon dispatcher, or the type silently renders as UnknownIcon.
5. spec/mbekg.md — the epistemic or pragmatic type table (§2.4).
6. web/src/utils.test.ts — the hand-written type list in the matching per-kind entry test (see below).

## Which of those a test will actually catch

This distinction matters more than the checklist, because a hand-list that looks like a guard is worse than no guard.

- CATCHES A MISSING TYPE ON ITS OWN: TestAll{Epistemic,Pragmatic}Types_MatchesSet, which compares map against slice (a disagreement ships a tool-schema enum rejecting a type the server accepts); and TestFactSchema_DescriptionsAreComplete, which fails until item 2 exists. Between them, items 1 and 2 are enforced.
- DOES NOT: the per-kind entry tests in web/src/utils.test.ts ('has entries for all 10 epistemic types', 'has entries for all 4 pragmatic types'). Both iterate a HARD-CODED `expectedTypes` array, so a fifth pragmatic type is simply not looked at until someone extends the list (#668 extended it to policy, heuristic, signal, frame by hand). That is item 6, and it is maintenance, not protection.
- PARTIALLY: the palette-distance test iterates Object.entries(typeStyles), so it covers a new entry automatically — but only once the entry exists. It fires when a colour is too close or malformed, never when the type is absent.
- NOT FOR A NEW TYPE: item 4, the TypeIcon dispatcher. web/src/icons.test.tsx gained, in #668, one test that TypeIcon draws `frame` with FrameIcon and not UnknownIcon; it names frame only, so the next type is again unchecked. Add a matching test or check it by eye.

So items 3, 4 and 5 have no test that fails when you forget them. That is the hole F01 fell into.

Do NOT trust the leaf COUNT in this title as a constant to code against; it has moved three times (12 → 13 → 14). Derive it from the helpers.

Every src ref below is pinned at 1eb481b6, the PR #668 merge commit, where the claims above were re-read: 14 leaves and the two Go maps, the derived lists, the factTypeDocs entry for frame, the typeStyles/TypeIcon/spec entries, and the hard-coded expectedTypes list.
