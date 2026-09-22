---
kind: pragmatic
type: policy
domain: [fact, ontology]
confidence: 0.9
sources: 1
entities: [OntologyNode.Attributes, attributeValidators, Ontology.Attr, Ontology.LearnDedupOff, fact.AttrLearnDedup, attributeDiags, ValidateOntologyYAML, ParseOntology, compiledRulesCache, serializeNode, internal/fact/ontology.go, internal/fact/diagnostics.go]
motifs: [forward-compatible-reader]
refs: ['src://7b4887ce51d9/internal/fact/ontology.go@a1f4bf09acbda4649ccbbf4a7f7a1e23e01098be:57ddc4950faab902931266a33c80f6f8cba823a9', 'src://7b4887ce51d9/internal/fact/diagnostics.go@a1f4bf09acbda4649ccbbf4a7f7a1e23e01098be:510e9663343ecc19f2524fd152c6e3aafb8ed551', 'src://7b4887ce51d9/spec/mbekg.md@a1f4bf09acbda4649ccbbf4a7f7a1e23e01098be:3f530c02a95331a6d3d40e1cd4962017bbc152df', 'kb://3ec012f5b4d2/kb/gotchas/fact/ontology/bare-topic-key-is-a-nil-node/b5b7bc15.md']
---
# Ontology nodes carry an `attributes` block that switches store behaviour, resolved by the ValidateFact walk; an unknown key is a WARNING, a bad value for a known key is FATAL

`OntologyNode.Attributes map[string]any` (yaml `attributes`) lets a topic or child node switch store behaviour for facts under it. Root-level attributes do not exist yet (F07 may add them).

DECLARATION: `attributeValidators` in internal/fact/ontology.go is the ONLY place a key is declared. Each entry maps a key to a value validator. The first key is `learn_dedup` (constant `fact.AttrLearnDedup`), whose values are exactly the strings "off" and "on". "on" behaves as absent; it exists so a child can undo a parent's "off".

RESOLUTION: `Ontology.Attr(topicPath, key)` walks exactly like `ValidateFact`. It lowercases each segment and goes root → topic → each DECLARED child, stopping at the first undeclared segment. The nearest declared value wins, and an undeclared deeper category inherits from its deepest declared ancestor. A bare key (nil node) counts as declared at any depth, matching ValidateFact. The resolved maps are cached per declared prefix in `compiledRulesCache.attrsByTopic`, built in the same pass as the rules cache, and are read-only after parse. Callers use the typed accessor `LearnDedupOff(topicPath)`, which is nil-safe, and never string-compare Attr's result.

VALIDATION: `ValidateOntologyYAML` (`attributeDiags`) walks the FULL tree. The kebab-case key check next to it walks only topics plus one level of children. An UNKNOWN key produces a SeverityWarning diagnostic that names the topic path, and the ontology is still returned. This deliberately deviates from proposal F02, which said to reject it. ParseOntology's comment gives the reason: the open path (repos/builder.go) must be able to read an ontology written by a NEWER knomit, and a fatal diagnostic there would leave the repo unable to accept writes. A BAD VALUE for a KNOWN key is a fatal error, because a binary that knows the key must not guess.

SCOPE: learn_dedup governs knomit_learn only. Review never reads it. If review ever needs to hold off, that is a second attribute with its own name.

SERIALIZE: `serializeNode` emits `attributes` with sorted keys. The yaml encoder quotes `"off"`/`"on"`, and they re-parse as the same strings. Without this, initSeed and custom-create would drop the block on commit.

CONSEQUENCE: adding a key means one entry in attributeValidators plus a typed accessor. Never read `node.Attributes` directly, because it holds only that node's own values and not what it inherits. Attributes are also divergence for IsSubsetOf; see the attributes-block-preset-refresh decision.
