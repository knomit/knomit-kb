---
type: principle
domain: [synthesize, concurrency]
confidence: 0.95
sources: 0
entities: [Pipeline.opener, Pipeline.StartSession, Pipeline.StartOrResumeSession, Pipeline.ContinueSessionForItem, Pipeline.handlePhase, Pipeline.completeSession, Deps, Strategy, sess.Branch, ri.AgentBranch, store.PipelineSession]
motifs: [out-of-band-state]
refs: ['src://7b4887ce51d9/internal/synthesize/pipeline.go@6d912194ff82aa94b28cdb216e9f750edfada8d6:451a628a325e7edb70b75feb6e515b50298dc7a3#L186-L192', 'src://7b4887ce51d9/internal/synthesize/strategy.go@6d912194ff82aa94b28cdb216e9f750edfada8d6:dcd234b748995034eaf281c233a0cb284ad8fe52', 'src://7b4887ce51d9/internal/synthesize/hypothesize_strategy.go@6d912194ff82aa94b28cdb216e9f750edfada8d6:1f3cb74f109fafcd81c0e85d6501798f4a7c5546', 'kb://3ec012f5b4d2/kb/decisions/synthesize/one-engine-two-drivers/19c69b3f.md', 'kb://3ec012f5b4d2/kb/decisions/mcp/review/start-resumes-live-session/e4f26b91.md']
---
# sess.Branch is captured once, when a session is created (Pipeline.opener); downstream engine code NEVER reads ri.AgentBranch()

`sess.Branch` is captured ONCE, when the session is created, and travels with the session for its full lifetime via `*store.PipelineSession`. It is the binding's write branch when the constructor was given one, else `ri.AgentBranch()`. Everything downstream reads `sess.Branch` and NEVER reaches back into `ri.AgentBranch()`: `ContinueSessionForItem`, `Current`, `CurrentItem`, `handlePhase`, `completeSession`, `findHypothesisTransitions`, every `Strategy` method (`Plan`, `Apply`, `Render`, `OnPhaseAdvance`), the methodology loaders, and the shared discover step.

Consequence: a session continuing across an AgentBranch change (the user's live agent identity changed mid-flight) still operates on its ORIGINAL branch. A resumed session likewise runs on the branch it was created on.

ANTI-PATTERN: reading `ri.AgentBranch()` inside any phase handler, strategy method, or helper. It silently switches the session's branch mid-flight. The ONLY legitimate read is `Pipeline.opener`, which both session-creating entry points use (`StartSession`, and `StartOrResumeSession` for the knomit_review handler), immediately before the create call that persists it to `sess.Branch`.

ENFORCED STRUCTURALLY, not just by convention: `synthesize.Deps`, the bundle every strategy method receives, carries NO branch and NO session. A strategy can only obtain the branch from the `*store.PipelineSession` it is handed. Verify with `grep 'AgentBranch()' internal/synthesize/`: it should return exactly one non-comment hit, inside `Pipeline.opener`.
