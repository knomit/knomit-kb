---
type: pattern
domain: [testing, archtest, layering]
confidence: 0.9
sources: 1
entities: [TestLayers, sameLayerEdges, checkLayers]
motifs: [unreachable-allowlist-entry]
refs: ['src://7b4887ce51d9/test/archtest/layers_test.go@fcc1613da698d7a18b02381959257a6f00029c3c:0610c20083fcef9e84f42ab0311a14708bcb6193', 'https://github.com/knomit/knomit/pull/592', 'kb://3ec012f5b4d2/kb/decisions/architecture/internal-layer-hierarchy/c81f5882.md']
---
# An allowlist guard must fail on stale keys: require every allowlist entry to be observed in the union of build configurations, or deleting an import leaves a free pass for its reintroduction

An allowlist of permitted exceptions (TestLayers' sameLayerEdges, the permitted same-layer imports) silently widens if entries outlive the edges they permit: once the import is deleted, the key stays and a later reintroduction of that edge passes unnoticed. Fix shape used in test/archtest/layers_test.go: checkLayers returns every same-layer edge it observed (allowed or not), TestLayers unions them across all build configurations (default, desktop, linux, windows, darwin), and any sameLayerEdges key absent from the union fails as stale ('delete it'). The union is taken across configurations on purpose: an edge that exists only under one GOOS or tag stays live because some configuration observes it.

Consequence: when you delete or move an import that an allowlist covers, delete the key in the same change, or the guard fails by name. Does NOT mean each configuration must observe every key; one configuration is enough.
