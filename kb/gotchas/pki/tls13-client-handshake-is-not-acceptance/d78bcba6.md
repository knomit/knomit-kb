---
type: observation
domain: [pki, tls, security, go]
confidence: 0.9
sources: 1
entities: [pki.Dial, pki.ErrRefusedByPeer, peerAccepted, tls.Dialer, net.OpError, remote error, F10, F13]
motifs: [one-sided-success, late-arriving-refusal]
refs: ['src://7b4887ce51d9/internal/pki/client.go@4b523a15c602c173f58465a909c5c19708cbb9c8:28e6de344dc536ce082528e5ddd04104c005f9d8', 'src://7b4887ce51d9/internal/pki/client_test.go@4b523a15c602c173f58465a909c5c19708cbb9c8:b56fcd41ab5553e5e0d7dfbe295e400f68a1b849', 'https://www.rfc-editor.org/rfc/rfc8446#section-4.4.2', 'kb://3ec012f5b4d2/kb/invariants/pki/no-insecure-skip-without-verifyconnection/a3e656d7.md']
---
# In TLS 1.3 a client's handshake COMPLETES before the server has judged the client certificate, so a successful tls.Dial proves only that WE accepted the peer — pki.Dial therefore completes one HTTP exchange and maps a remote alert to ErrRefusedByPeer

OBSERVED (F19 phase 2 manual run, two real `knomit serve` instances, go1.26.6): after instance B's certificate was revoked and instance A's CRL updated, A refused B at the handshake and logged `pki: certificate revoked` — yet `tls.Dialer.DialContext` on B returned a connection with no error, and the first version of `pki.Dial` reported 'dial ok'. The unit tests had not caught it because every Dial refusal test was the CLIENT refusing the SERVER; none had the server refuse the client.

WHY: in TLS 1.3 the client sends its Certificate and Finished and is done; the server verifies the client certificate AFTER that. If it refuses, its alert reaches the client only on the client's next read (Go reports it as a *net.OpError with Op "remote error", e.g. 'tls: bad certificate'). A completed client handshake therefore proves one direction only.

WHAT knomit DOES: `pki.Dial` (the probe F10 peers and F13 host election will call) completes one minimal HTTP/1.1 exchange after the handshake (`HEAD /`, `Connection: close`, bounded by the context deadline or a 10s defensive bound). Any status line means the peer accepted us; a remote alert returns `ErrRefusedByPeer`; anything else is a plain error. The refusal REASON is only in the peer's log — the dialer sees an alert, not ErrRevoked. `TestDial_ServerRefusingUsIsAnErrorNotOK` was written red first and turns red again when the check is removed.

CONSEQUENCE for F10/F13 and anyone else probing a peer: never read 'handshake succeeded' as 'mutually authenticated'. `pki.HTTPClient` is not affected in practice because a request always reads a response, which surfaces the alert — but code that dials raw TLS and then idles, or that caches 'reachable' from a dial, is.

MISREADING TO AVOID: this is not a knomit verifier bug and not specific to revocation — any server-side refusal of the client certificate (other root, expired, SAN mismatch) behaves the same way.
