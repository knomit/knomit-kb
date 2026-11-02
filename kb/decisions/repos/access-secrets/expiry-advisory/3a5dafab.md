---
type: observation
domain: []
confidence: 0.95
sources: 1
entities: [secrets, expiry, status]
refs: ['file:///home/pba/data/knomit/knomit/.claude/plans/2026-11-02-access-secrets-design.md']
---
# Secret expiry is advisory: sync keeps using an expired secret and only the UI flags it

Describes the agreed design for feat/access-secrets, not shipped behaviour.

Options considered: stop using a secret once its expiry date passes; keep using it and only warn.

Rationale: the recorded date may be wrong and the forge is the authority on whether a token works. Stopping sync because a date passed would be a failure even if everything else worked.

Choice: sync keeps using an expired secret. The server computes a status (ok, expiring, expired, none); expiring means within 14 days. The UI flags the secret and every repo linked to it. No notifications outside the UI (no email, log alerts, MCP pushes).

Non-scope: expiry does not gate, disable or deprioritise sync; no background calls to forges to verify tokens.
