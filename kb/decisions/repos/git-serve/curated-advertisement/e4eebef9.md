---
type: observation
domain: [repos, store, web, git, git-serve]
confidence: 0.95
sources: 1
entities: [buildAdvRefs, Service.UpstreamBranch, GitRemoteHandler, splitGitRoute, errWantNotAdvertised, refs/knomit/agent-base, refs/remotes/origin]
motifs: [view-not-raw-state, fail-loud-not-partial]
refs: ['src://7b4887ce51d9/internal/store/httphandler_advrefs.go@c7423c3e3a7ad8280a73b9269d1f416844d14656:d93e23ba97cbd9c237a026366e664fa48ba20a43', 'src://7b4887ce51d9/internal/store/httphandler.go@c7423c3e3a7ad8280a73b9269d1f416844d14656:ae58ff42277f5c3ac4af1c3eaddc451fe1002bcb', 'src://7b4887ce51d9/internal/web/gitremote.go@c7423c3e3a7ad8280a73b9269d1f416844d14656:87c55711700dc695f0111f6a3cfd1601ab4d4bf9', 'src://7b4887ce51d9/internal/store/httphandler_receivepack.go@c7423c3e3a7ad8280a73b9269d1f416844d14656:d4c5a0a6f3a5384d6fb9f9c54a2d5942389786f5', 'kb://3ec012f5b4d2/kb/decisions/store/configurable-upstream-branch/bbb79d85.md', 'kb://3ec012f5b4d2/kb/architecture/store/git-serve/receive-pack/7642c213.md', 'https://github.com/knomit/knomit/pull/205', 'https://github.com/knomit/knomit/pull/206', 'https://github.com/knomit/knomit/pull/340']
---
# The /git endpoint serves a CURATED advertisement, not the store's ref database — HEAD on the consensus branch, only refs/heads/<upstream> and refs/heads/agent/* listed, and a missing consensus branch is a refusal at both endpoints

Decided 2026-09-15 (PR #205) so that one knomit instance can be the git origin of another. Before this, /git handed go-git's built-in server the raw ref database: HEAD pointed at the serving instance's AGENT branch, and refs/remotes/origin/* (the serving instance's own remote-tracking copies of GitHub, including other machines' agent branches) and the private refs/knomit/agent-base/* watermark were all advertised. A subscriber that found no refs/heads/main silently followed the other instance's unmerged agent branch.

What is served now (buildAdvRefs, internal/store/httphandler_advrefs.go): HEAD is a symref to refs/heads/<upstream> where <upstream> is Service.UpstreamBranch() — Remote.Branch from the origin row, else "main" — evaluated per request; only refs/heads/<upstream> and refs/heads/agent/* are listed; refs/remotes/* and refs/knomit/* are hidden. Capabilities: agent=knomit/<version>, ofs-delta, shallow, symref. Wants outside the advertised tips are refused with a git ERR pkt (git's own allowAnySHA1InWant=false), so hidden refs are also unfetchable by hash.

Missing consensus branch: if refs/heads/<upstream> does not exist, BOTH /info/refs and /git-upload-pack answer with an ERR pkt naming the branch (real git: `fatal: remote error: knomit: consensus branch does not exist in this store: "master"`, exit 128; go-git returns the same text). This replaced an earlier draft that advertised a headless view, under which `git clone` exited 0 with an empty working tree — silent partial success. Reachable case: upstream configured as master while the store holds main. The ERR sits after the `# service=git-upload-pack` line and its flush; both clients render it.

Route tolerance (PR #206, splitGitRoute in internal/web/gitremote.go): at most ONE ".git" suffix and at most ONE trailing slash resolve to the same repo; kb.git.git, kb///, and KB still 404. The two forms miss at different layers (.git is an outer rm.Get miss; a trailing slash makes the inner mux path //info/refs), so each has its own fix and test.

Options rejected: (a) raw advertisement as before — subscribers of a store without local main follow the agent branch, and every machine name that ever pushed to the origin leaks; (b) consensus-branch only — a peer could never inspect unmerged agent work.

Misreadings: "hidden" does NOT mean "only hidden from ls-remote" — wants are validated against the advertised tips, so a hash of a hidden ref is refused. "Curated" does NOT change packfile bytes; only names and HEAD differ. The .git/slash tolerance is NOT a general suffix-strip or slash-collapse.

Since F11 the receive-pack advertisement (`/info/refs?service=git-receive-pack`) is built from the SAME buildAdvRefs view, minus HEAD (receive-pack never advertises one), with its own capabilities — agent, report-status, side-band-64k, ofs-delta, and never delete-refs. So a peer branch pushed to the host (refs/heads/agent/<peer>) is from then on advertised to every fetcher like any agent branch (kb/architecture/store/git-serve/receive-pack).
