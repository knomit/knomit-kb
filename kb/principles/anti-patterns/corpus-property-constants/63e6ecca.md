---
kind: pragmatic
type: policy
domain: [global]
confidence: 1
sources: 1
entities: [designer]
motifs: [false-universal-default]
refs: ['src://7b4887ce51d9/internal/corpus/synthesize/restatement_conformance_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:ae5c73211af8f69bcd0786a888f955f67bc22e0f', 'src://7b4887ce51d9/internal/domain/fact/motif.go@0610c20083fcef9e84f42ab0311a14708bcb6193:45e8aa546c262bdacd5cf3f53cf3a75acf8b8881', 'src://7b4887ce51d9/internal/corpus/synthesize/bridge_motif.go@0610c20083fcef9e84f42ab0311a14708bcb6193:b5de060e9d11ed88a259f6d3f4590fc0d973c4e3', 'src://7b4887ce51d9/internal/corpus/synthesize/motif_disjoint.go@0610c20083fcef9e84f42ab0311a14708bcb6193:55625d5833b1d385622aefbaca6a14e1523f4629', 'kb://3ec012f5b4d2/kb/decisions/synthesize/motif/per-corpus-activation/389e94d1.md', 'kb://3ec012f5b4d2/kb/decisions/synthesize/motif/df-graded-disjointness-gate/5a51f321.md', 'kb://3ec012f5b4d2/kb/decisions/synthesize/consolidation-scope-fix/467f51b7.md', 'https://github.com/knomit/knomit/pull/99']
---
# Constants encoding a CORPUS PROPERTY are forbidden as fixed values — derive them from the repo's own distributions or do not have them

Designer ruling, MN13 (2026-08-21), amended 2026-08-24 to add a third class.

Every numeric constant in this codebase is one of three classes, and the class must be stated where the constant is defined.

1. CORPUS-PROPERTY constants — FORBIDDEN as fixed values. Absolute cosines, density cuts, expected rates, anything that claims a fact about a corpus. These must be derived from the repo's own data — percentiles of its own distributions, its own judge-verdict history — or not exist. A number measured on a research corpus is CALIBRATION EVIDENCE, never a shipped value.

The reason is that such a constant is a claim, and it is a claim about a corpus the code will never see. Identical code prints an operating point of 0.783 on knomit-kb and 0.884 on core. Shipping either number would be asserting that every future corpus resembles the one it was measured on — and the assertion is invisible, because a float literal looks like a setting rather than an empirical claim.

2. RESOURCE-BUDGET constants — legitimate, but must be NAMED and DOCUMENTED as budgets where they are defined. Latency budgets, judge slots, LLM spend, structural K, SQL parameter chunk sizes. These assert nothing about any corpus; they say what we are willing to spend. MaxMotifs = 3 is one: it is a write-discipline budget, and its doc comment says so in those words.

3. STATISTICAL-VALIDITY FLOORS — legitimate, as a minimum POPULATION. Added 2026-08-24 for motifActivationFloor = 3. A floor is not a proportion and not a claim about any corpus's distribution: below it there are too few instances IN EXISTENCE for the mechanism to be doing anything but spending slots on noise, and below it THE MECHANISM DOES NOTHING. That last part is the test — it is what distinguishes this class from a threshold someone picked and then defended.

Enforced by test, not by review. TestConformance_NoCorpusPropertyConstants parses the phase's files and fails on any undeclared float literal. This is deliberate: class 1 violations are exactly the kind that survive review, because a plausible number in a plausible place reads as calibrated.

WHAT THIS DOES NOT MEAN.

- It does NOT ban numbers. Two of the three classes are legitimate; the rule is that the class must be declared at the definition site, so the next reader can tell an empirical claim from a spending decision.
- It does NOT mean derived values need no justification. A percentile is a choice too — p90 was measured and confirmed only as HARMLESS (cuts move 2→9 across p80–p95 with identical candidate and served counts), which is weaker than confirmed correct, and the fact should say so.
- It does NOT make a research measurement useless. It makes it evidence for choosing a DERIVATION, not a value to paste.
