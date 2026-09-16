---
type: observation
domain: [repos, web, git-serve, security]
confidence: 0.9
sources: 1
entities: [GitRemoteHandler, web.Server.Handler, /git, KNOMIT_GIT_SERVE]
motifs: [boundary-outside-the-process]
refs: ['src://7b4887ce51d9/internal/web/gitremote.go@3df17f54b6f6ae29984d43b926c500a834f84831:6af6eb698f6ac7404d64143b67f4bd451f24910b', 'https://github.com/knomit/knomit/pull/205']
---
# /git is deliberately unauthenticated — the network is the boundary — pending a server-wide auth design that /git will inherit; there is no auth mechanism anywhere in the HTTP server to reuse

Decided 2026-09-15 with the user (PR #205). Options: (A) no auth now, the network (tailnet, VPN, reverse proxy) gates access as it does for the API; (B) a static bearer token on /git only, via a git.token config value; (C) defer to a server-wide auth design. Chosen: A phrased as C — keep /git open, consistent with the rest of the server, and treat authentication as ONE design for the whole HTTP server later rather than a one-off on /git that the API would not follow.

Consequences: anyone who can reach the port can fetch every served repo, every advertised branch. The client side already supports token/basic/ssh/none auth methods, but nothing on the server checks them, so a subscriber should use auth method none against a knomit origin. When exposing an instance, put the whole server behind the network boundary; do not assume /git is gated because the API "feels" private.

Misreading: this is NOT "auth on /git was forgotten". It was considered and deferred; adding a /git-only token now would set a precedent the API does not follow.
