---
kind: pragmatic
type: policy
domain: [integrations, hooks, bridge, invariant]
confidence: 0.95
sources: 2
entities: [hookPreInvocation, invocationNum, conversationId, markerPath, preInvocationInput, tools/bridge/antigravity/hook_pre_invocation.go]
motifs: [absence-encodes-value]
refs: ['src://7b4887ce51d9/tools/bridge/antigravity/hook_pre_invocation.go@687fb378dcb0e064498cd24b0e0baa3786567a34:23720a6705776978d33509ef6d6cc5f635cf7b72']
---
# A once-per-session hook guard must fail CLOSED on a missing field — decode the gating field as a pointer, because Go's zero value silently means "yes, fire"

A hook that fires on EVERY model call and gates itself down to once-per-conversation has two guards, and BOTH of them defaulted to "fire" when their input was missing.

The mechanism: `InvocationNum int` decoded an absent, renamed or restructured JSON field to Go's zero value `0`, which the hook read as "first invocation of the conversation". The second guard was a marker file keyed by `conversationId`; when that id was unusable, `markerPath` returned `""` and the code SKIPPED the marker check rather than refusing. Together they fail open: the entire knowledge-base context block is prepended to every single model call, for the life of the session, with nothing in the log distinguishing "marker written" from "marker impossible".

The field names were unverified guesses at a beta platform API. Piping `{"invocation_num":7,"conversation_id":"c9"}` — the same payload in snake_case — produced a full injection on both of two consecutive calls.

**The rule.** Any field a hook's fire/skip decision depends on must be decoded as a POINTER (or otherwise distinguish absent from zero), and every guard that cannot evaluate must skip rather than proceed. Staying silent costs one greeting; failing open costs unbounded context on every call.

**The counter-trap, found while closing this one.** "Cannot evaluate" must mean genuinely cannot — not merely "the value looks unfamiliar". The first fix made `markerPath` reject any `conversationId` outside `[A-Za-z0-9_-]`, so a timestamped id such as `conv_2026-08-18T10:30:00Z` — a shape this beta API may well use — would have skipped EVERY conversation for EVERY user, with one line in the log to show for it. Fail-closed on an input you genuinely cannot use costs one greeting; fail-closed on an input you could have handled costs the whole feature. The id is now HASHED (`sha256` → hex) rather than validated, which keeps the write inside the cache directory for any id at all; "unusable" narrows to the empty id, which still skips. Where a value can be made safe, make it safe, and reserve the skip for where it cannot.

**Foreseeable misreading:** this is NOT "validate hook input". Validation catches malformed values. The defect on the first guard is a well-formed payload whose gating field is simply ABSENT — the zero value is indistinguishable from a legitimate 0, which is exactly the value that means "fire". And on the second guard validation was the actively wrong remedy, per the counter-trap above.
