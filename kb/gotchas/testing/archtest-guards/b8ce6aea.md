---
type: pattern
domain: [testing, archtest, layering]
confidence: 0.9
sources: 1
entities: [TestLayers, checkLayers, internalGoDirs, go list]
motifs: [bypass-through-outside-node, measured-constant-floor]
refs: ['src://7b4887ce51d9/test/archtest/layers_test.go@fcc1613da698d7a18b02381959257a6f00029c3c:0610c20083fcef9e84f42ab0311a14708bcb6193', 'https://github.com/knomit/knomit/pull/592', 'kb://3ec012f5b4d2/kb/decisions/architecture/internal-layer-hierarchy/c81f5882.md']
---
# A layer check that inspects only imports inside the guarded tree is bypassed by laundering through a non-internal package; and a non-vacuity size floor is a measured constant, so derive the expected set from the filesystem

LAUNDERING. If a layer rule only examines edges between packages under internal/, an internal package can reach a higher layer by importing a non-internal knomit package (for example under tools/ or cmd/) that itself imports a higher layer. Fix shape in TestLayers (test/archtest/layers_test.go): list the whole module (knomit/...), not only internal/, and require that any non-internal knomit package an internal package imports imports nothing from knomit. This is derived from the go list data, with no allowlist; a non-internal package absent from the listing also fails. The only real edge today is edge/web -> knomit/web (the embedded UI), which is a leaf.

FLOOR. A non-vacuity check that asserts 'at least N packages were seen' encodes a measured constant that drifts on every package add or remove. Derive the expected set from the filesystem instead: internalGoDirs walks internal/ for directories with a .go file (skipping testdata and names starting with _ or .), and each must be listed by at least one build configuration, else the failure names the directory; a walk finding no directories fails rather than passing vacuously.

Does NOT mean test files are covered: go list .Imports omits _test.go files, and tests may import anything by user ruling.
