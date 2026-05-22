---
type: principle
domain: [synthesize, concurrency]
confidence: 0.95
sources: 0
entities: [Reviewer.StartSession, Reviewer.ContinueSession, sess.Branch, ri.AgentBranch]
refs: ['src://knomit/internal/synthesize/review.go@307b67d']
---
# sess.Branch is captured once at StartSession; downstream methods NEVER read ri.AgentBranch()

sess.Branch is captured from ri.AgentBranch() ONCE at StartSession time and travels with the session for its full lifetime via *store.PipelineSession. All downstream Reviewer methods (ContinueSession, handleWorkPhase, handleReflectPhase, completeSession, findHypothesisTransitions, etc.) read sess.Branch — they NEVER reach back into ri.AgentBranch(). This means a session continuing across an AgentBranch change (e.g. the user's live agent identity changed mid-flight) still operates on its ORIGINAL branch. Anti-pattern: reading r.ri.AgentBranch() inside any phase handler or helper — it would silently switch the session's branch. The only place ri.AgentBranch() is legitimately read is StartSession itself, immediately followed by pipelineIdx.CreatePipelineSession(ctx, 'review', branch) which persists it to sess.Branch.
