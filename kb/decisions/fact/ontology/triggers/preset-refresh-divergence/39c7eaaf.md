---
kind: pragmatic
type: policy
domain: [fact, ontology, repos, triggers]
confidence: 0.9
sources: 1
entities: [Ontology.SubsetDivergence, DivergenceTriggers, nodeIsSubsetOf, triggersSubset, refreshDivergence, serializeNode, internal/fact/ontology.go]
motifs: [consumer-defines-check]
refs: ['kb://3ec012f5b4d2/kb/decisions/fact/ontology/attributes-block-preset-refresh/17bba4fd.md', 'kb://3ec012f5b4d2/kb/invariants/repos/ontology/refresh-preserves-root-attributes/77ad0332.md', 'src://7b4887ce51d9/internal/fact/ontology.go@c9c599dee475771c4226212ac0889817c6b268d4:f6267d0856e06df8b417a64552c607de81fccc23', 'src://7b4887ce51d9/internal/repos/stages.go@c9c599dee475771c4226212ac0889817c6b268d4:49cea475154491f62031b7fdbe035f5523fbddbf', 'https://github.com/knomit/knomit/pull/332']
---
# Triggers are preset-refresh DIVERGENCE (reason "triggers"): a stored node whose triggers the preset does not carry identically keeps its file and forgoes auto-upgrade; presets carry none today

`nodeIsSubsetOf` (internal/fact/ontology.go) sets `behaviourDivergence.triggers` when a stored node declares triggers the preset does not declare byte-identically (compared as marshalled YAML via `triggersSubset`). `SubsetDivergence` then returns `DivergenceTriggers` ("triggers"), and priority is shape > attributes > triggers > context > guidance (context and guidance reasons were added after this fact; the switch is in SubsetDivergence).

WHY: the boot-time refresh in `RepoInstance.loadOntology` (internal/repos/stages.go) WRITES the preset over any stored ontology for which `refreshDivergence` (SubsetDivergence plus identical root attributes) is empty. Ignoring triggers there would erase a repo's triggers on every boot. This is the same reasoning as for attributes (kb/decisions/fact/ontology/attributes-block-preset-refresh).

CONSEQUENCES:
- A preset-derived repo that declares any trigger stops receiving preset auto-upgrades. The refresh logs `reason=triggers`.
- `TestEmbeddedPresetsCarryNoTriggers` pins that the embedded presets have none. BEFORE a preset ships a trigger, `refreshDivergence` must also refuse to ADD triggers, as it refuses to add root attributes (kb/invariants/repos/ontology/refresh-preserves-root-attributes). Otherwise an upgrade would silently add fleet-wide behaviour in an own-signed commit.
- `Serialize` writes a node's `triggers` back verbatim from the original yaml node, including unknown keys and malformed entries, so a round trip never drops a newer knomit's triggers.
