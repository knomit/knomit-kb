---
type: observation
domain: [verify, fleet, security]
confidence: 0.95
sources: 1
entities: [fleet repository, member record, verify_signatures, verify_signers, F19 certificates, CRL, knomit-cert header]
refs: ['kb://3ec012f5b4d2/kb/decisions/fleet/member-record/e328b6ea.md', 'kb://3ec012f5b4d2/kb/invariants/store/verify/acceptance-gate/74a27150.md', 'kb://3ec012f5b4d2/kb/decisions/fact/ontology/root-attributes/90268b66.md', 'https://github.com/knomit/knomit/pull/334']
---
# Who may sign a KB's commits is decided ONLY by a fleet repository's member records; no key list, certificate, CRL or root key lives in any KB or commit

DECISION (F09 PR 5, user rulings 2026-09-27). The set of keys a knowledge base accepts comes from one place: the member records of a fleet repository (a knomit repo with the fleet ontology preset). A KB's ontology carries only the switch `verify_signatures: off|log|enforce`.

USER, VERBATIM (on the verify_signers list PRs 1-3 had shipped): "looks like \"verify_signers\" is required to have all the keys of all the agents that can contribute to the repo - I disagree with this, it's plain stupid ... it will make IMPOSSIBLE for new knomit instances to join an existing repo because the ontology MUST be modified. ... what we discussed was ONLY to turn signing checking on or off via verify_signatures in the ontology. The actual key collection - what must be accepted - MUST come from the PKI chain."
Then, replacing the PKI-chain designs: "we're making this super complicated IMO - remember, knomit is about knowledge and facts, so a knomit repo could be used to carry all this. How about this: registration in a fleet is done via a repo which will have a special role 'fleet'. In that fleet repo, agents will publish their public keys, we can store the revocation list, and so on. That will be the source of truth for the fleet."

OPTIONS REJECTED:
- `verify_signers`, an authorized-key list in each KB's root ontology (built in PRs 1-3): every new instance would need an ontology edit in every KB. Removed in PR 5.
- F19 certificates carried in commits (a signed `knomit-cert` header, first on the first commit per certificate, later on every commit; fallback: certificates under .knomit/certs/).
- CRL distribution (per-instance CRL files, then a root-signed CRL header with highest-number-wins, then CI variables for the CRL and fleet root).
- Ancestry-ordered revocation (a revoked key stays valid for ancestors of the first accepted revoking commit; review showed two instances could reach different verdicts on one history) and the root-rotation schemes proposed to go with it.
All went with the redirect above: no certificates in commits, no headers, no CRL, no root variables, no ancestry or time rules. F19 certificates remain transport (mTLS) only.

NON-SCOPE: this does not say the fleet repo is trusted blindly. Its records become authoritative only when a human merges them into its main (see the fleet registration-model decision), and the gate admits only the CURRENT key of an ACTIVE record (kb/invariants/store/verify/acceptance-gate/74a27150.md).
