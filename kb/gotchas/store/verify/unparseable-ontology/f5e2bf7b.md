---
type: observation
domain: [store, verify, ontology]
confidence: 0.9
sources: 1
entities: [incomingValid, ontologyOf, loadOntology, ReadVerifySettings]
motifs: [asymmetric-fail-modes]
refs: ['src://7b4887ce51d9/internal/engine/store/gitserve/httphandler_receivepack.go@0610c20083fcef9e84f42ab0311a14708bcb6193:211af41a8ac17ed7083b0eb42e46f6530b900499', 'src://7b4887ce51d9/internal/engine/store/facts/fact_write_validate.go@0610c20083fcef9e84f42ab0311a14708bcb6193:dffe12c47544b0323813697c787de36fb4051dc4', 'kb://3ec012f5b4d2/kb/invariants/store/verify/validation-net-change/87cace22.md', 'https://github.com/knomit/knomit/pull/533']
---
# A repository whose main's ontology does not parse behaves ASYMMETRICALLY: receive-pack and the fetch judge refuse EVERY push in EVERY verify_signatures mode (the rules cannot be known), while the store's own writers (validateWrite) skip validation — the likely trigger is an older binary hosting an ontology a newer one wrote

VERIFIED at squash commit 6f241959 (PR #533). incomingValid (internal/engine/store/gitserve/httphandler_receivepack.go) reads the ontology at upstream's tip with ontologyOf; a parse error returns `knomit: the ontology at <upstream> could not be read: ...` and the push is refused, in off mode too. Before #533 an off-mode host read only verify_signatures (ReadVerifySettings parses less) and admitted such pushes, so an off-mode host that now cannot parse its ontology starts refusing pushes it used to take. The store's writers skip: validateWrite runs nothing for a branch whose ontology does not parse, because a repository that cannot read its ontology is made unwritable where it is opened (loadOntology), and in-process writers cannot write .knomit/ (private-path rule), so the skip is not exploitable. `knomit verify ci` (CheckRange) now parses the ontology even with verification off; an unparseable one is exit 2.

LIKELY TRIGGER: an older binary hosting a repository whose ontology a newer binary wrote (a new key, validation or artifacts form the older parser rejects). CONSEQUENCE: upgrade the host; do not 'fix' it by loosening the entry check. MISREADINGS: 'writes work, so pushes must too': no, the two sides differ on purpose. 'Off mode never refuses': since #533 it refuses every push when main's ontology does not parse.
