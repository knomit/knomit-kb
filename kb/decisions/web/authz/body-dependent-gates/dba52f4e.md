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

A resolver peeks at most 1 MiB of the JSON body and puts the bytes back so the handler reads the whole body; a body that is not a JSON object resolves to the STRONGEST permission (fail closed; the handler answers 400 anyway).

Not covered by resolvers, on purpose: checks that depend on stored state rather than the request (handlers_origin_hal.go linkedOriginSpec and handleHALDeleteOrigin, origins/handlers.go session commit when the repo is linked). Those routes declare their base (repo:origin) and keep the in-handler check, converted to Policy.Has(secret:manage) in P4.

The equivalence test allows a resolver to add only repo:origin or secret:manage on top of its base: the operator and admin gates handlers run on the body today (httpx.Admits).
