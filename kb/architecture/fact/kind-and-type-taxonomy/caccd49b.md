---
type: observation
domain: [fact, taxonomy]
confidence: 0.95
sources: 0
entities: [Kind, Type, EpistemicTypes, PragmaticTypes, AllEpistemicTypes, AllPragmaticTypes, DefaultKind, DefaultEpistemicType, factTypeDocs, typeStyles, TypeIcon, spec/mbekg.md]
motifs: [undetected-derivation-gap]
refs: ['src://7b4887ce51d9/internal/fact/kind.go@a7bed6733bd534cbc30f181e12c3d9612559432d:e9fedce857eaa7b804658b10aef47665436e4d5f', 'src://7b4887ce51d9/internal/fact/epistemic_type.go@a7bed6733bd534cbc30f181e12c3d9612559432d:967a848d86a6d4968d8a21b55d182ed940d3c6ec', 'src://7b4887ce51d9/internal/store/index.go@a7bed6733bd534cbc30f181e12c3d9612559432d:86800b25927ee62bb77cafbf11d913425613c479', 'src://7b4887ce51d9/internal/mcp/instructions.go@a7bed6733bd534cbc30f181e12c3d9612559432d:f00cca0d16b17d2f99e3a2b9d886d850414334e3', 'kb://3ec012f5b4d2/kb/decisions/fact/pragmatic-types/signal/e48a7a32.md', 'kb://3ec012f5b4d2/kb/decisions/fact/epistemic-types/insight/f9640cdc.md']
---
# Kind has 2 members (Epistemic, Pragmatic); Type has 13 leaves; authoritative sets live in maps

Kind (kind.go) is a string-typed enum with two members: Epistemic ('epistemic') and Pragmatic ('pragmatic'). Type (epistemic_type.go, pragmatic_type.go) is also string-typed and carries 13 leaf values: 10 epistemic (Observation, Concept, Process, Principle, Pattern, Reference, Synthesis, Insight, Hypothesis, Methodology) + 3 pragmatic (Policy, Heuristic, Signal).

Insight was added in PR #70 (ordered between Synthesis and Hypothesis) — see [[insight]]. Signal was added by F01 on branch `feat/signal-type`, ordered last among the pragmatic types — see [[signal]]; it types coordination content that is consumed rather than believed.

Authoritative sets: EpistemicTypes map[Type]bool and PragmaticTypes map[Type]bool — the SINGLE source of truth used by Kind.AllowsType(t). Stable ordering via AllEpistemicTypes() / AllPragmaticTypes() for code that needs deterministic iteration. DefaultKind = Epistemic; DefaultEpistemicType = Observation — pragmatic facts have NO default type (policy ≠ heuristic ≠ signal; the author must declare which). Type itself is the umbrella — the previously-separate EpistemicType was collapsed into Type to remove call-site casts.

## Adding a new Type: the full checklist

An earlier revision of this fact said that after the Go helpers "nothing else needs editing", because the consumers it knew about all derive their lists. That was wrong, and F01 proved it: two hand-maintained tables in the web UI have no derivation, so a new type reached the Go enum, the MCP schema and the spec while the UI still could not draw it.

DERIVED — no edit needed, verified at a7bed673:
- internal/mcp/instructions.go renders both type lists via instructionTypeLines(fact.All*Types(), ...).
- internal/store/index.go builds the `type` completion list from AllEpistemicTypes()+AllPragmaticTypes().

MUST BE EDITED BY HAND:
1. The EpistemicTypes or PragmaticTypes map AND the matching AllXxxTypes() helper (internal/fact).
2. internal/mcp/factschema.go — a factTypeDocs entry (gloss + aside).
3. web/src/utils.ts — a `typeStyles` entry (color, bg, label, icon). The new colour must sit ≥ 60 weighted-RGB from EVERY existing type colour, and 4- or 8-digit hex is rejected.
4. web/src/icons.tsx — an SVG component plus a `case` in the TypeIcon dispatcher, or the type silently renders as UnknownIcon.
5. spec/mbekg.md — the epistemic or pragmatic type table (§2.4).
6. web/src/utils.test.ts — the hand-written type list in the matching per-kind entry test (see below).

## Which of those a test will actually catch

This distinction matters more than the checklist, because a hand-list that looks like a guard is worse than no guard.

- CATCHES A MISSING TYPE ON ITS OWN: TestAll{Epistemic,Pragmatic}Types_MatchesSet, which compares map against slice (a disagreement ships a tool-schema enum rejecting a type the server accepts); and TestFactSchema_DescriptionsAreComplete, which fails until item 2 exists. Between them, items 1 and 2 are enforced.
- DOES NOT: the per-kind entry tests in web/src/utils.test.ts ('has entries for all 10 epistemic types', 'has entries for all 3 pragmatic types'). Both iterate a HARD-CODED `expectedTypes` array, so a fourth pragmatic type is simply not looked at until someone extends the list. That is item 6, and it is maintenance, not protection.
- PARTIALLY: the palette-distance test iterates Object.entries(typeStyles), so it covers a new entry automatically — but only once the entry exists. It fires when a colour is too close or malformed, never when the type is absent.
- NOT AT ALL: item 4, the TypeIcon dispatcher. web/src/icons.test.tsx does not touch TypeIcon. Check it by eye.

So items 3, 4 and 5 have no test that fails when you forget them. That is the hole F01 fell into.

Do NOT trust the leaf COUNT in this title as a constant to code against; it has moved twice (12 → 13). Derive it from the helpers.

Ref note: kind.go, epistemic_type.go, index.go and instructions.go are pinned at a7bed673 and are unchanged by F01. pragmatic_type.go, factschema.go, web/src/utils.ts, web/src/icons.tsx and spec/mbekg.md ARE changed on feat/signal-type and carry no blob ref yet — they must be pinned to the merge commit before this fact leaves the experiment.
