---
type: observation
domain: [repos, store, web, git, git-serve]
confidence: 0.95
sources: 1
entities: [buildAdvRefs, Service.UpstreamBranch, GitRemoteHandler, splitGitRoute, errWantNotAdvertised, refs/knomit/agent-base, refs/remotes/origin]
motifs: [view-not-raw-state, fail-loud-not-partial]
refs: ['src://7b4887ce51d9/internal/store/httphandler_advrefs.go@3df17f54b6f6ae29984d43b926c500a834f84831:d21a9297d8e3962997a423047aa626f7a4367ee9', 'src://7b4887ce51d9/internal/store/httphandler.go@3df17f54b6f6ae29984d43b926c500a834f84831:fa0b27c5126394fc906cb589f54965c316337e23', 'src://7b4887ce51d9/internal/web/gitremote.go@3df17f54b6f6ae29984d43b926c500a834f84831:6af6eb698f6ac7404d64143b67f4bd451f24910b', 'kb://3ec012f5b4d2/kb/decisions/store/configurable-upstream-branch/bbb79d85.md', 'https://github.com/knomit/knomit/pull/205', 'https://github.com/knomit/knomit/pull/206']
---
# The /git endpoint serves a CURATED advertisement, not the store's ref database — HEAD on the consensus branch, only refs/heads/<upstream> and refs/heads/agent/* listed, and a missing consensus branch is a refusal at both endpoints

Decided 2026-09-15 (PR #205) so that one knomit instance can be the git origin of another. Before this, /git handed go-git's built-in server the raw ref database: HEAD pointed at the serving instance's AGENT branch, and refs/remotes/origin/* (the serving instance's own remote-tracking copies of GitHub, including other machines' agent branches) and the private refs/knomit/agent-base/* watermark were all advertised. A subscriber that found no refs/heads/main silently followed the other instance's unmerged agent branch.

What is served now (buildAdvRefs, internal/store/httphandler_advrefs.go): HEAD is a symref to refs/heads/<upstream> where <upstream> is Service.UpstreamBranch() — Remote.Branch from the origin row, else "main" — evaluated per request; only refs/heads/<upstream> and refs/heads/agent/* are listed; refs/remotes/* and refs/knomit/* are hidden. Capabilities: agent=knomit/<version>, ofs-delta, shallow, symref. Wants outside the advertised tips are refused with a git ERR pkt (git's own allowAnySHA1InWant=false), so hidden refs are also unfetchable by hash.

Missing consensus branch: if refs/heads/<upstream> does not exist, BOTH /info/refs and /git-upload-pack answer with an ERR pkt naming the branch (real git: `fatal: remote error: knomit: consensus branch does not exist in this store: "master"`, exit 128; go-git returns the same text). This replaced an earlier draft that advertised a headless view, under which `git clone` exited 0 with an empty working tree — silent partial success. Reachable case: upstream configured as master while the store holds main. The ERR sits after the `# service=git-upload-pack` line and its flush; both clients render it.

Route tolerance (PR #206, splitGitRoute in internal/web/gitremote.go): at most ONE ".git" suffix and at most ONE trailing slash resolve to the same repo; kb.git.git, kb///, and KB still 404. The two forms miss at different layers (.git is an outer rm.Get miss; a trailing slash makes the inner mux path //info/refs), so each has its own fix and test.

Options rejected: (a) raw advertisement as before — subscribers of a store without local main follow the agent branch, and every machine name that ever pushed to the origin leaks; (b) consensus-branch only — a peer could never inspect unmerged agent work.

Misreadings: "hidden" does NOT mean "only hidden from ls-remote" — wants are validated against the advertised tips, so a hash of a hidden ref is refused. "Curated" does NOT change packfile bytes; only names and HEAD differ. The .git/slash tolerance is NOT a general suffix-strip or slash-collapse.
