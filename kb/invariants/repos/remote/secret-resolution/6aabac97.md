---
kind: pragmatic
type: policy
domain: [repos, remote, secrets, security, auth]
confidence: 0.9
sources: 1
entities: [resolveLinkedAuth, checkSecretFits, makeRemoteAuthFn, Manager.resolveSpecAuth, Manager.ResolveSpecAuth, authConfigFromSpec, ErrSecretOnPeer, ErrSecretTypeMismatch, control.ErrSecretHostNotAllowed, RemoteAuthConfig.SSHKeyPEM]
motifs: [fail-loud-not-lenient, single-choke-point]
refs: ['src://7b4887ce51d9/internal/corpus/repos/secret_auth.go@968ad8e4bd19903426eadbf9cc838ec0bd8203ab:20d98f2317a0077b74d72c0ec17dcfc23ee29ddc', 'src://7b4887ce51d9/internal/corpus/repos/sync.go@968ad8e4bd19903426eadbf9cc838ec0bd8203ab:2165acc725acb7823386fbd5e273acf957a645f1', 'src://7b4887ce51d9/internal/engine/store/remotesync/remote.go@968ad8e4bd19903426eadbf9cc838ec0bd8203ab:047965366967829b82c489adfc01d0d16b6364c8', 'src://7b4887ce51d9/internal/corpus/repos/secret_auth_test.go@968ad8e4bd19903426eadbf9cc838ec0bd8203ab:d73fe1b826b408320726c33167f8524597ba628c', 'src://7b4887ce51d9/internal/edge/web/api/origins/handlers.go@968ad8e4bd19903426eadbf9cc838ec0bd8203ab:a7525b52103c24f9f4244a5cb188a068342d144a', 'kb://3ec012f5b4d2/kb/invariants/repos/control-db/secrets/f05c8c88.md', 'kb://3ec012f5b4d2/kb/decisions/repos/access-secrets/host-bound-admin-only-linking/90bf32a8.md', 'kb://3ec012f5b4d2/kb/decisions/repos/access-secrets/plain-http-allowed/688ddfe0.md']
---
# A linked origin's credential is resolved from its stored secret ALONE at every git operation, through one resolver; a failure is an error naming the secret, never [remote], a pasted token or anonymous

Every git operation on an origin with a SecretID resolves the credential through resolveLinkedAuth (internal/corpus/repos/secret_auth.go): the reconcile loop and fleet publishes via makeRemoteAuthFn(fallback, keyPath, secrets) / Manager.remoteAuthFn(), and every spec-carried operation (clone, initialize, subscribe, probe, initialized probe, fleet move, origin-session test) via Manager.resolveSpecAuth / ResolveSpecAuth. resolveLinkedAuth reveals the secret (Secrets.Reveal, fail-loud), re-runs checkSecretFits (host on the secret's hosts list via control.HostAllowed AND type fits the URL scheme: token/basic need http(s), ssh_key needs ssh) and builds the auth config from the secret alone; only known_hosts comes from [remote]. An ssh_key secret is used in memory (RemoteAuthConfig.SSHKeyPEM, toml:"-"), never written to a file.

Consequences: (1) a missing, undecryptable, wrong-type or off-list secret makes the reconcile round a sync error (last_status error, last_error naming the secret by NAME) and no fetch/push is attempted, so the remote never sees the secret or an anonymous request; (2) replacing a secret's value takes effect on the next operation with nothing rebuilt; (3) a cert (fleet peer) origin with a SecretID is ErrSecretOnPeer.

Rule for new code: resolve a spec's credential only through resolveSpecAuth (ResolveSpecAuth outside the package; the origin-session test uses it for pasted and stored credentials alike). Manager.resolveAuthConfig (formerly the exported ResolveAuth) and authConfigFromSpec are unexported and called only inside resolveSpecAuth (TestAuthConfigFromSpecHasOneCaller). Misreading to avoid: the host check at link time does NOT make the resolution-time check redundant; a URL can change after linking (in memory or via attach), and the resolver re-checks every time.

Errors: a host off the list is control.ErrSecretHostNotAllowed (the repos package does not forward it; TestParents_NoForwardsIntoChildren), a type that does not fit the scheme is repos.ErrSecretTypeMismatch, a cert record with a link is repos.ErrSecretOnPeer.

Dialled URL: the loop checks the host against the stored URL, but go-git dials the git remote in .git/config. remoteAuthConfig therefore resolves a linked secret only while remotesync.Remote.GitURL (the git remote's URL, read in GetRemote) equals the stored URL; otherwise it is a sync error and the secret is not sent (TestLoop_GitRemoteElsewhereNeverSendsTheSecret).
