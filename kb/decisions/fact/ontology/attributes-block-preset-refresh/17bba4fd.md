---
kind: pragmatic
type: policy
domain: [fact, ontology, repos]
confidence: 0.9
sources: 1
entities: [Ontology.IsSubsetOf, Ontology.SubsetDivergence, nodeIsSubsetOf, repoBuilder.loadOntology, EmbeddedPresetByID, internal/repos/builder.go, internal/fact/ontology.go, TestLoadOntology_AttributesSurviveBoot]
motifs: [consumer-defines-check]
refs: ['src://7b4887ce51d9/internal/repos/builder.go@a1f4bf09acbda4649ccbbf4a7f7a1e23e01098be:61abb3d315be765bc6717cd800ee27c0ab8fd20d', 'src://7b4887ce51d9/internal/fact/ontology.go@a1f4bf09acbda4649ccbbf4a7f7a1e23e01098be:57ddc4950faab902931266a33c80f6f8cba823a9']
---
# IsSubsetOf treats ontology attributes as DIVERGENCE, because the boot refresh overwrites any subset with the preset — a flagged preset repo keeps its file and forgoes auto-upgrade

`Ontology.IsSubsetOf` is not a pure shape comparison. Its only production caller, the boot-time refresh in `repoBuilder.loadOntology` (internal/repos/builder.go), reads "subset" as SAFE TO OVERWRITE. When the stored ontology's id matches an embedded preset, IsSubsetOf(preset) holds, and the serialized bytes differ, it WRITES the preset over the stored file.

The F02 brief originally said IsSubsetOf should IGNORE attributes ("behaviour, not shape"). That would have erased a flag on every boot. Take a source-code repo whose only change is `learn_dedup: off` on a preset topic: its taxonomy is a subset, its bytes differ because of the attribute, and so the preset is written over it.

OPTIONS CONSIDERED:
- A (chosen): IsSubsetOf compares attributes by value. A stored attribute must be present with an equal value (reflect.DeepEqual) in the preset. Attributes the PRESET carries and the stored file lacks are fine, since the preset is only adding. A flagged repo takes the existing "diverged, upgrade skipped" path, the same as a custom topic or rule.
- B (deferred): keep ignoring attributes and have the refresh graft the stored attributes onto the preset before writing. Upgrades would keep flowing, but repos/builder.go would become a second place that has to know attributes exist.

WHY A: it is small, and it fails safe. The cost is that a repo that flags a preset topic stops receiving preset auto-upgrades. To make that cost visible, `Ontology.SubsetDivergence(other)` returns "" (subset), "attributes" (taxonomy and validations would be a subset, and only attributes differ) or "shape". The refresh's warning carries it as `reason`. B stays open as a follow-up if flagged preset repos turn out to need upgrades. F08 will likely ship its own coordination preset.

CONSEQUENCE: do not "simplify" IsSubsetOf back to shape-only. Its meaning is set by the overwrite in loadOntology. TestLoadOntology_AttributesSurviveBoot (repos) and TestOntologyAttr_AttributesAreDivergenceFromAPreset (fact) pin both halves.
