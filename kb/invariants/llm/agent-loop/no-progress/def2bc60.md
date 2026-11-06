---
kind: pragmatic
type: policy
domain: [llm, chat, tools]
confidence: 0.85
sources: 1
entities: [Loop.ShortCircuit, repeatGuard, NoProgressRepeats, ErrNoProgress, NoProgressNote_DRAFT, NoProgressMessage_DRAFT, mcp.RepeatAnswerable, TestLoop_IdenticalSuccessfulCallIsShortCircuitedThenEndsTheTurn, TestRepeatAnswerable, TestChat_RepeatedSuccessfulCallEndsTheTurn]
motifs: [runaway-retry-bound]
refs: ['kb://3ec012f5b4d2/kb/invariants/llm/agent-loop/repeat-failure/c09ba08f.md', 'kb://3ec012f5b4d2/kb/invariants/llm/agent-loop/turn-drop/da80d551.md', 'src://7b4887ce51d9/internal/engine/llm/agent.go@a6e89ef1bbe9850052a5b0f5d1ee5a5baf6b2b4f:39f955e5fd8e8a1b440a37f7b7771835d6e392ce', 'src://7b4887ce51d9/internal/engine/llm/noprogress_test.go@a6e89ef1bbe9850052a5b0f5d1ee5a5baf6b2b4f:9ef92c5685896481b5a3b1f50101c3ea598da351', 'src://7b4887ce51d9/internal/edge/mcp/tool_effects.go@a6e89ef1bbe9850052a5b0f5d1ee5a5baf6b2b4f:2e79d7e9a00aa4ceb9ce503d1a8519a5e14ccc02']
---
# The agent loop answers a byte-identical call that just succeeded from its earlier result with a note, and ends the turn on the next identical call; only the call immediately before counts, and Loop.ShortCircuit (nil = guard off) says which calls may be answered that way

Companion of kb/invariants/llm/agent-loop/repeat-failure/c09ba08f.md, which only counts FAILURES. Live acceptance saw a 4B model call knomit_bind with identical arguments 23 times in one turn; each call succeeded, so the failure guard never tripped. Loop.runCalls now remembers the last call that RAN and SUCCEEDED when Loop.ShortCircuit(tool, args) allows it. The next call with the same tool and byte-identical arguments, with nothing in between, is NOT run and is not sent to the approval gate: it is answered with the earlier result plus NoProgressNote_DRAFT, as a success. The next identical call after that (the third in a row) ends the turn: it and the rest of the reply are answered SkippedResult so every tool_use stays paired (da80d551), and the turn error wraps ErrNoProgress with NoProgressMessage_DRAFT. 23 identical binds therefore give 1 real call, 1 answered repeat, then the end. NoProgressRepeats = 1 is a policy, not a measurement.

Reset: any different call (other arguments or tool), a failure, a decline, a window-handled call (load_tools or a not-loaded call) or invalid JSON forgets the remembered call, because the state a repeat would see may have changed (a bind changes what a later query returns). A failing call is never remembered; failures stay the repeat-failure guard's. Only calls in a single stretch are caught: A,B,A,B alternation is not (MaxCalls still bounds it).

Which calls: the chat passes mcp.RepeatAnswerable for EVERY provider, Anthropic included. It says yes for reads, for knomit_bind (state-changing but idempotent for identical arguments) and for writes (an identical learn/update/retract after the first succeeded must not run twice silently; the earlier result tells the model it was done), and no for a call carrying a cursor (it consumes the page it names), for every knomit_review and knomit_hypothesize call (session steps: page advances, current may take the next step), and for any tool or action the Replay table does not know or arguments that do not decode. It is not mcp.Replayable (that asks whether a result dropped from history may be offered back).

Does NOT mean: a turn that never repeats changes by a byte (Anthropic request bytes are untouched when nothing repeats); the guard sees only successes of the call just before; a Loop with ShortCircuit nil has no guard. The note and the turn-end sentence are _DRAFT constants awaiting the user's approval; the wording lives in agent.go.
