---
type: principle
domain: [synthesize, concurrency]
confidence: 0.95
sources: 0
entities: [Pipeline.StartSession, Pipeline.ContinueSessionForItem, Pipeline.handlePhase, Pipeline.completeSession, Deps, Strategy, sess.Branch, ri.AgentBranch, store.PipelineSession]
motifs: [out-of-band-state]
refs: ['src://knomit/internal/synthesize/pipeline.go@6356eab8', 'src://knomit/internal/synthesize/strategy.go@6356eab8', 'src://knomit/internal/synthesize/hypothesize_strategy.go@6356eab8', kb/decisions/synthesize/one-engine-two-drivers/19c69b3f.md]
---
# sess.Branch is captured once at StartSession; downstream engine code NEVER reads ri.AgentBranch()

`sess.Branch` is captured from `ri.AgentBranch()` ONCE at StartSession and travels with the session for its full lifetime via `*store.PipelineSession`. Everything downstream — `ContinueSessionForItem`, `handlePhase`, `completeSession`, `findHypothesisTransitions`, every `Strategy` method (`Plan`, `Apply`, `Render`, `OnPhaseAdvance`), the methodology loaders, and the shared discover step — reads `sess.Branch` and NEVER reaches back into `ri.AgentBranch()`.

Consequence: a session continuing across an AgentBranch change (the user's live agent identity changed mid-flight) still operates on its ORIGINAL branch.

ANTI-PATTERN: reading `ri.AgentBranch()` inside any phase handler, strategy method, or helper — it silently switches the session's branch mid-flight. The ONLY legitimate read is in `Pipeline.StartSession` itself, immediately followed by `CreatePipelineSession(ctx, strategy.Tool(), branch)`, which persists it to `sess.Branch`.

HOW IT IS NOW ENFORCED STRUCTURALLY, not just by convention: `synthesize.Deps` — the bundle every strategy method receives — deliberately carries NO branch and NO session. A strategy can only obtain the branch from the `*store.PipelineSession` it is handed, so there is no `ri.AgentBranch()` reachable below StartSession. Verify with `grep 'AgentBranch()' internal/synthesize/`: it should return exactly one non-comment hit, inside StartSession.

HISTORY — this fact previously named `Reviewer.handleWorkPhase` and `Reviewer.handleReflectPhase`, which no longer exist: the P1.1 engine extraction collapsed them into `Pipeline.handlePhase`, and the subject is now `Pipeline` rather than `Reviewer` (which survives only as a facade). The extraction also FIXED a live violation: hypothesize had reimplemented the session lifecycle inside internal/mcp and threaded the live `agentBranch` through its continue and complete paths instead of reading `sess.Branch`, so a mid-session identity change moved its writes. That violation existed for months without a test catching it — treat the grep above as the check, not the absence of failures.
