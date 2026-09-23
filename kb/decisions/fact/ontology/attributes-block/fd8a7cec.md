---
kind: pragmatic
type: policy
domain: [fact, ontology]
confidence: 0.9
sources: 1
entities: [OntologyNode.Attributes, attributeValidators, Ontology.Attr, Ontology.LearnDedupOff, fact.AttrLearnDedup, attributeDiags, ValidateOntologyYAML, ParseOntology, compiledRulesCache, serializeNode, internal/fact/ontology.go, internal/fact/diagnostics.go]
motifs: [forward-compatible-reader]
refs: ['src://7b4887ce51d9/internal/fact/ontology.go@331305497a626c1ae5f9f76e744716e888e7e268:c053731a101467d6d4549f6330ce5b16c6220893', 'src://7b4887ce51d9/internal/fact/diagnostics.go@331305497a626c1ae5f9f76e744716e888e7e268:9315a5a18f7d554bd194d2c5e09f83efb844a847', 'src://7b4887ce51d9/internal/mcp/learn.go@331305497a626c1ae5f9f76e744716e888e7e268:39f0cb3b3ad7edbd2dc7654ca556a46867526519', 'src://7b4887ce51d9/spec/mbekg.md@331305497a626c1ae5f9f76e744716e888e7e268:bb669b04dcdd6215474f8936a32b3b734f574d1a', 'kb://3ec012f5b4d2/kb/gotchas/fact/ontology/bare-topic-key-is-a-nil-node/b5b7bc15.md', 'kb://3ec012f5b4d2/kb/gotchas/mcp/learn/dedup-search-scope-is-a-string-prefix/047f3cdd.md', 'https://github.com/knomit/knomit/pull/255', 'https://github.com/knomit/knomit/issues/260']
---
# Ontology nodes carry an `attributes` block that switches store behaviour, resolved by the ValidateFact walk; an unknown key is a WARNING, a bad value for a known key is FATAL

`OntologyNode.Attributes map[string]any` (yaml `attributes`) lets a topic or child node switch store behaviour for facts under it. Root-level attributes do not exist yet (F07 may add them).

DECLARATION: `attributeRegistry` in internal/fact/ontology.go is the ONLY place a key is declared. Each entry is an `attributeSpec`: a description of the accepted values, a validator, and the value that behaves as ABSENT. The first key is `learn_dedup` (constant `fact.AttrLearnDedup`), whose values are exactly the strings "off" and "on". "on" is the absent value; it exists so a child can undo a parent's "off".

RESOLUTION: `Ontology.Attr(topicPath, key)` walks exactly like `ValidateFact`. It lowercases each segment and goes root → topic → each DECLARED child, stopping at the first undeclared segment. The nearest declared value wins, and an undeclared deeper category inherits from its deepest declared ancestor. A bare key (nil node) counts as declared at any depth. The resolved maps are cached per declared prefix in `compiledRulesCache.attrsByTopic`, built in the same pass as the rules cache, and are read-only after parse. Callers use the typed, nil-safe `LearnDedupOff(topicPath)` and never string-compare Attr's result.

VALIDATION: `ValidateOntologyYAML` (`attributeDiags`) walks the FULL tree. The kebab-case key check next to it walks only topics plus one level of children.
- An UNKNOWN key produces a SeverityWarning diagnostic naming the topic path, and the ontology is still returned. This deliberately deviates from proposal F02, which said to reject it. The open path (repos/builder.go) must be able to read an ontology written by a NEWER knomit (see ParseOntology's comment).
- A BAD VALUE for a KNOWN key is FATAL, on the open path too: the repo refuses writes with a named error. The alternative, warning and treating the key as absent, would make an older binary silently run dedup on a topic a newer one switched off. That is exactly the silent loss the flag exists to stop. Loud and fixed by upgrading beats silent.

CONSEQUENCE, THE CLOSED-VALUE RULE: each key's value set is CLOSED. A new behaviour ships as a NEW attribute key, never as a new value of an existing key, because an older binary would reject the new value and the repo would stop accepting writes. The rule is also stated in the attributeRegistry comment and in spec §3.2.

WHAT learn_dedup: off DOES (learn only; review never reads it):
- INCOMING side: applyDedupMerge skips the search for a flagged incoming fact, and checkSameSubjectCollisions skips it too.
- CANDIDATE side of the merge: applyDedupMerge also declines a search MATCH whose own topic path (`topicPathOf(ontologyRoot, match.Path)`) is flagged. This is needed because the merge search is a raw path prefix (store's `path LIKE dir%`): a search from an unflagged parent `ops/tasks` returns facts under a flagged child `ops/tasks/protocol/` and would merge into them (PR #255 review; the prefix issue itself is #260).
- Because the incoming skip sits before the search, a flagged fact ALSO SKIPS HYPOTHESIS SUBSUMPTION. An observation that settles a hypothesis under a flagged topic leaves BOTH live, and review never touches the topic either. This is accepted because coordination topics do not carry hypotheses, but it is a consequence, not an accident.

KNOWN LIMITS (stated, not fixed):
1. The flag is learn-only, so review's prune can still merge near-identical KNOWLEDGE-kind facts under a flagged topic later. In effect the flag fully protects signals, which review never seeds on, and protects knowledge facts only at write time.
2. The same-subject CANDIDATE side is not governed by the attribute, only by F01's type skip (signal, hypothesis). A flagged topic filling with entity-sharing OBSERVATIONS can therefore still cause same-subject refusals for unflagged incoming facts elsewhere.

SERIALIZE: `serializeNode` emits `attributes` with sorted keys and returns an encode error instead of dropping the key. The yaml encoder quotes `"off"`/`"on"`, and they re-parse as the same strings.

CONSEQUENCE for adding a key: add one attributeRegistry entry (including its absent value, if any) plus a typed accessor. Never read `node.Attributes` directly, because it holds only that node's own values and not what it inherits. Attributes are also divergence for IsSubsetOf; see the attributes-block-preset-refresh decision.
