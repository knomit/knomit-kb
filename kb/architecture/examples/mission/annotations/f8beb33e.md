---
type: reference
domain: [fleet, mission, examples, ontology, context]
confidence: 0.9
sources: 1
entities: [knomit-recipes mission template, annotations, F26, F22 context, target knowledge base, TestMissionAnnotations_TypedByContext, TestMissionAnnotations_TwoTasksTwoPaths, TestMissionAnnotations_QueryByContext]
motifs: [typed-side-channel, prefix-filter-reach]
refs: ['kb://3ec012f5b4d2/kb/architecture/examples/mission/cross-check/d70b0d8c.md', 'kb://3ec012f5b4d2/kb/decisions/fact/context/f22-design/4b5f11d0.md', 'kb://3ec012f5b4d2/kb/invariants/fact/context/two-gates/ebd742be.md', 'kb://3ec012f5b4d2/kb/architecture/examples/mission/template/cce49f62.md', 'src://034f37d5b4a5/.knomit/templates/mission/.knomit/ontology.yaml@ddc3864365dcba77b489aa85c5cf843df5279eb5:19c66370d79b5b894cebd9c2810b512e6d1515dc', 'src://034f37d5b4a5/.knomit/templates/mission/README.md@ddc3864365dcba77b489aa85c5cf843df5279eb5:2c59de66d41fff74c458d211725ea05cd7854122', 'src://7b4887ce51d9/.gitmodules@0e9288eb1d53759184f5bb3e57b5d257cd277ea3:e5e25d3a756910b6034d32976a4bfea569327b23', 'src://7b4887ce51d9/internal/mcp/mission_annotations_test.go@a2a4b6acc994bbe4f3ba46ffa01ceaf59dab2b66:4b58744aa7b3bfed1d13e1879e465201f9fb181c', 'src://7b4887ce51d9/internal/repos/mission_annotations_test.go@0e9288eb1d53759184f5bb3e57b5d257cd277ea3:807662a5114cb5efc84874b9185ae32a58d38f3a']
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

LOCATION (since chore/templates-to-recipes, knomit against dev, with knomit-recipes PR #1): the template, ontology included, is `.knomit/templates/mission/` in https://github.com/knomit/knomit-recipes, no longer knomit `examples/mission/`. knomit's tests read it from the submodule `third_party/knomit-recipes` (pinned commit); see kb/architecture/examples/mission/template/cce49f62.md.
