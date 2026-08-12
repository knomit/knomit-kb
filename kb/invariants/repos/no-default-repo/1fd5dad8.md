---
kind: pragmatic
type: policy
domain: [repos, lifecycle, bootstrap, invariant]
confidence: 0.95
sources: 1
entities: [Manager.Start, Manager.openOne, Manager.Create, Manager.Archive, repoBuilder.openGit, config.GitConfig, internal/repos/manager.go, internal/repos/lifecycle.go]
refs: ['src://7b4887ce51d9/internal/repos/manager.go@0df3b891b86627bcce901c448974ad9cc99deb10:06cf7eeab4383ed0c642ce38e3b8b887040c2f2a', 'src://7b4887ce51d9/internal/repos/builder.go@0df3b891b86627bcce901c448974ad9cc99deb10:a10b69cbf5d20057fbef3be5fafc42bade0fe9c0', 'src://7b4887ce51d9/internal/repos/lifecycle.go@0df3b891b86627bcce901c448974ad9cc99deb10:2edd77e3d3ff69c910e3439472b42c16ba6a940b', 'src://7b4887ce51d9/internal/config/config.go@0df3b891b86627bcce901c448974ad9cc99deb10:bb9c9026a0068dca14e6752429fcaeaf24450ff2']
---
# knomit has NO default repo — Manager.Start opens what is on disk and CREATES nothing; zero repos is a valid steady state

No repo is privileged in knomit, by any name. There is no `config.DefaultRepoName` constant, no `isDefault` flag on repoBuilder, and no `initDefaultGit`. All were removed (2026-08-07).

The two halves of the rule:

1. **Start OPENS; it never CREATES.** `Manager.Start` makes `<home>/repos/`, opens `control.db`, globs `repos/*.db` (sorted, skipping session sidecars and invalid names), starts the session reaper, and registers exactly what it found. On a fresh home that is ZERO repos, and Start returns nil — an empty knomit is a healthy boot, not a failure. `repoBuilder.openGit` therefore returns the `OpenRepo` error unconditionally: a .db with no git data is a broken repo, not an invitation to seed one under that name.

2. **Zero is reachable as well as initial.** `Manager.Archive` has no default-repo guard and no last-repo guard (`ErrCannotArchiveDefault` and `ErrCannotArchiveLast` are both deleted). Archiving the only repo succeeds, returns 200, and leaves the manager empty; the archive is restorable, so nothing is lost.

Every repo is born through `Manager.Create` — `initLocal` (InitRepo + ontology seed) for preset/custom, `initClone` (InitFromRemote) for clone. That is the ONLY creation path; `POST /api/v1/repos` is its only remote entry point, and MCP cannot create repos at all.

CONSEQUENCE for anything that resolves a repo: you may not fall back to a hardcoded name, and you must handle the empty set. `knomit reset --name`, `knomit verify --repo` and the bridge's `--repo` are all REQUIRED flags for this reason. The web UI picks `repoList[0]`, never a name, and its `reposLoaded`-gated "no repos" screen is the ordinary first-run state.

WHAT THIS DOES NOT MEAN: the name "core" was not renamed to something else — the CONCEPT of a default repo is gone. A test fixture or a user's own repo may still be called "core"; that name now carries no meaning to the product. Do not reintroduce a startup-creates-a-repo path to make a fresh install "less empty".
