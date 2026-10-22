---
type: principle
domain: [synthesize, concurrency]
confidence: 0.95
sources: 0
entities: [Pipeline.opener, Pipeline.StartSession, Pipeline.StartOrResumeSession, Pipeline.ContinueSessionForItem, Pipeline.handlePhase, Pipeline.completeSession, Deps, Strategy, sess.Branch, ri.AgentBranch, store.PipelineSession]
motifs: [out-of-band-state]
refs: ['src://7b4887ce51d9/internal/corpus/synthesize/pipeline.go@0610c20083fcef9e84f42ab0311a14708bcb6193:a539c47c8a7376d5fb3fbf59d30742e2f1645e33', 'src://7b4887ce51d9/internal/corpus/synthesize/strategy.go@0610c20083fcef9e84f42ab0311a14708bcb6193:9291b0e763fd2b390854a7b56cfda5cfc5975d0d', 'src://7b4887ce51d9/internal/corpus/synthesize/hypothesize_strategy.go@0610c20083fcef9e84f42ab0311a14708bcb6193:40b96a34826042bc006b1cbc2b6b6edbbad0380a', 'kb://3ec012f5b4d2/kb/decisions/synthesize/one-engine-two-drivers/19c69b3f.md', 'kb://3ec012f5b4d2/kb/decisions/mcp/review/start-resumes-live-session/e4f26b91.md']
---
# sess.Branch is captured once, when a session is created (Pipeline.opener); downstream engine code NEVER reads ri.AgentBranch()

`sess.Branch` is captured ONCE, when the session is created, and travels with the session for its full lifetime via `*store.PipelineSession`. It is the binding's write branch when the constructor was given one, else `ri.AgentBranch()`. Everything downstream reads `sess.Branch` and NEVER reaches back into `ri.AgentBranch()`: `ContinueSessionForItem`, `Current`, `CurrentItem`, `handlePhase`, `completeSession`, `findHypothesisTransitions`, every `Strategy` method (`Plan`, `Apply`, `Render`, `OnPhaseAdvance`), the methodology loaders, and the shared discover step.

Consequence: a session continuing across an AgentBranch change (the user's live agent identity changed mid-flight) still operates on its ORIGINAL branch. A resumed session likewise runs on the branch it was created on.

ANTI-PATTERN: reading `ri.AgentBranch()` inside any phase handler, strategy method, or helper. It silently switches the session's branch mid-flight. The ONLY legitimate read is `Pipeline.opener`, which both session-creating entry points use (`StartSession`, and `StartOrResumeSession` for the knomit_review handler), immediately before the create call that persists it to `sess.Branch`.

ENFORCED STRUCTURALLY, not just by convention: `synthesize.Deps`, the bundle every strategy method receives, carries NO branch and NO session. A strategy can only obtain the branch from the `*store.PipelineSession` it is handed. Verify with `grep 'AgentBranch()' internal/corpus/synthesize/`: it should return exactly one non-comment hit, inside `Pipeline.opener`.
