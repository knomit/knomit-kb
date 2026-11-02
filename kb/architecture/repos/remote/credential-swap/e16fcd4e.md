---
type: observation
domain: [repos, remote, credentials, web]
confidence: 0.9
sources: 1
entities: [Manager.ChangeOriginCredential, CredentialChange, ErrNoOrigin, ErrLinkedRepo, ErrInvalidCredential, Origins.SetCredential, handleSetOriginCredential, 'PUT /repos/{repo}/origin/credential']
motifs: [compare-and-set-write]
refs: ['kb://3ec012f5b4d2/kb/invariants/repos/control-db/secret-link/845c96d6.md', 'kb://3ec012f5b4d2/kb/invariants/repos/remote/attach-identity/545ae852.md', 'src://7b4887ce51d9/internal/corpus/repos/credential.go@ff756c8d79fb028c5a9ebd74e28bf655f7d0d101:ad68cfaaa6c53525910198114077465a545572ea', 'src://7b4887ce51d9/internal/engine/store/remotesync/remote.go@ff756c8d79fb028c5a9ebd74e28bf655f7d0d101:774000da13e1f25c8e2f64974016b60d2cff1af4', 'src://7b4887ce51d9/internal/corpus/repos/control/origins.go@ff756c8d79fb028c5a9ebd74e28bf655f7d0d101:08edbc4a61619bc975e0a532a6a866fbc5d02c06', 'src://7b4887ce51d9/internal/corpus/repos/events.go@ff756c8d79fb028c5a9ebd74e28bf655f7d0d101:f5e1ce9569ca712cdd5c73a79af3d7b3b2d34319', 'src://7b4887ce51d9/internal/edge/web/api/manage/handlers_origin_credential.go@ff756c8d79fb028c5a9ebd74e28bf655f7d0d101:d2bdca6a4145b2d5d832d3d3bd7c4f1d51db58b5']
---
# PUT /repos/{repo}/origin/credential (Manager.ChangeOriginCredential) changes only the credential: a compare-and-set on the stored URL in control.db, then a compare-and-set on the WHOLE injected origin in memory (SetOriginIf); a lost race to an attach, re-attach or detach changes nothing in memory and answers ErrOriginChanged

Manager.ChangeOriginCredential(ctx, ri, CredentialChange{AuthMethod, AuthToken, SecretID}, unlink) is the light credential path. It reads the stored origin (control.Origins.Get) and refuses: no origin (ErrNoOrigin, 409 "No origin"), a fleet peer origin with auth_method cert (ErrPeerCredential, 400), a SecretID in the change (errLinkNotSupported: linking is not wired yet), a pasted credential on a linked repo unless unlink is set (ErrLinkedRepo, 409, whose text names an unlink command the HTTP endpoint cannot yet send, since the handler always passes unlink=false), and a malformed credential or one whose method does not fit the URL scheme (ErrInvalidCredential, 400; wraps MethodMismatch, token needs a token, basic needs a non-empty user AND a non-empty password, ssh and none take no token, cert is never an input).

TWO compare-and-sets, one per copy of the origin:
1. Durable: Origins.SetCredential(uid, stored.URL, ...) writes only WHERE url = stored.URL. An attach that re-pointed the origin BEFORE this write wins: control.ErrOriginChanged (409 "Origin changed") and nothing is written anywhere.
2. In memory: BEFORE the durable write it captures the injected origin (svc.Remote().InjectedOrigin(); with the store not open, the stored row as an open would inject it). After re-reading the row (cur), it injects with svc.Remote().SetOriginIf(expect, ...) (remotesync.Index, under originMu), which applies only while an origin is injected AND it equals the captured one field by field (URL, branch, method, token). So an attach, a same-URL re-attach with another credential, or a detach that lands after the durable write keeps memory; the swap answers control.ErrOriginChanged without signalling a credential change. When that attach or detach lands AFTER the swap's durable write, it also overwrote (or deleted) the row, so control.db and memory agree on its state.

Residual window, not closed: a same-URL re-attach that lands between the capture and the durable write passes the URL compare-and-set, so control.db ends with the swap's credential while memory keeps the attach's (both for the same URL; the next open re-injects control.db). A URL change in that window is refused by step 1.

Why the in-memory step must be conditional: makeRemoteAuthFn builds auth from the in-memory origin while go-git dials the URL in .git/config. An unconditional SetOrigin(cur) after an attach would send the OLD origin's new token to the attached host, and after a detach would resurrect the removed origin in memory.

Only when the in-memory step applies (or the store is not open, which needs no write since opening injects the control.db row) does it call ri.credentialChanged().

Related: applySwap writes the new origin with Origins.Replace (returns the row it replaced, read in the same transaction) and, on a copy failure, undoes with Origins.Restore. Restore puts the replaced row back WHOLE, except when the swap kept the SAME URL and the credential changed after the write: then the newer credential is kept. Across URLs it never keeps it: a credential set after the write was vetted against the incoming URL and would otherwise be sent to the previous host. So a light swap that answered 200 before the swap's write is not reverted (Replace returns its row), and one that targeted the incoming origin is dropped with it.

The endpoint is behind operator (like PUT /origin), takes {auth_method, token?, user?, password?} assembled with httpx.AssembleAuthToken, and in this PR refuses a secret_id key with 400 (linking arrives with operation-time secret resolution). manage.Mount already takes an admin gate parameter for that.

Never replace SetOriginIf with SetOrigin (or with a URL-only compare) in this path, and never hold a per-repo mutex shared with applySwap to serialise it (applySwap drains the store under WithRead; a shared lock risks deadlock).

What this does NOT mean: it is not a way to change the URL or branch (those must go through AttachOrigin/SwapStore), and it does not configure the git remote (only a cert origin changes the git URL, and cert is refused).
