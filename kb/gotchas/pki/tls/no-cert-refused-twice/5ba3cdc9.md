---
type: observation
domain: [pki, tls, testing, security]
confidence: 0.95
sources: 1
entities: [pki.reloader.configFor, snapshot.check, tls.RequireAnyClientCert, tls.RequestClientCert, VerifyConnection, TestBootServer_TLSRefusesNoClientCert, AuthMiddleware]
motifs: [layered-check-masks-sabotage, sabotage-must-cross-layers]
refs: ['src://7b4887ce51d9/internal/pki/server.go@243e0a5cf8ab321b0b10d0968e946343dd7d1517:0e8d8f9046bbf7a74b8707eb26a5493e25f0ef17', 'src://7b4887ce51d9/tools/desktop/boot_tls_test.go@243e0a5cf8ab321b0b10d0968e946343dd7d1517:27fdfd961a79551845339a436ea067f00c55ebb2', 'kb://3ec012f5b4d2/kb/gotchas/pki/require-any-client-cert-is-stronger/de5b6d33.md']
---
# The mTLS listener refuses a certificate-less client at the TLS layer TWICE — ClientAuth RequireAnyClientCert AND snapshot.check's 'peer presented no certificate' — so a sabotage that weakens only one of them changes nothing observable

internal/pki/server.go configFor sets ClientAuth: tls.RequireAnyClientCert, and its VerifyConnection runs snapshot.check, which returns ErrUntrustedRoot when cs.PeerCertificates is empty. Either alone refuses a client with no certificate before any HTTP request is read.

MEASURED at 31f4072d (knomit#256): weakening ClientAuth to RequestClientCert left TestBootServer_TLSRefusesNoClientCert GREEN, and so did making check return nil on no certificate. Only weakening BOTH let the request through to AuthMiddleware, which answered 403 — and the test, which counts requests reaching the HTTP layer and refuses any response, went red.

CONSEQUENCE for reviewers designing sabotage: 'weaken RequireAnyClientCert' is not a falsifying mutation on its own; name both layers. A green result for a single-layer mutation is the defence working, not the test failing.

NOT MEANT: this does not make either layer redundant to delete. ClientAuth is also what makes crypto/tls ASK for a certificate; the comment on it explains why RequireAndVerifyClientCert is deliberately not used.
