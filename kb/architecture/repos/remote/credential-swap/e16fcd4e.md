---
type: observation
domain: [repos, remote, credentials, web]
confidence: 0.9
sources: 1
entities: [Manager.ChangeOriginCredential, CredentialChange, ErrNoOrigin, ErrLinkedRepo, ErrInvalidCredential, Origins.SetCredential, handleSetOriginCredential, 'PUT /repos/{repo}/origin/credential']
motifs: [compare-and-set-write]
refs: ['kb://3ec012f5b4d2/kb/invariants/repos/control-db/secret-link/845c96d6.md', 'kb://3ec012f5b4d2/kb/invariants/repos/remote/attach-identity/545ae852.md', 'src://7b4887ce51d9/internal/corpus/repos/credential.go@1d4d4f01dc57594ed9d33c3af0a3f9d3da3d1084:a455abbc570c54184afce41af8777989ab7a0f88', 'src://7b4887ce51d9/internal/engine/store/remotesync/remote.go@1d4d4f01dc57594ed9d33c3af0a3f9d3da3d1084:bb5976c02ddccc60fae7ad328a1b0806d1281f94', 'src://7b4887ce51d9/internal/corpus/repos/control/origins.go@1d4d4f01dc57594ed9d33c3af0a3f9d3da3d1084:2673493d54a0ffdae4f726f867372cdb92110c37', 'src://7b4887ce51d9/internal/corpus/repos/events.go@1d4d4f01dc57594ed9d33c3af0a3f9d3da3d1084:774cd5c54d7ad806420fd0881f10c69fffff2a0e', 'src://7b4887ce51d9/internal/edge/web/api/manage/handlers_origin_credential.go@1d4d4f01dc57594ed9d33c3af0a3f9d3da3d1084:d2bdca6a4145b2d5d832d3d3bd7c4f1d51db58b5']
---
# PUT /repos/{repo}/origin/credential (Manager.ChangeOriginCredential) changes only the credential: a compare-and-set on the stored URL in control.db, then a compare-and-set on the injected URL in memory (SetOriginIfURL); a lost race to an attach or detach changes nothing and answers ErrOriginChanged

Manager.ChangeOriginCredential(ctx, ri, CredentialChange{AuthMethod, AuthToken, SecretID}, unlink) is the light credential path. It reads the stored origin (control.Origins.Get) and refuses: no origin (ErrNoOrigin, 409 "No origin"), a fleet peer origin with auth_method cert (ErrPeerCredential, 400), a pasted credential on a linked repo unless unlink is set (ErrLinkedRepo, 409, naming the unlink command), and a malformed credential or one whose method does not fit the URL scheme (ErrInvalidCredential, 400; wraps MethodMismatch, token needs a token, basic needs a non-empty user AND a non-empty password, ssh and none take no token, cert is never an input).

TWO compare-and-sets, one per copy of the origin:
1. Durable: Origins.SetCredential(uid, stored.URL, ...) writes only WHERE url = stored.URL. An attach that re-pointed the origin BEFORE this write wins: control.ErrOriginChanged (409 "Origin changed") and nothing is written anywhere.
2. In memory: after re-reading the row (cur), it injects with svc.Remote().SetOriginIfURL(cur.URL, ...) (remotesync.Index, under originMu), which applies only while an origin is injected AND its URL is cur.URL. An attach or detach that lands AFTER the durable write (between the re-read and the injection) has already written control.db, the git remote and memory; the swap then changes nothing in memory and answers control.ErrOriginChanged, without signalling a credential change. The durable credential it wrote was overwritten by the attach (or deleted by the detach), so control.db and memory end up agreeing on the attach's (or detach's) state.

Why the in-memory step must be conditional: makeRemoteAuthFn builds auth from the in-memory origin while go-git dials the URL in .git/config. An unconditional SetOrigin(cur) after an attach would send the OLD origin's new token to the attached host, and after a detach would resurrect the removed origin in memory.

Only when the in-memory step applies (or the store is not open, which needs no write since opening injects the control.db row) does it call ri.credentialChanged().

Related: applySwap now writes the new origin with Origins.Replace (returns the row it replaced, read in the same transaction) and, on a copy failure, undoes with Origins.Restore, which keeps a credential changed since the write instead of reverting it. So a light swap that answered 200 is not silently reverted by a failed swap.

The endpoint is behind operator (like PUT /origin), takes {auth_method, token?, user?, password?} assembled with httpx.AssembleAuthToken, and in this PR refuses a secret_id key with 400 (linking arrives with operation-time secret resolution). manage.Mount already takes an admin gate parameter for that.

Never replace SetOriginIfURL with SetOrigin in this path, and never hold a per-repo mutex shared with applySwap to serialise it (applySwap drains the store under WithRead; a shared lock risks deadlock).

What this does NOT mean: it is not a way to change the URL or branch (those must go through AttachOrigin/SwapStore), and it does not configure the git remote (only a cert origin changes the git URL, and cert is refused).
