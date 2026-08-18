---
kind: pragmatic
type: policy
domain: [integrations, hooks, bridge, invariant]
confidence: 0.95
sources: 1
entities: [hookPreInvocation, invocationNum, conversationId, markerPath, preInvocationInput, tools/bridge/antigravity/hook_pre_invocation.go]
refs: ['src://7b4887ce51d9/tools/bridge/antigravity/hook_pre_invocation.go@78b0809c89324fc4c08e29f5182118a107dc8a80:5bf9833172178bf25e3a68f1a51a1ee90720c704']
---
# A once-per-session hook guard must fail CLOSED on a missing field — decode the gating field as a pointer, because Go's zero value silently means "yes, fire"

A hook that fires on EVERY model call and gates itself down to once-per-conversation has two guards, and BOTH of them defaulted to "fire" when their input was missing.

The mechanism: `InvocationNum int` decoded an absent, renamed or restructured JSON field to Go's zero value `0`, which the hook read as "first invocation of the conversation". The second guard was a marker file keyed by `conversationId`; when that id was absent or failed the path-safety check, `markerPath` returned `""` and the code SKIPPED the marker check rather than refusing. Together they fail open: the entire knowledge-base context block is prepended to every single model call, for the life of the session, with nothing in the log distinguishing "marker written" from "marker impossible".

The field names were unverified guesses at a beta platform API. Piping `{"invocation_num":7,"conversation_id":"c9"}` — the same payload in snake_case — produced a full injection on both of two consecutive calls.

**The rule.** Any field a hook's fire/skip decision depends on must be decoded as a POINTER (or otherwise distinguish absent from zero), and every guard that cannot evaluate must skip rather than proceed. Staying silent costs one greeting; failing open costs unbounded context on every call.

**Foreseeable misreading:** this is NOT "validate hook input". Validation catches malformed values. The defect here is a well-formed payload whose gating field is simply ABSENT — the zero value is indistinguishable from a legitimate 0, which is exactly the value that means "fire".
