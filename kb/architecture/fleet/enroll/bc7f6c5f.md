---
type: reference
domain: [fleet, repos, web, architecture]
confidence: 0.92
sources: 1
entities: [Manager.EnrollFleetMember, fleetURLForBundle, FleetError.Choices, address_required, address_unknown, address_not_https, address_not_applicable, addresses_required, external_addresses]
motifs: [ask-dont-guess]
refs: ['kb://3ec012f5b4d2/kb/invariants/fleet/enroll/5c5a3bef.md', 'kb://3ec012f5b4d2/kb/invariants/fleet/member-addresses/40335d34.md', 'src://7b4887ce51d9/internal/repos/fleet_membership.go@e417fd76d128f25adff9ffb02517b6fd094651ca:41f1c3a467f966433e39fcf73881dea13e7545ca', 'src://7b4887ce51d9/internal/web/handlers_fleet_membership.go@e417fd76d128f25adff9ffb02517b6fd094651ca:d908578af3414d64ecfc5d03c824c5e510d95a65', 'src://7b4887ce51d9/internal/web/static/openapi.yaml@e417fd76d128f25adff9ffb02517b6fd094651ca:f62fcecc708da357e56f942c64c60e0ef714c3ac', 'kb://3ec012f5b4d2/kb/architecture/fleet/host-record/9c4e1732.md', 'kb://3ec012f5b4d2/kb/architecture/fleet/join-check/6fba2b2c.md', 'https://github.com/knomit/knomit/pull/526', 'https://github.com/knomit/knomit/pull/527', 'https://github.com/knomit/knomit/pull/528']
---
# POST /fleet/enroll takes an optional address and the bundle carries ONE fleet URL: with several external_addresses and none chosen it is address_required (with choices), an unlisted one address_unknown, a non-https one address_not_https, and an address given for a copy that has an origin address_not_applicable

Built in PR #526 (merge 0488c05d), internal/repos/fleet_membership.go fleetURLForBundle; every refusal comes BEFORE any issuance. The user's ruling: "why not ask during the bundle generation which one to use, if more than one are present?" The master chooses, at enroll, the address the member will reach it at; nothing guesses and nothing tries addresses in order.

THE URL IN THE BUNDLE.
- The fleet repository on this instance syncs with an ORIGIN (a forge, or a hub that is not this instance): the bundle carries that origin's URL as stored, and `address` is refused: address_not_applicable (422, the message names the URL). There is nothing to choose.
- This instance is the hub (no origin): https://<chosen external address>/git/<repo name>, address trimmed of a trailing slash. With exactly ONE entry in [external_addresses] no choice is needed. With none: addresses_required (422). With several and no `address`: address_required (422, `choices` lists them). An `address` that is not one of the configured entries: address_unknown (422, `choices` lists them). A chosen or single address that is not https:// (the config accepts any scheme://host[:port]): address_not_https (422), because members reach the hub's mTLS listener ([tls].addr) over https and an http bundle could never work.

CONSEQUENCES. The bundle URL is operator text copied verbatim from external_addresses (kb/invariants/fleet/member-addresses/40335d34.md), so a wrong address is the operator's. A caller that gets address_required must ask the operator and call enroll again; enroll is not idempotent for the same id once a leaf is on main (kb/invariants/fleet/enroll/5c5a3bef.md), but these refusals happen before issuance so a retry is safe. Enroll's other 422 codes: invalid_agent_id, invalid_key.

MISREADINGS. 'address is a way to point a forge-hosted fleet elsewhere': it is refused there. 'The bundle lists all the hub's addresses': it carries one.

UPDATE (PR #527, verified at merge commit c6e282db) — KNOWN GAP (F3, queued for PR7b). For a fleet hosted on a forge, fleetURLForBundle returns the master's origin URL VERBATIM (o.URL), userinfo included. So while the origin URL embeds a credential (a token in the URL), every enroll answer's `fleet_url` and every bundle `url:` line carries it; the joiner clones with it, stores it in its origin row, and join:check's response `url` echoes it. The HOST RECORD and GET /fleet strip userinfo (credentialFree), so this is the one place a forge fleet's origin credential still leaves the master. The `address_not_applicable` text (it embeds o.URL) reaches only the local operator of the master. This reads false once fleetURLForBundle returns credentialFree(o.URL) (the joiner supplies its own auth). Related: a forge origin with an http:// scheme is enrolled into a bundle that join refuses (see kb/architecture/fleet/join-check/6fba2b2c.md, F1). MISREADING: "credentials never leave the master": that holds for the host record and GET /fleet only.

## UPDATE (PR #528, verified at merge e417fd76): the F3 gap above is CLOSED

fleetURLForBundle now returns credentialFree(o.URL) (internal/repos/fleet_membership.go), so the enroll answer's `fleet_url` and the bundle `url:` line carry no userinfo; the joiner supplies its own auth for a forge. credentialFree also strips `scheme://...@` by regex when url.Parse fails, and keeps an ssh:// user (TestCredentialFree; TestCreateFleet_PreflightRefusals asserts no `tok@` in any refusal; TestFleetCreate_Refusals is the REST case). READ THE KNOWN-GAP PARAGRAPH ABOVE AS HISTORY. MISREADING: "no credential is ever shown": the address_not_applicable text reaches only the master's local operator and is also credential-free.
