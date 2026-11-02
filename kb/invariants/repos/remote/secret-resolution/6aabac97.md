---
kind: pragmatic
type: policy
domain: [repos, remote, secrets, security, auth]
confidence: 0.9
sources: 1
entities: [resolveLinkedAuth, checkSecretFits, makeRemoteAuthFn, Manager.resolveSpecAuth, Manager.ResolveSpecAuth, authConfigFromSpec, ErrSecretOnPeer, ErrSecretTypeMismatch, ErrSecretHostNotAllowed, RemoteAuthConfig.SSHKeyPEM]
motifs: [fail-loud-not-lenient, single-choke-point]
refs: ['src://7b4887ce51d9/internal/corpus/repos/secret_auth.go@fbfb495620bb3f4e8c7c2296908597383379725a:a7f5de26f66c893e7709b15f2732bd48e32fa5a9', 'src://7b4887ce51d9/internal/corpus/repos/sync.go@fbfb495620bb3f4e8c7c2296908597383379725a:81b9d59fec18dacc3933c6d5b470dac9ac52b4eb', 'src://7b4887ce51d9/internal/corpus/repos/secret_auth_test.go@b8efc0fa815766170e6306a461edc190c7516acc:390429461a08e5b51932fbd1d6a391a47065444a', 'kb://3ec012f5b4d2/kb/invariants/repos/control-db/secrets/f05c8c88.md', 'kb://3ec012f5b4d2/kb/decisions/repos/access-secrets/host-bound-admin-only-linking/90bf32a8.md']
---
# A linked origin's credential is resolved from its stored secret ALONE at every git operation, through one resolver; a failure is an error naming the secret, never [remote], a pasted token or anonymous

Every git operation on an origin with a SecretID resolves the credential through resolveLinkedAuth (internal/corpus/repos/secret_auth.go): the reconcile loop and fleet publishes via makeRemoteAuthFn(fallback, keyPath, secrets) / Manager.remoteAuthFn(), and every spec-carried operation (clone, initialize, subscribe, probe, initialized probe, fleet move, origin-session test) via Manager.resolveSpecAuth / ResolveSpecAuth. resolveLinkedAuth reveals the secret (Secrets.Reveal, fail-loud), re-runs checkSecretFits (host on the secret's hosts list via control.HostAllowed AND type fits the URL scheme: token/basic need http(s), ssh_key needs ssh) and builds the auth config from the secret alone; only known_hosts comes from [remote]. An ssh_key secret is used in memory (RemoteAuthConfig.SSHKeyPEM, toml:"-"), never written to a file.

Consequences: (1) a missing, undecryptable, wrong-type or off-list secret makes the reconcile round a sync error (last_status error, last_error naming the secret by NAME) and no fetch/push is attempted, so the remote never sees the secret or an anonymous request; (2) replacing a secret's value takes effect on the next operation with nothing rebuilt; (3) a cert (fleet peer) origin with a SecretID is ErrSecretOnPeer.

Rule for new code: never call authConfigFromSpec or Manager.ResolveAuth with a spec that may carry SecretID; go through resolveSpecAuth (TestAuthConfigFromSpecHasOneCaller enforces that authConfigFromSpec has exactly one caller). Misreading to avoid: the host check at link time does NOT make the resolution-time check redundant; a URL can change after linking (in memory or via attach), and the resolver re-checks every time.
