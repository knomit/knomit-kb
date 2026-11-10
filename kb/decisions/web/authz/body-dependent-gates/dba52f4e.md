---
type: observation
domain: [web, authz, security, decisions]
confidence: 0.85
sources: 1
entities: [routeRule.Resolve, routeRule.Possible, withSecret, resolveRepoCreate, resolveArchivedRestore, resolveGit, peekJSON, admitSecretID, httpx.Admits, handlerEscalations]
motifs: [fail-closed-parsing, ordered-permission-set]
refs: []
---
# Body-dependent route gates (inventory G9) are route-registry RESOLVERS returning one of an ordered Possible set (weakest first), never a union: under the fixed seed every stronger entry (secret:manage, admin-only) implies the weaker ones; state-dependent admin checks (a repo whose origin is LINKED to a stored secret) stay in-handler and become Policy.Has(secret:manage) at the flip

route_rules.go resolvers: POST /repos (origin -> repo:origin, origin.secret_id -> secret:manage, else repo:manage); POST /archived/{id}/restore (secret_id -> secret:manage, pasted credential -> repo:origin, else repo:manage); POST /repos:probe-origin, /repos:probe-initialized, POST /repos/{repo}/origin-sessions/ (secret_id -> secret:manage, else repo:origin); PUT /repos/{repo}/origin/credential (secret_id INCLUDING null, which unlinks -> secret:manage, else repo:origin); POST /fleet/create, /fleet/join, /fleet/join:check, /fleet/follow (secret_id or origin.secret_id -> secret:manage, else fleet:manage); /git/* (receive-pack either leg, via isGitPush -> repo:push, else kb:read).

Each body resolver lives NEXT TO ITS HANDLER (internal/edge/web/api/manage/permissions.go: ResolveRepoCreate, ResolveArchivedRestore, ResolveProbeOrigin, ResolveProbeInitialized, ResolveOriginCredential, ResolveFleetCreate, ResolveFleetJoin, ResolveFleetFollow; api/origins/permissions.go: ResolveCreateSession). It decodes the body into the HANDLER'S OWN request struct with httpx.PeekJSON, which uses DecodeJSON's decoder (one JSON value, no trailing data), puts the bytes back, and applies the handler's own test. So the resolver sees exactly what the handler sees: field names match case-insensitively, a duplicate key decodes into the same pointer, and {"origin":{}} is an origin (repo:origin, as the handler's operator gate). An undecodable or oversized body (over the handler's own limit, else 1 MiB) resolves to the STRONGEST permission.

Never decode a body for a permission decision into a generic map or a struct of your own: P1 review proved that {"origin":{"url":..,"Secret_ID":"s1"}}, a duplicate "origin" key, {"SECRET_ID":"s"} on the credential route and {"Secret_id":"s"} on origin sessions each resolved to repo:origin while the handler linked the secret.

Not covered by resolvers, on purpose: checks that depend on stored state rather than the request (handlers_origin_hal.go linkedOriginSpec and handleHALDeleteOrigin, origins/handlers.go session commit when the repo is linked). Those routes declare their base (repo:origin) and keep the in-handler check, converted to Policy.Has(secret:manage) in P4.

P4 MUST: (1) keep every in-handler operator/admin check (httpx.Admits) as Policy.Has(repo:origin|secret:manage) — the resolvers choose only the route-level permission and are not the authority; (2) check Policy.Has(Possible[0]) BEFORE calling Resolve, so a caller who lacks even the weakest permission (anonymous) is refused without the server reading or buffering its body.

The equivalence test allows a resolver to add only repo:origin or secret:manage on top of its base: the operator and admin gates handlers run on the body today (httpx.Admits).
