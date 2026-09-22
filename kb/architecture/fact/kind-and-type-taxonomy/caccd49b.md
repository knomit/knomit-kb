---
type: observation
domain: [fact, taxonomy]
confidence: 0.95
sources: 0
entities: [Kind, Type, EpistemicTypes, PragmaticTypes, AllEpistemicTypes, AllPragmaticTypes, DefaultKind, DefaultEpistemicType]
refs: ['src://knomit/internal/fact/kind.go@307b67d', 'src://knomit/internal/fact/epistemic_type.go@307b67d', 'src://knomit/internal/fact/pragmatic_type.go@307b67d', 'src://knomit/internal/fact/epistemic_type.go@021730b', 'src://knomit/internal/store/index.go@021730b']
---
# Kind has 2 members (Epistemic, Pragmatic); Type has 13 leaves; authoritative sets live in maps

Kind (kind.go) is a string-typed enum with two members: Epistemic ('epistemic') and Pragmatic ('pragmatic'). Type (epistemic_type.go, pragmatic_type.go) is also string-typed and carries 13 leaf values: 10 epistemic (Observation, Concept, Process, Principle, Pattern, Reference, Synthesis, Insight, Hypothesis, Methodology) + 3 pragmatic (Policy, Heuristic, Signal).

Insight was added in PR #70 (ordered between Synthesis and Hypothesis) — see [[insight]]. Signal was added by F01 on branch `feat/signal-type`, ordered last among the pragmatic types — see [[signal]]; it types coordination content that is consumed rather than believed.

Authoritative sets: EpistemicTypes map[Type]bool and PragmaticTypes map[Type]bool — the SINGLE source of truth used by Kind.AllowsType(t). Stable ordering via AllEpistemicTypes() / AllPragmaticTypes() for code that needs deterministic iteration (e.g. autocomplete output, tool schema enums) — PR #70 switched searchIndex.Completions' `type` case to derive from these helpers instead of a hardcoded list, so new types surface automatically.

DefaultKind = Epistemic; DefaultEpistemicType = Observation — pragmatic facts have NO default type (policy ≠ heuristic ≠ signal; the author must declare which). Type itself is the umbrella — the previously-separate EpistemicType was collapsed into Type to remove call-site casts.

Adding a new Type: add it to either EpistemicTypes or PragmaticTypes map AND to the matching AllXxxTypes() helper. Two tests hold you to the rest: TestAll{Epistemic,Pragmatic}Types_MatchesSet catches a map/slice disagreement (which would ship a tool-schema enum rejecting a type the server accepts), and TestFactSchema_DescriptionsAreComplete in internal/mcp fails until the new type has a factTypeDocs entry. Nothing else needs editing — internal/mcp/instructions.go and internal/store/index.go both render their type lists from the AllXxxTypes() helpers rather than restating them.

Do NOT trust the leaf COUNT in this title as a constant to code against; it has moved twice (12 → 13). Derive it from the helpers.
