---
type: concept
domain: [fleet, architecture, guide, oauth, pki, auth]
confidence: 0.85
sources: 1
entities: [oauth.ApprovalStatement, Issuer.ApplySigned, knomit oauth approve --sign, '[oauth].addr', BearerMiddleware, root.crt, ErrNoFleetRoot]
motifs: [out-of-band-consent]
refs: ['kb://3ec012f5b4d2/kb/invariants/oauth/signed-approval-statement/c862c8b8.md', 'kb://3ec012f5b4d2/kb/decisions/pki/root-custody/plain-key-file/781b626f.md', 'kb://3ec012f5b4d2/kb/decisions/oauth/consent-path-1/7ac59e2a.md', 'kb://3ec012f5b4d2/kb/invariants/oauth/approval-local-principals-only/6d4a8514.md', 'kb://3ec012f5b4d2/kb/decisions/oauth/browser-approval-proof/8a0aafb0.md', 'kb://3ec012f5b4d2/kb/decisions/oauth/third-listener-401-is-narrow/2a2fc1fd.md', 'src://7b4887ce51d9/internal/oauth/signedapprove.go@72919aa0eb572e138ef4e6b0d44fac156adb116e:0d8b811797dbacdda5d4097d4e8a11bc6f782905', 'src://7b4887ce51d9/cmd/oauth_sign.go@72919aa0eb572e138ef4e6b0d44fac156adb116e:7ecf9e6e0998bac65ffc738f6cae833921d0dcb4', 'https://github.com/knomit/knomit/pull/519']
---
# Fleet guide — OAuth approvals: an MCP client's OAuth request is PARKED and approved out of band by a local principal (socket CLI or the same-origin web UI), or by a statement signed with the FLEET ROOT key that an instance verifies against its installed root.crt; the bearer token it yields is judged only on the separate [oauth].addr listener and never reaches /git

Part of the fleet guide (kb/architecture/fleet/guide/). Describes merged dev at c9c599de, except how --sign reads the root key (72919aa0, PR #519).

WHY IT IS FLEET MACHINERY. Remote agents reach an instance's MCP endpoint with OAuth bearer tokens. Approving those requests is the one place where the FLEET ROOT key, besides signing certificates, is used at runtime.

FLOW. /oauth/authorize parks the request and shows only its id. An approval is accepted from (a) a local-listener principal (the unix socket, or the named pipe on Windows: `knomit oauth approve <id> --as <subject> [--scopes ...]` / `deny`), (b) the loopback web UI holding admin with the full same-origin browser proof, or (c) a SIGNED approval: `knomit oauth approve --sign` on the fleet root machine reads the plain <dir>/root.key (pki.LoadRoot; no keychain, no passphrase, no prompt for the key; PR #519) and, after printing the request, asks a y/N consent (`--yes` skips it) before it signs ONE canonical, domain-tagged statement (first line knomit-oauth-approve/v1 or knomit-oauth-deny/v1, then the instance fingerprint, pending id, request digest, [subject, sorted scopes,] expiry) with the fleet root key; the instance REBUILDS those bytes from validated fields and verifies them against the root.crt installed in [tls].dir (no root installed = ErrNoFleetRoot), refusing replays. Never from a certificate or bearer principal, whatever it holds. Bearer tokens are judged only on the third listener [oauth].addr (its BearerMiddleware is the only 401 knomit sends); that listener mounts no /git, so a token can never push.

CONSEQUENCES. An operator holding the fleet root can approve OAuth requests on any instance that installed that root, without a session on it. Whoever holds the fleet root directory (the plain root.key) holds this power: there is no export-root/import-root any more, so it moves only if the operator copies the directory by hand (and LoadRoot refuses a copy that landed with a wider mode).

MISREADING. 'An operator certificate can approve OAuth': no; approvals come only from local principals, the browser-proofed loopback admin, or a root-signed statement.
