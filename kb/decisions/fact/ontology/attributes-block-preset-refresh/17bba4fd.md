---
kind: pragmatic
type: policy
domain: [fact, ontology, repos]
confidence: 0.9
sources: 1
entities: [Ontology.IsSubsetOf, Ontology.SubsetDivergence, nodeIsSubsetOf, refreshDivergence, RepoInstance.loadOntology, EmbeddedPresetByID, internal/repos/stages.go, internal/fact/ontology.go, TestLoadOntology_AttributesSurviveBoot]
motifs: [consumer-defines-check]
refs: ['kb://3ec012f5b4d2/kb/invariants/repos/ontology/refresh-preserves-root-attributes/77ad0332.md', 'src://7b4887ce51d9/internal/repos/stages.go@c9c599dee475771c4226212ac0889817c6b268d4:49cea475154491f62031b7fdbe035f5523fbddbf', 'src://7b4887ce51d9/internal/fact/ontology.go@c9c599dee475771c4226212ac0889817c6b268d4:f6267d0856e06df8b417a64552c607de81fccc23', 'src://7b4887ce51d9/internal/repos/builder_ontology_test.go@c9c599dee475771c4226212ac0889817c6b268d4:3a59864a6aa5919ce4b0217d7537bc188e4a6ac3', 'src://7b4887ce51d9/spec/mbekg.md@331305497a626c1ae5f9f76e744716e888e7e268:bb669b04dcdd6215474f8936a32b3b734f574d1a', 'https://github.com/knomit/knomit/pull/255']
---
# IsSubsetOf treats ontology attributes as DIVERGENCE, because the boot refresh overwrites any subset with the preset — a flagged preset repo keeps its file and forgoes auto-upgrade

`Ontology.IsSubsetOf` is not a pure shape comparison. Its meaning is set by the boot-time refresh in `RepoInstance.loadOntology` (internal/repos/stages.go), which reads "subset" as SAFE TO OVERWRITE. The refresh no longer calls IsSubsetOf itself (production code has no IsSubsetOf caller; tests do): it calls `refreshDivergence`, which is `SubsetDivergence` plus a requirement that the ROOT attributes are identical (`fact.RootAttributesEqual`; see kb/invariants/repos/ontology/refresh-preserves-root-attributes). When the stored ontology's id matches an embedded preset, refreshDivergence is empty, and the serialized bytes differ, it WRITES the preset over the stored file.

The F02 brief originally said IsSubsetOf should IGNORE attributes ("behaviour, not shape"). That would have erased a flag on every boot. Take a source-code repo whose only change is `learn_dedup: off` on a preset topic: its taxonomy is a subset, its bytes differ because of the attribute, and so the preset is written over it.

OPTIONS CONSIDERED:
- A (chosen): IsSubsetOf compares attributes by value. A stored attribute must be present with an equal value (reflect.DeepEqual) in the preset. A value that behaves as ABSENT (the key's `absent` in attributeRegistry; for learn_dedup that is "on") is compared as absent, so spelling out the default never stops an upgrade. Attributes the PRESET carries and the stored file lacks are fine, since the preset is only adding. A flagged repo takes the existing "diverged, upgrade skipped" path, the same as a custom topic or rule.
- B (deferred): keep ignoring attributes and have the refresh graft the stored attributes onto the preset before writing. Upgrades would keep flowing, but stages.go loadOntology would become a second place that has to know attributes exist.

WHY A: it is small, and it fails safe. The cost is that a repo that flags a preset topic stops receiving preset auto-upgrades. To make that cost visible, `Ontology.SubsetDivergence(other)` returns "" (subset), "attributes" (taxonomy and validations would be a subset, and only attributes differ) or "shape" (shape wins when both differ). Since this fact was written SubsetDivergence also reports "triggers", "context" and "guidance", in the order shape > attributes > triggers > context > guidance (ontology.go's switch). It computes this in ONE walk, and the refresh's warning carries it as `reason`. B stays open as a follow-up if flagged preset repos turn out to need upgrades. F08 will likely ship its own coordination preset.

CONSEQUENCE: do not "simplify" IsSubsetOf back to shape-only. Its meaning, and SubsetDivergence's, is set by the overwrite in loadOntology. TestLoadOntology_AttributesSurviveBoot (repos) and TestOntologyAttr_AttributesAreDivergenceFromAPreset (fact, including the explicit-`on` case) pin both halves.
