---
type: observation
domain: [build, release, tooling, changelog]
confidence: 0.9
sources: 1
entities: [tools/relnotes/changes.go, renderChanges, RenderChanges, RenderForDistill, Changes.Direct, TestCollectSeparatesDirectCommitsFromPRCommits, -bodies-out, Collect]
refs: ['src://7b4887ce51d9/tools/relnotes/changes.go@c49944d61f8708462d3e69aa55c2a5ce42e27af5:01098631ed19899fa7064e784ed342d22e6aa3de#L118-L159', 'src://7b4887ce51d9/tools/relnotes/changes_test.go@c49944d61f8708462d3e69aa55c2a5ce42e27af5:bfea66d8c033e8d0994597a931ee83b627758b92', 'https://github.com/knomit/knomit/pull/69']
---
# A missing `### Direct commits` section in relnotes output is CORRECT, not a bug — and `RenderChanges` must never grow bodies

Two easy-to-"fix" non-bugs in `tools/relnotes`.

**1. The absent `### Direct commits` section.** `renderChanges` emits that heading only when `len(c.Direct) > 0`. It exists so a hotfix pushed straight to `dev` — reachable from no PR merge — can never vanish silently from a changelog. On the reference range `312d532b^..4154e92c` it correctly does NOT render: all 73 non-merge commits there arrived through the 9 PR merges. **Do not "fix" its absence.** An earlier draft of the PR #69 design wrongly claimed `bump version (0abeb089)` was a direct commit; it came via PR #50. The behaviour is covered by `TestCollectSeparatesDirectCommitsFromPRCommits` — that test, not release history, is what proves the path works, so verify against it before changing the condition.

**2. Stdout must stay title-only.** `RenderChanges` (→ stdout, the changelog users read in the release notes) and `RenderForDistill` (→ `-bodies-out <path>`, PR bodies appended, fed to the model) are thin wrappers over ONE function, `renderChanges(c Changes, withBodies bool)`. The `withBodies` bool is the seam and both wrappers run off the same `Collect` result, so the enriched variant costs no extra `gh pr view` calls. Never route bodies into the `withBodies=false` path to "share more code" — the two outputs have different audiences (humans reading a release page vs. a model needing rationale), and stdout growing 30 KB of PR bodies is the failure this separation prevents.

**WHAT THIS DOES NOT MEAN:** the two renderers are NOT duplicated implementations awaiting a refactor — they already share one. The thing that must not be unified is the OUTPUT, not the code.
