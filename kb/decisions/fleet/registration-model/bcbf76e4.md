---
type: observation
domain: [fleet, repos, web, cli]
confidence: 0.95
sources: 1
entities: [/api/v1/fleet, knomit fleet, Fleet tab, control.db, fleet ontology, knomit.toml]
refs: ['kb://3ec012f5b4d2/kb/architecture/fleet/registration/05e5af16.md', 'kb://3ec012f5b4d2/kb/decisions/fleet/member-record/e328b6ea.md', 'kb://3ec012f5b4d2/kb/decisions/verify/forge-ci/fda838bb.md', 'kb://3ec012f5b4d2/kb/decisions/fleet/durable-unregistration/adb65151.md', 'kb://3ec012f5b4d2/kb/decisions/fleet/source-of-truth/6ddfdd1a.md', 'kb://3ec012f5b4d2/kb/invariants/fleet/baked-ontology/reserved-id/4cdcf99e.md', 'https://github.com/knomit/knomit/pull/334', 'https://github.com/knomit/knomit/pull/520']
---
# Fleet membership is instance state: registered through the running server's REST API (CLI is a thin client), accepted by a merge of the agent branch into the fleet's main (originally ruled as a human merge; the product text now says the fleet's master merges), tracked in control.db, and the fleet repo is recognised by its reserved ontology id alone (the built-in fleet ontology, not a preset)

DECISION (F09 PR 5, user rulings 2026-09-27; D4 of the design).

USER, VERBATIM:
- "no, not in the ontology, this will prevent us from rebuilding the fleet repo. The fleet should be specified as part of the knomit instance registration"
- "create an agent branch, record there the agent facts, push to remote. Then, a human accepts or declines the registration"
- "wouldn't registering via the API actually make a lot more sense ? Then one can start the server in 'standalone' mode, then using the UI or curl could register in a fleet, unregister from a fleet, move it to a new fleet, and so on. Also, why not store the fleet info in control.db ? It's about the state of the fabric, not how knomit instance actually starts"
- D4: "we need to define an ontology for the fleet, and one way to detect which repo is the fleet repo is to see which one has the fleet ontology. We can also enforce a single repo with that ontology"

OPTIONS REJECTED:
- A `fleet: <URL>` attribute in each KB's ontology (the fleet could not be rebuilt without editing every KB).
- Operator-only registration by subscribing an agent and writing its record out of band.
- A `[fleet].repo` setting in knomit.toml, and CLI-only registration.
- Marking the fleet repo with a `role: fleet` attribute; a reserved repo name; a control.db row holding the fleet URL.

CHOICE: GET/PUT/DELETE /api/v1/fleet on the running server, the Fleet tab, and `knomit fleet` as a local-socket client. PUT clones the fleet, writes this instance's record on its agent branch and pushes; the record is pending until a human merges it. The fleet repo is the one mounted repo whose ontology id is `fleet` (at most one per instance; since PR #520 that ontology is built into knomit, not a preset, and the id is reserved). Mechanics: kb/architecture/fleet/registration/05e5af16.md.

NON-SCOPE: no KB ontology ever names a fleet; the CI gate gets the fleet URL from its own workflow input (kb/decisions/verify/forge-ci/fda838bb.md).

UPDATE (PR #520, verified at merge commit 1a6c03f7): the title's "recognised by its ontology preset alone" is wrong now. A MOUNTED repo is still recognised by its ontology id alone (fleetRepo, IsFleet), but the ontology is the built-in fleet ontology, not a preset, no create door can produce it (the id `fleet` is reserved, kb/invariants/fleet/baked-ontology/reserved-id/4cdcf99e.md), and at boot only the repository recorded in control.db instance_fleet_repo is admitted, so recognition by id is no longer sufficient to be the fleet at boot. Also reworded in the product: the record counts once the fleet's MASTER merges it into main (consensus: auto on a knomit-hosted fleet), not a human merge; on a forge-hosted fleet a person still merges on the forge until the hosting change lands. The user's verbatim rulings above are history and unchanged.

UPDATE (PR #524, verified at merge commit 9db8c106): the merge that accepts a registration is made by the fleet's MASTER automatically, on every hosting (knomit-hosted hub: after receive-pack judges the pushed branch; forge: after the master's own fetch judges the fetched branch), then main is published without force. 'The product text now says the fleet's master merges' is now also what the code does; no human merge is needed. See kb/architecture/fleet/master-merges.
