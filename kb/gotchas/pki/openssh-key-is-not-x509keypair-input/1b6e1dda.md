---
type: observation
domain: [pki, tls, identity, security]
confidence: 0.9
sources: 1
entities: [~/.knomit/id_ed25519, pki.LoadSigner, pki.PKCS8, tls.X509KeyPair, tls.Certificate.PrivateKey, ssh.ParseRawPrivateKey, go-git ClientKey, transportWithClientCert]
motifs: [format-mismatch-at-boundary, second-copy-of-secret]
refs: ['src://7b4887ce51d9/internal/pki/key.go@681fef9b82a681de6f87cc27d882101354243095:752474cad79a97f9e6827fd9f9887d9b09127ab0', 'src://7b4887ce51d9/internal/pki/server.go@681fef9b82a681de6f87cc27d882101354243095:d73dab715fc55f17fbe013571ffc146bd06f5387', 'https://github.com/go-git/go-git/blob/v5.19.2/plumbing/transport/http/common.go#L228', 'kb://3ec012f5b4d2/kb/invariants/pki/fingerprint-is-ssh-wire/1a663278.md']
---
# The instance key is OpenSSH PEM ("OPENSSH PRIVATE KEY"), which tls.X509KeyPair — and therefore go-git's ClientCert/ClientKey options — cannot parse; knomit loads it into an in-memory crypto.Signer and never writes a second copy in PKCS#8

`internal/app/identity.go` writes the instance key with `ssh.MarshalPrivateKey(priv, "")` — an 'OPENSSH PRIVATE KEY' PEM block. `tls.X509KeyPair` accepts only PKCS#1/PKCS#8/SEC1 key PEM and refuses it. go-git v5.19.2 builds its mTLS client with exactly that call (`transportWithClientCert`, plumbing/transport/http/common.go:228), so passing the key file to go-git's `ClientKey` option fails.

THE TEMPTING FIX IS FORBIDDEN: converting the key to PKCS#8 on disk creates a second copy of the one secret whose theft is the whole compromise. knomit's rule: `~/.knomit/id_ed25519` stays the ONLY on-disk copy. `pki.LoadSigner` parses it with `ssh.ParseRawPrivateKey` — which returns `*ed25519.PrivateKey`, a POINTER, dereferenced so callers see the value type — and hands the in-memory `crypto.Signer` to `tls.Certificate{PrivateKey: …}`. `pki.PKCS8` exists for an API that needs DER in memory; its doc forbids writing the result.

CONSEQUENCE for phase 2b (go-git instance-to-instance fetch): do not route the certificate through go-git's ClientCert/ClientKey/CABundle options. They also append the CA to the SYSTEM pool and apply Go's hostname check, both wrong for a knomit peer. Use a knomit http.Client built by pki.HTTPClient under a knomit-specific scheme instead.

ALSO: an `ssh.Signer` (what app.App holds) cannot give back its private key, so TLS cannot reuse it; the key file is read a second time into memory. That is a second READ, not a second copy.
