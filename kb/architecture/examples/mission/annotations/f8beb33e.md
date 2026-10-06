---
type: reference
domain: [fleet, mission, examples, ontology, context]
confidence: 0.9
sources: 1
entities: [examples/mission, annotations, F26, F22 context, target knowledge base, TestMissionAnnotations_TypedByContext, TestMissionAnnotations_TwoTasksTwoPaths, TestMissionAnnotations_QueryByContext]
motifs: [typed-side-channel, prefix-filter-reach]
refs: ['kb://3ec012f5b4d2/kb/architecture/examples/mission/cross-check/d70b0d8c.md', 'kb://3ec012f5b4d2/kb/decisions/fact/context/f22-design/4b5f11d0.md', 'kb://3ec012f5b4d2/kb/invariants/fact/context/two-gates/ebd742be.md', 'src://7b4887ce51d9/examples/mission/.knomit/ontology.yaml@9fc858071f777155b3232a689c6f784deb3224ec:19c66370d79b5b894cebd9c2810b512e6d1515dc', 'src://7b4887ce51d9/examples/mission/README.md@9fc858071f777155b3232a689c6f784deb3224ec:dbc3bfba8434193ac231a3f3f14b23b44525e3bf', 'src://7b4887ce51d9/internal/mcp/mission_annotations_test.go@9fc858071f777155b3232a689c6f784deb3224ec:9ccb493a79309498f6a0235996c6ef7ec38a5b6f', 'src://7b4887ce51d9/internal/repos/mission_annotations_test.go@9fc858071f777155b3232a689c6f784deb3224ec:ee1501b45708623c73858bb805afbcea0cc7b072']
---
# Since F26 the mission template has NO knowledge-base template: knowledge goes to the target KB the charter names (and each task's `knowledge base:` line) as ordinary facts, and the mission repo's one non-signal topic is annotations/<task id>/ (learn_dedup: off), typed by F22 context: kind enum [verdict] required, verdict enum [corroborate, contradict], confidence number 0..1

Built by the F26 template follow-up PR (branch feat/f26-mission-shape, against dev), replacing PR #410's examples/mission-kb/ (forecast + verdicts topics, the hypothesis-format validations), which is deleted with its tests (internal/repos/mission_kb_test.go, internal/mcp/mission_kb_dedup_test.go). User rulings 2026-10-04: a generic mission KB loses the learning; a hypothesis needs no mission format; one topic in the mission repo.

What goes where: findings, syntheses, hypotheses -> the TARGET knowledge base (any existing KB, from any template), under its own topics, as ordinary facts (a hypothesis is plain `type: hypothesis`; TestMission_CrossCheckAnnotatesFoldWrites learns one with an ordinary body). Annotations -> the mission repo, kb/annotations/<task id>/.

The topic as declared in examples/mission/.knomit/ontology.yaml (F22 syntax, the `context:` block sits on the topic node beside `attributes:`, not inside it):
  annotations:
    attributes: {learn_dedup: off}
    context:
      kind:       {type: enum, values: [verdict], required: true}
      verdict:    {type: enum, values: [corroborate, contradict]}
      confidence: {type: number, min: 0, max: 1}
There is no verdicts topic: a verdict is an annotation with context.kind: verdict. A mission adds a kind by adding an enum value before the repo is created (the ontology is fixed at create).

Consequences a consumer acts on:
- An annotation with no kind (or no context at all), an undeclared key (e.g. target), verdict: maybe, kind: note, or confidence 1.5 is REFUSED and the error names the key (TestMissionAnnotations_TypedByContext).
- The target is named ONLY by a ref, kb://<12-hex repo id>/<path>; knomit does not check refs into another repo, so a mistyped target is not refused. The repo id is the first 12 hex of the KB's root commit (knomit_repos lists it), NOT RepoInstance.UID and NOT the full root hash: a 40-hex id is refused as a malformed kb:// ref.
- Reading one task's verdicts: knomit_query path kb/annotations/<task id>/ WITH the trailing slash plus context {kind: verdict}. The path filter is a raw prefix: without the slash, task xcheck-1 also returns xcheck-12's verdicts (TestMissionAnnotations_QueryByContext pins both).
- learn_dedup: off is load-bearing even though each task has its own folder: learn's dedup search scope is a raw path prefix (#260), so a verdict learned into annotations/xcheck-1/ searches annotations/xcheck-12/ too and, with dedup on, folds into the other task's file (TestMissionAnnotations_TwoTasksTwoPaths control).

Misreading to avoid: 'annotations replace hypotheses' — they do not; hypotheses stay in the target KB and only the fold changes them (kb/architecture/examples/mission/cross-check/d70b0d8c.md).
