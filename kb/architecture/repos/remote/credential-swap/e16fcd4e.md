---
type: observation
domain: [repos, remote, credentials, web]
confidence: 0.9
sources: 1
entities: [Manager.ChangeOriginCredential, CredentialChange, ErrNoOrigin, ErrLinkedRepo, ErrInvalidCredential, Origins.SetCredential, handleSetOriginCredential, 'PUT /repos/{repo}/origin/credential']
motifs: [compare-and-set-write]
refs: ['src://7b4887ce51d9/internal/corpus/repos/credential.go@0a591714344c5464f788c1e72f1113b8d4cd5c5e:e2350b4b434bea253829822add3efa246da14521', 'src://7b4887ce51d9/internal/edge/web/api/manage/handlers_origin_credential.go@0a591714344c5464f788c1e72f1113b8d4cd5c5e:d2bdca6a4145b2d5d832d3d3bd7c4f1d51db58b5', 'kb://3ec012f5b4d2/kb/invariants/repos/control-db/secret-link/845c96d6.md', 'kb://3ec012f5b4d2/kb/invariants/repos/remote/attach-identity/545ae852.md']
---
# PUT /repos/{repo}/origin/credential (Manager.ChangeOriginCredential) changes only the credential: compare-and-set on the stored URL, re-read, inject into the running store, credentialChanged; no probe, guard, git-remote rewrite or Sync restart

Manager.ChangeOriginCredential(ctx, ri, CredentialChange{AuthMethod, AuthToken, SecretID}, unlink) is the light credential path. It reads the stored origin (control.Origins.Get) and refuses: no origin (ErrNoOrigin, 409 "No origin"), a fleet peer origin with auth_method cert (ErrPeerCredential, 400), a pasted credential on a linked repo unless unlink is set (ErrLinkedRepo, 409, naming the unlink command), and a malformed credential or one whose method does not fit the URL scheme (ErrInvalidCredential, 400; wraps MethodMismatch, token needs a token, basic needs user:password, ssh and none take no token, cert is never an input). It then writes with Origins.SetCredential(uid, stored.URL, ...), a compare-and-set: if an attach re-pointed the origin meanwhile, control.ErrOriginChanged (409 "Origin changed") and nothing is written, in control.db or in memory. On success it re-reads the row, calls svc.Remote().SetOrigin with the re-read URL, branch, method and token (skipped with a debug line when the store is not open, since opening injects the control.db row), and calls ri.credentialChanged().

The endpoint is behind operator (like PUT /origin), takes {auth_method, token?, user?, password?} assembled with httpx.AssembleAuthToken, and in this PR refuses a secret_id key with 400 (linking arrives with operation-time secret resolution). manage.Mount already takes an admin gate parameter for that.

What this does NOT mean: it is not a way to change the URL or branch (those must go through AttachOrigin/SwapStore), and it does not configure the git remote (only a cert origin changes the git URL, and cert is refused).
