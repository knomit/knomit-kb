---
type: observation
domain: [fleet, repos, web, cli]
confidence: 0.95
sources: 1
entities: [/api/v1/fleet, knomit fleet, Fleet tab, control.db, fleet ontology preset, knomit.toml]
refs: ['kb://3ec012f5b4d2/kb/architecture/fleet/registration/05e5af16.md', 'kb://3ec012f5b4d2/kb/decisions/fleet/member-record/e328b6ea.md', 'kb://3ec012f5b4d2/kb/decisions/verify/forge-ci/fda838bb.md', 'https://github.com/knomit/knomit/pull/334']
---
# Fleet membership is instance state: registered through the running server's REST API (CLI is a thin client), accepted by a human merging the agent branch into the fleet's main, tracked in control.db, and the fleet repo is recognised by its ontology preset alone

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

CHOICE: GET/PUT/DELETE /api/v1/fleet on the running server, the Fleet tab, and `knomit fleet` as a local-socket client. PUT clones the fleet, writes this instance's record on its agent branch and pushes; the record is pending until a human merges it. The fleet repo is the one mounted repo whose ontology is the `fleet` preset (at most one per instance). Mechanics: kb/architecture/fleet/registration/05e5af16.md.

NON-SCOPE: no KB ontology ever names a fleet; the CI gate gets the fleet URL from its own workflow input (kb/decisions/verify/forge-ci/fda838bb.md).
