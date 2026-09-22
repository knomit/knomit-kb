---
type: observation
domain: [repos, web, git-serve, security]
confidence: 0.9
sources: 1
entities: [GitRemoteHandler, web.Server.Handler, /git, KNOMIT_GIT_SERVE, AuthMiddleware, writeGate, isGitFetch, '[auth].require', '[tls].addr']
motifs: [boundary-outside-the-process]
refs: ['src://7b4887ce51d9/internal/web/gitremote.go@3df17f54b6f6ae29984d43b926c500a834f84831:6af6eb698f6ac7404d64143b67f4bd451f24910b', 'src://7b4887ce51d9/internal/web/server.go@4b523a15c602c173f58465a909c5c19708cbb9c8:71ed02494485426ab54919bb2c621cf87e6358ee', 'src://7b4887ce51d9/internal/web/readonly.go@4b523a15c602c173f58465a909c5c19708cbb9c8:db1c242d3824cd1767059cd3b68514ff4f1e210c', 'src://7b4887ce51d9/internal/web/auth_middleware.go@4b523a15c602c173f58465a909c5c19708cbb9c8:fba38078d92bed81406207d1c7ea1622334582c0', 'src://7b4887ce51d9/cmd/serve_tls.go@4b523a15c602c173f58465a909c5c19708cbb9c8:abca154ecf36d3d8fce04592015e1ab2cb04a121', 'https://github.com/knomit/knomit/pull/205', 'https://github.com/knomit/knomit/pull/243', 'kb://3ec012f5b4d2/kb/decisions/auth/tls-separate-listener/34bcb7f1.md', 'kb://3ec012f5b4d2/kb/decisions/auth/instance-read-implicit/85dd6b5d.md']
---
# /git was deliberately left unauthenticated in PR #205 pending ONE server-wide auth design, and it now inherits that design: phase 1 gates receive-pack by `write` and refuses unauthenticated clones under [auth].require, phase 2 serves /git to enrolled instances over mTLS — but the PLAINTEXT port still serves fetches to anyone who can reach it unless require is on

Decided 2026-09-15 with the user (PR #205). Options: (A) no auth now, the network (tailnet, VPN, reverse proxy) gates access as it does for the API; (B) a static bearer token on /git only, via a git.token config value; (C) defer to a server-wide auth design. Chosen: A phrased as C — keep /git open, consistent with the rest of the server, and treat authentication as ONE design for the whole HTTP server later rather than a one-off on /git that the API would not follow.

THAT DESIGN HAS SINCE LANDED, and /git inherited it without a /git-specific mechanism (F19):
- Phase 1 (PR #243): `AuthMiddleware` sits on the OUTER router above the `/git` mount, so every /git request gets a principal; `writeGate` wraps the mount, so `git-receive-pack` needs `write` while `git-upload-pack` (clone/fetch) is exempt from the method gate. With `[auth].require = true`, both halves of a fetch answer 403 without a principal.
- Phase 2 (F19 phase 2 PR): the optional mTLS listener (`[tls].addr`) serves the SAME handler, /git included, to enrolled instances only — never anonymous, reads included. An instance holds `read` implicitly and needs a grants row for anything else.

CONSEQUENCES NOW, stated in full: on the PLAINTEXT port with `require = false` (the default), anyone who can reach the port can still fetch every served repo and every advertised branch — reads are not enforced anywhere, only the transport boundary is. So the original advice stands for the plaintext port: put it behind the network boundary. The mTLS port is the one that needs no network boundary for authentication. The CLIENT side has not caught up: knomit's subscriber (go-git, `internal/repos`) cannot yet present an instance certificate — that is phase 2b — so a subscriber still uses auth method none against the plaintext port of a knomit origin.

Misreading: this is NOT 'auth on /git was forgotten', and it is no longer 'no auth mechanism exists to reuse'. It is also NOT 'fetches are now authenticated' — on the plaintext port without require, they are not.
