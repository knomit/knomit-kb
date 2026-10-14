---
type: observation
domain: [pki, fleet, architecture]
confidence: 0.9
sources: 1
entities: [pki.IssueLeaf, pki.Issuance, pki.RevokeSerial, pki.Revocation, pki.SignCRL, pki.ParseRevokedLedger, pki.IssueInstance, pki.Revoke, pki.IssueCRL, pki.NeedsRenewal, pki.CRLNeedsResign, renewWindowDivisor, newSerial, issued.jsonl, revoked.jsonl]
motifs: [pure-core-thin-shell]
refs: ['src://7b4887ce51d9/internal/pki/issue.go@8e86308d84b35c8225020d0827fffb561908ad12:61920d4c727c613fb00d7ada4c9a36901004da33', 'src://7b4887ce51d9/internal/pki/crl.go@8e86308d84b35c8225020d0827fffb561908ad12:d1a971ec2430e3518d62f59f8569161293e9ebd9', 'src://7b4887ce51d9/internal/pki/renew.go@8e86308d84b35c8225020d0827fffb561908ad12:3caf5dee816726450656a141452c8fb68b5a561a', 'src://7b4887ce51d9/internal/pki/ledger_test.go@8e86308d84b35c8225020d0827fffb561908ad12:cbaa67f263717023852a38305392707172a749fb', 'src://7b4887ce51d9/internal/pki/custody_review_test.go@8e86308d84b35c8225020d0827fffb561908ad12:328c056c60c02694b9e6b0428101323097e26570', 'https://github.com/knomit/knomit/pull/512']
---
# Issuance and revocation are pure in internal/pki: IssueLeaf and RevokeSerial return the leaf/CRL PEM and the exact ledger line, SignCRL is the pure IssueCRL; IssueInstance, Revoke and IssueCRL are thin wrappers that place them in the fleet root directory; renewal falls due in the final THIRD of an artifact's own lifetime

So the fleet reconcile can write certificates, the CRL and the two ledgers as fleet-repo artifacts (design D7/P5), signing is split from file placement (PR #512):
- `IssueLeaf(root, pub, host, role, validity, now) (Issuance{CertPEM, Record, Line}, error)` writes nothing; `Line` is the JSON record plus '\n', byte-identical to what `IssueInstance` appends to issued.jsonl. The SAN carries the role and the host as a free label only (knomit://<role>/<label>, no fp8, PR #513); the identity of the leaf is its key (Identity.Fingerprint is computed from the leaf's public key), never a name. The wrappers place files in the fleet root directory (default <home>/fleet since PR #513). No behaviour change otherwise.
- `RevokeSerial(root, prior []Revoked, current *big.Int, r, thisUpdate, next) (Revocation{Line, CRLPEM, Number}, error)`: the CRL lists prior + r, Number = current+1 (1 when current is nil). Callers holding ledger bytes parse them with `ParseRevokedLedger`. r is listed as the ledger reads it back (JSON round trip), so a later reissue from the ledger lists the same time.
- `SignCRL(root, revoked, number, thisUpdate, next)` is the pure form of `IssueCRL`.
- The wrappers (`IssueInstance`, `Revoke`, `IssueCRL`) stay because cmd/identity.go and internal/pki/pkitest call them. `Revoke` now signs before appending, so a failed sign writes nothing.

RENEWAL WINDOW (renew.go): `NeedsRenewal(cert, now)` / `CRLNeedsResign(crl, now)` are true when less than ONE THIRD (renewWindowDivisor = 3) of the artifact's OWN lifetime remains (NotAfter-NotBefore, NextUpdate-ThisUpdate); no day-count literal. The check is `end.Sub(now) < life/renewWindowDivisor`, with no time.Duration overflow (the earlier `remaining*3 < life` form overflowed past about 97 years left and made a long-lived cert due from day one). Zero or negative lifetime is due; a CRL without NextUpdate never falls due.

TEST SEAM. `newSerial` is a package variable only so a test can pin the serial and compare the pure and file variants field by field (TestIssueLeaf_EquivalentToIssueInstance, TestRevokeSerial_EquivalentToRevoke). DER is not compared.

MISREADING. The pure functions take the clock as a parameter; they do not read time.Now(). A caller that passes a stale `now` issues a stale certificate.
