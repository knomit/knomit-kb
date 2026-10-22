---
type: observation
domain: [auth, windows, bridge, security]
confidence: 0.95
sources: 1
entities: [winio.DialPipeContext, winio.DialPipeAccess, winio.DialPipeAccessImpLevel, winio.PipeImpLevelAnonymous, winio.PipeImpLevelIdentification, ImpersonateNamedPipeClient, OpenThreadToken, ERROR_CANT_OPEN_ANONYMOUS, auth.DialLocal, TestPeerCred_AnonymousImpersonationLevelYieldsNoPeer]
motifs: [default-loses-the-guarantee, failure-presents-as-success]
refs: ['src://7b4887ce51d9/internal/foundation/auth/local_windows.go@0610c20083fcef9e84f42ab0311a14708bcb6193:334953a0094c26dce25419c586463451fa60020a', 'src://7b4887ce51d9/internal/foundation/auth/peercred_windows.go@0610c20083fcef9e84f42ab0311a14708bcb6193:3760cfb75f0c5da24e0fe88d49b79b1ce40e81d2', 'src://7b4887ce51d9/internal/foundation/auth/peercred_windows_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:9ae747e31f69850d1ae34927611458c963d828c4', 'https://github.com/knomit/knomit/pull/251']
---
# go-winio dials named pipes at PipeImpLevelAnonymous, where the server's OpenThreadToken fails with ERROR_CANT_OPEN_ANONYMOUS (1347) — a client that wants to be RECOGNISED must dial at PipeImpLevelIdentification

winio.DialPipeContext and winio.DialPipeAccess both dial at PipeImpLevelAnonymous. At SECURITY_ANONYMOUS the server's ImpersonateNamedPipeClient SUCCEEDS but the following OpenThreadToken FAILS, so there is no SID — and the caller silently becomes anonymous: a bridge that looks connected and holds none of its grants.

MEASURED on Windows 10 Pro 19045, go-winio v0.6.2, x/sys/windows v0.46:

    OpenThreadToken: winapi error #1347

1347 is ERROR_CANT_OPEN_ANONYMOUS. Clients must dial with winio.DialPipeAccessImpLevel(ctx, path, GENERIC_READ|GENERIC_WRITE, winio.PipeImpLevelIdentification). auth.DialLocal does, and it is the only dial path the bridge and the tests use.

VERIFIED BY FORCED FAILURE: reverting auth.DialLocal to PipeImpLevelAnonymous fails five tests in internal/foundation/auth, including the dedicated negative control TestPeerCred_AnonymousImpersonationLevelYieldsNoPeer — which carries a positive control on the SAME listener, so a broken listener cannot be mistaken for the level. TestReadClientSID_AnonymousFailsWithCantOpenAnonymous asserts the errno itself with errors.Is, because ERROR_ACCESS_DENIED would also come from OpenThreadToken and mean something quite different. Both live in internal/foundation/auth/peercred_windows_test.go.
