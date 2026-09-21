---
kind: pragmatic
type: policy
domain: [fact, testing, web, validation]
confidence: 0.95
sources: 1
entities: [TestMN4_MotifValidationHasOneCallSite, motifGateCallSites, StripSubjectMotifs, SerializeFact, handlers_fact_write.go]
motifs: [text-scan-sees-comments, rationale-copied-premise-lost]
refs: ['src://7b4887ce51d9/internal/fact/motif_conformance_test.go@629d0315068ffec8b3b930d1acdeff295d2b88e6:01e37d32c557eb4b790d0e64d863a2c5756448ce', 'src://7b4887ce51d9/internal/fact/format.go@629d0315068ffec8b3b930d1acdeff295d2b88e6:39aed3ebb1b309e05e27a8654b2b6900d3c6f5b3', 'https://github.com/knomit/knomit/pull/239']
---
# internal/fact's motif conformance test matches on SOURCE TEXT, so deleting an offending call leaves it red while the helper's name survives in a comment

`TestMN4_MotifValidationHasOneCallSite` scans files for the gate helpers' NAMES. Removing the offending statement is therefore not sufficient: a replacement COMMENT explaining why the call was removed keeps the test red, and reads to anyone reproducing the fix as "the fix did not work".

The underlying rule is worth stating separately from the mechanism. A direct caller needs a `motifGateCallSites` entry only when the path genuinely never reaches `SerializeFact`, which applies the subject-motif gate internally. Needing an entry usually means the MISSING SerializeFact call is the real defect — the allowlist's own comment says so. The REST raw-editor PUT is on it legitimately, because it commits the client's bytes verbatim and is the one write path that never reaches serialization on its own.

HOW THIS BIT: a resolution path copied that PUT's explicit gate call AND its justifying comment. The call was surplus — that path ends in SerializeFact — and the comment was false in its new home. A rationale copied along with a call does not inherit its premise, and the comment made the surplus call look deliberate.
