---
type: observation
domain: [oauth, auth, identity]
confidence: 0.9
sources: 1
entities: [oauth.Verifier.Verify, oauth.TokenPrincipal, Issuer.Approve, FamilySpec, subjectRE, knomit oauth approve, idpFlow]
motifs: [name-is-not-identity]
refs: ['src://7b4887ce51d9/internal/edge/oauth/verify.go@16fb71bc530e47acd2568b59dbe9891cb229ecbb:9d410f091ff3bdafc08e5c49cfe6de454df170da', 'src://7b4887ce51d9/internal/edge/oauth/issuer.go@16fb71bc530e47acd2568b59dbe9891cb229ecbb:4f0d72d2a6faf0d5aff405a270aa75941be8d907', 'src://7b4887ce51d9/internal/edge/oauth/idpflow.go@16fb71bc530e47acd2568b59dbe9891cb229ecbb:c22934c85e9bfb6c992b192aa8a537e5aee0058d', 'src://7b4887ce51d9/internal/edge/oauth/store.go@16fb71bc530e47acd2568b59dbe9891cb229ecbb:5e0f97cf42d5191e7baefc4789a29a86aeed4642', 'src://7b4887ce51d9/cmd/oauth.go@16fb71bc530e47acd2568b59dbe9891cb229ecbb:9b724d9fef1d17d45c825475596d79d18069c7bb']
---
# An OAuth token's principal is host:<subject>@token, and the subject names WHO the token acts as — the identity-provider account (github-<numeric id>) or the subject the approving operator typed — never the client app; the client id is a separate column

Checked at fix/mcp-token-audience (16fb71bc) for the G2 ruling, which keys MCP ownership on the principal.

- `Verifier.Verify` (internal/edge/oauth/verify.go) returns `TokenPrincipal(fam.Subject)` = `Principal{Kind: host, ID: subject, Via: token}`. The family row stores ClientID and Subject separately (oauth.Store, FamilySpec{ClientID, Subject, ...}); nothing copies one into the other.
- The subject is set ONLY at approval, `Issuer.Approve(ctx, id, subject, ...)` (internal/edge/oauth/issuer.go), which validates it against subjectRE. Its callers: the identity-provider flow passes `s.subject.ID`, the provider's account id `github-<numeric id>` (idpflow.go) — the person who signed in; the local approval API and `knomit oauth approve <id> --as <subject>` (cmd/oauth.go, oauth_pending_api.go) and the signed approve statement pass the subject the approving operator NAMES ("the token acts as host:<subject>@token"), required, no default. No path defaults it to the client_id.
- Refresh keeps the family's subject.

CONSEQUENCE: keying anything per principal (grants, MCP cursor/task ownership) keys it per person/approver-chosen identity: two clients approved for the same subject are ONE principal and share what is owned by it, and one client approved under two subjects is two.

WHAT THIS DOES NOT MEAN: for manual approvals the subject is only as meaningful as the operator made it — it is a name, not a verified identity (an operator who approves two people as `laptop` makes them one principal). It is not bound to a repo/lens uid either; token→uid binding is a separate, later change.
