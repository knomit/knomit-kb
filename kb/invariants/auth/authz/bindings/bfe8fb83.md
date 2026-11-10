---
kind: pragmatic
type: policy
domain: [auth, authz, security]
confidence: 0.9
sources: 1
entities: [authz.Store.Bind, authz.ErrTokenBound, authz.TokenRoles, authz.RoleNode, authz.RoleGuest, checkBindable, admits, TestBind_RefusesOwner, TestBind_TokenOnlyOneRoleNeverAdmin, TestBind_AnonymousGuestOnly, TestBind_NodeRoleIsNeverBound, TestMaterialize_AnonymousNeverMoreThanGuest]
motifs: [default-deny-on-error, defence-in-depth]
refs: []
---
# authz.Store.Bind refuses: the owner (implicit, total), ANY binding of a node principal and the node role for any principal (a node's only role comes from its instance certificate), anonymous to anything but guest, a token to anything but reader/contributor/operator, a token that already has a role (ErrTokenBound), and a non-canonical principal; an OS principal may hold several roles (union)

Binding rules enforced by internal/foundation/authz Store.Bind (checkBindable plus the token count inside the transaction):
- The owner (authz.Owner(), compared by Kind+ID, label ignored) is never bound: it is implicit and holds everything.
- RoleNode is never bound to anyone, and a node principal (node:<fp>) is never bound to ANY role: a fleet node's only role is the node role its certificate carries, passed as an implicit role to Materializer.For (§13.1). These refusals go slightly beyond the plan's listed rules (the second added in P1 review).
- The principal must be in its canonical spelling (round-trips through ParsePrincipal).
- anonymous may be bound only to guest (§15.3).
- token:<id> may be bound only to one of TokenRoles() = reader, contributor, operator (never admin, guest or node), and only if it has NO binding yet, including the same role again: a token holds exactly one role; UnbindAll (revocation) frees it.
- OS principals (os:uid:<n> / os:sid:<SID>) may hold several roles; re-binding a held role is a no-op that does not move the version.
- Every binding needs a non-empty By.

Defence in depth: the Materializer independently drops roles a principal may not hold (admits): unknown roles, RoleNode on a non-node principal (stored OR passed as implicit), and anything but guest on anonymous (stored row, or MaterializerOptions.AnonymousRoles). So a hand-edited row binding anonymous to admin grants nothing beyond guest.

What this does NOT mean: Bind does not stop an OS account being bound to admin; nothing in P1 enforces the 'no change may leave nobody with auth:manage' rule, because the owner always holds auth:manage.
