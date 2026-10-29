---
type: principle
domain: [synthesize, concurrency]
confidence: 0.95
sources: 0
entities: [Pipeline.opener, Pipeline.StartSession, Pipeline.StartOrResumeSession, Pipeline.ContinueSessionForItem, Pipeline.handlePhase, Pipeline.completeSession, Deps, Strategy, sess.Branch, ri.AgentBranch, store.PipelineSession]
motifs: [out-of-band-state]
refs: ['src://7b4887ce51d9/internal/corpus/synthesize/pipeline.go@184fc2da6e7d44fbe80cd34aca39e5754e5178b0:b958314ae33c131c74c6b30b00a658786b4b92a3', 'src://7b4887ce51d9/internal/corpus/synthesize/step/strategy.go@184fc2da6e7d44fbe80cd34aca39e5754e5178b0:c6f9007259a61ebc75e9cad4a08118e20f1786f8', 'src://7b4887ce51d9/internal/corpus/synthesize/hypothesize_strategy.go@184fc2da6e7d44fbe80cd34aca39e5754e5178b0:0e585c2782867dedaeaa06fb904d1ffbd6f02229', 'src://7b4887ce51d9/internal/corpus/synthesize/session_branch_test.go@184fc2da6e7d44fbe80cd34aca39e5754e5178b0:4f67898e0c5bfaba24d7d50cc566384b4daa335b', 'src://7b4887ce51d9/internal/edge/mcp/hypothesize.go@184fc2da6e7d44fbe80cd34aca39e5754e5178b0:2ace69dca69aca086703441cede8077993235e32', 'src://7b4887ce51d9/internal/edge/mcp/hypothesize_branch_test.go@184fc2da6e7d44fbe80cd34aca39e5754e5178b0:81e3d430af239fa2a1c65825f98b73a21a3d869f', 'kb://3ec012f5b4d2/kb/decisions/synthesize/one-engine-two-drivers/19c69b3f.md', 'kb://3ec012f5b4d2/kb/decisions/mcp/review/start-resumes-live-session/e4f26b91.md']
---
# sess.Branch is captured once, when a session is created (Pipeline.opener); downstream engine code NEVER reads ri.AgentBranch()

`sess.Branch` is captured ONCE, when the session is created, and travels with the session for its full lifetime via `*store.PipelineSession`. It is the binding's write branch when the constructor was given one, else `ri.AgentBranch()`. Everything downstream reads `sess.Branch` and NEVER reaches back into `ri.AgentBranch()`: `ContinueSessionForItem`, `Current`, `CurrentItem`, `handlePhase`, `completeSession`, `findHypothesisTransitions`, every `Strategy` method (`Plan`, `Apply`, `Render`, `OnPhaseAdvance`), the methodology loaders, and the shared discover step.

Consequence: a session continuing across an AgentBranch change (the user's live agent identity changed mid-flight) still operates on its ORIGINAL branch. A resumed session likewise runs on the branch it was created on.

ANTI-PATTERN: reading `ri.AgentBranch()` inside any phase handler, strategy method, or helper. It silently switches the session's branch mid-flight. The ONLY legitimate read is `Pipeline.opener`, which both session-creating entry points use (`StartSession`, and `StartOrResumeSession` for the knomit_review handler), immediately before the create call that persists it to `sess.Branch`.

ENFORCED STRUCTURALLY, not just by convention: `synthesize.Deps`, the bundle every strategy method receives, carries NO branch and NO session. A strategy can only obtain the branch from the `*store.PipelineSession` it is handed. Verify with `grep 'AgentBranch()' internal/corpus/synthesize/`: it should return exactly one non-comment hit, inside `Pipeline.opener`.

AMENDED (PR #680, review N2): a CONTINUATION by session_id is now also refused, reading exactly as "session not found", when the caller names a branch (`Pipeline.branch`, which knomit_review and knomit_hypothesize set to the binding's write branch on every path: start, current and continue) that differs from `sess.Branch` (`Pipeline.activeSession`). A session_id is a secret, not a reach, and a caller confined to some branches (a lens) must not continue a session on another. It reads no new ri.AgentBranch(): an in-process caller with no branch is not checked. Consequence for the line above: an MCP caller whose binding's write branch changed mid-flight (agent identity change) can no longer continue its old session through the new binding; it starts a new one. Pinned by TestActiveSession_RefusesASessionOnAnotherBranch (engine) and TestHypothesize_SessionIDContinuesOnlyOnItsBranch (MCP, both continuation paths).
