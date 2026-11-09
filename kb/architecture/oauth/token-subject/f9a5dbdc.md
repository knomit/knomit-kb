---
type: observation
domain: [oauth, auth, identity]
confidence: 0.9
sources: 1
entities: [oauth.Verifier.Verify, oauth.TokenPrincipal, Issuer.Approve, FamilySpec, subjectRE, knomit oauth approve]
motifs: [name-is-not-identity]
refs: ['src://7b4887ce51d9/internal/edge/oauth/verify.go@b5d17520fb0a037b1d3e323189f615099f9a113e:9d410f091ff3bdafc08e5c49cfe6de454df170da', 'src://7b4887ce51d9/internal/edge/oauth/issuer.go@b5d17520fb0a037b1d3e323189f615099f9a113e:6a1e25102cf99b8ec4d041d469dfffb5397cb8e2', 'src://7b4887ce51d9/internal/edge/oauth/store.go@b5d17520fb0a037b1d3e323189f615099f9a113e:544b3d3f9835dddeb767571fee883c14a1b32473', 'src://7b4887ce51d9/internal/edge/web/oauth_pending_api.go@b5d17520fb0a037b1d3e323189f615099f9a113e:40aff3436539b0ef06574f8f5054d952db57fc2b', 'src://7b4887ce51d9/cmd/oauth.go@b5d17520fb0a037b1d3e323189f615099f9a113e:b9d101dbda64f507d3bc8a519d82ee80e97f9f1a']
---
# An OAuth token's principal is host:<subject>@token, and the subject names WHO the token acts as — the subject the approving operator typed — never the client app; the client id is a separate column

Checked at the PR #680 merge for the G2 ruling, which keys MCP ownership on the principal; re-verified at the PR #814 merge (b5d17520). Principal construction is now issuer.go's TokenPrincipal(subject), which Verifier.Verify calls.

- `Verifier.Verify` (internal/edge/oauth/verify.go) returns `TokenPrincipal(fam.Subject)` = `Principal{Kind: host, ID: subject, Via: token}`. The family row stores ClientID and Subject separately (oauth.Store, FamilySpec{ClientID, Subject, ...}); nothing copies one into the other.
- The subject is set ONLY at approval, `Issuer.Approve(ctx, id, subject, ...)` (internal/edge/oauth/issuer.go), which validates it against subjectRE. Its callers: the local approval API and `knomit oauth approve <id> --as <subject>` (cmd/oauth.go, oauth_pending_api.go) and the signed approve statement pass the subject the approving operator NAMES ("the token acts as host:<subject>@token"), required, no default. No path defaults it to the client_id.
- Refresh keeps the family's subject.

CONSEQUENCE: keying anything per principal (grants, MCP cursor/task ownership) keys it per approver-chosen identity: two clients approved for the same subject are ONE principal and share what is owned by it, and one client approved under two subjects is two.

UPDATE (auth P0): the GitHub identity-provider approval path (consent path 3, subject `github-<numeric id>`) was removed; its source files are gone and no longer cited. WHAT THIS DOES NOT MEAN: the subject is only as meaningful as the operator made it — it is a name, not a verified identity (an operator who approves two people as `laptop` makes them one principal). It is not bound to a repo/lens uid either; token→uid binding is a separate, later change.
