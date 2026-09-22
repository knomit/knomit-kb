---
type: observation
domain: [auth, security, mcp, bridge, fleet, planning]
confidence: 0.95
sources: 1
entities: [internal/auth, auth.Principal, auth.Allowed, AuthMiddleware, writeGate, gatePermission, cmd/serve.go, knomit.sock, '[auth].require', '[auth].loopback_default', F19, 'PR #243']
motifs: [cheapest-credential-first, type-before-producers]
refs: ['src://7b4887ce51d9/internal/auth/principal.go@cec7c5c10b4a752dadbb34a45cb749427923ba8d:f8dbcdb73210aaaf45c5534116b5193ed9e64433', 'src://7b4887ce51d9/internal/web/auth_middleware.go@cec7c5c10b4a752dadbb34a45cb749427923ba8d:8dc48207941b3ccbcadccb284305e61238fcf387', 'src://7b4887ce51d9/internal/auth/peercred.go@cec7c5c10b4a752dadbb34a45cb749427923ba8d:e0ad74261e79661917d9241f368f9da2ad3bd18c', 'src://7b4887ce51d9/cmd/serve.go@cec7c5c10b4a752dadbb34a45cb749427923ba8d:dc85b02faae31208f3a332d4a032ae196958efe8', 'kb://3ec012f5b4d2/kb/invariants/auth/one-principal-one-check/8d5cc79d.md', 'kb://3ec012f5b4d2/kb/decisions/auth/socket-peer-creds-for-local-bridge/e6369a31.md', 'kb://3ec012f5b4d2/kb/gotchas/desktop/separate-server-boot-path/cd92bf55.md', 'kb://3ec012f5b4d2/kb/invariants/auth/local-listener-single-site/9cf869d3.md', 'kb://3ec012f5b4d2/kb/decisions/auth/socket-liveness-flock/b27c556d.md', 'https://github.com/knomit/knomit/pull/243', 'https://github.com/knomit/knomit/pull/250', 'https://github.com/knomit/knomit/issues/248']
---
# Authentication ships in four phases with the principal layer and the unix socket FIRST, certificates second, OAuth third, fleet policy last — because phase 1 covers every existing user with no new credential and the other three each need one

**Context.** F19 revision 2 (the authentication proposal, `.claude/future/fleet/proposals/F19-pki-x509-mtls.md`, not tracked in git) replaced a three-tier X.509 design with a hybrid: client certificates for knomit-to-knomit, OAuth-issued bearer tokens for MCP hosts that cannot hold a certificate, and a unix domain socket with kernel peer credentials for the local bridge, all producing one `auth.Principal` checked by one `auth.Allowed`. The work was too large for one PR and had to be ordered.

**Options considered (put to the maintainer 2026-09-22):**
1. Core, socket, PKI, OAuth — principal type + permission layer + grants table + unix socket with peer creds + bridge dials it; then `internal/pki` and mTLS between instances; then the OAuth issuer and bearer tokens; then master-signed fleet policy in `.knomit/policy/`.
2. PKI first — certificates and mTLS between instances before the principal layer, socket and OAuth.
3. OAuth first — token issuer and bearer verification so remote hosts work first.

**Rationale.** Option 1 is the only phase that is useful to every existing single-machine user and that introduces NO new credential: the socket's credential is the 0700 data root plus the kernel's uid, so nothing is stored, rotated or leaked, and the self-declared bridge pid becomes verified. It also builds the type every later phase produces, so certificates and tokens slot in without touching enforcement. Option 2 gets the fleet edge earliest but leaves local bridges self-declared longer and builds an issuance/CRL lifecycle before anything consumes it. Option 3 puts the most new surface (an authorization server) first, for the fewest users. `[auth].require` defaults false and anonymous loopback keeps full local rights, so phase 1 changes nobody's day on upgrade.

**The choice.** Option 1. Phase 1 landed as PR #243 (merge cec7c5c1): `internal/auth`, `AuthMiddleware` + `writeGate`, MCP `permissionFilter` + `gatePermission`, `grants` table, `client_session_peers`, socket listener in `cmd/serve.go`, bridge per-dial socket preference with TCP fallback. Phase 1c (PR #250, merge a8e864ad) closed the gap that phase 1 had opened the socket only in `cmd/serve.go`: `auth.ListenLocal` is now the single site, called from both `cmd/serve.go` and the desktop's `bootServer`, with a flock beside the socket so a second knomit process never steals a live instance's socket. Phases 2–4 are proposals only.

**Non-scope.** This orders the WORK; it does not say the socket is the preferred credential in general. Certificates remain the credential between instances and tokens for hosts; the socket is only for a bridge on the same machine as its server. Nor does phase 1 enforce any permission but `write`: `read`, `push:own`, `merge:main`, `operator`, `admin` exist as vocabulary and grant rows with no enforcement point until F11/F12/phase 4.
