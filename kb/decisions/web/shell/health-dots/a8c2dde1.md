---
type: observation
domain: [web, ui, shell, remote]
confidence: 0.9
sources: 1
entities: [remoteFailing, otherRepoFailing, rail-health, rail-toggle-dot, remoteErrorText, RepoInfo.remote]
motifs: [derived-state-liability]
refs: ['src://7b4887ce51d9/web/src/facets.ts@18598fbc4b5cfb3d078ab104efcfdf4463364421:5a255dc90a4f1f82cbbff1c61aef97950f4a8295', 'src://7b4887ce51d9/web/src/TargetRail.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:50aae885bc7ff8644882ad7bd8205cdd9521be00', 'src://7b4887ce51d9/web/src/TopBar.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:c36df268fc11b2f54f672d6e7b41f68b84ea6e85', 'src://7b4887ce51d9/web/src/App.rail.test.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:41083a2c5d5b9ed8d6a386ffa35c37177fd6ea4f', 'src://7b4887ce51d9/web/src/TopBar.test.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:62da4f9311a84e2ce8380998926ea62a28605b8c', 'kb://3ec012f5b4d2/kb/decisions/web/repos/remote-health-on-listing/fd41124e.md']
---
# Rail health dots read the listing for OTHER repos and remoteErrorText(state) for the current repo; the toggle dot lights only for a repo other than the current one; none under serverReadOnly

The per-repo remote alert that ManageOverview used to give lives on the rail rows and the rail toggle.

- A rail row shows `rail-health-<name>` when serverReadOnly is false AND either (a) the repo is not the current repo and its `RepoInfo.remote.sync` or `.push` is 'error' (`remoteFailing` in facets.ts), or (b) the repo is the current repo and `remoteErrorText(state)` is non-empty.
- The toggle shows `rail-toggle-dot` when serverReadOnly is false AND some repo whose name is NOT `state.repo` satisfies `remoteFailing` (`otherRepoFailing`).
- The current repo is excluded from the toggle because its failure already has the app banner. Its row follows the banner's source so the two never disagree: dismissing the banner clears that row's dot.
- `RepoInfo.remote` is kept current by the `remote` kind on /repo-events (edges only). An absent `remote` means nothing to show, never healthy.

Consequence: do not light the toggle from `remoteErrorText`, and do not read `repo.remote` for the current repo's row.

WHAT THIS DOES NOT MEAN: "the toggle dot means some repo is failing" is incomplete. A failure on the current repo alone does NOT light it (pinned in TopBar.test.tsx and App.rail.test.tsx).
