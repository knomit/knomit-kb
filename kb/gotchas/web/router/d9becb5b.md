---
type: observation
domain: [web, routing, origin-sessions, chi]
confidence: 0.85
sources: 1
entities: [Server.originSessionsRouter, apiRouter, chi.Mux.Mount, NotFound, MethodNotAllowed]
motifs: [first-writer-wins]
refs: ['src://7b4887ce51d9/internal/edge/web/router.go@c3c293b01e0b4004b9f61bb0a6fb1716ac6385b0:3f9256b823837b8ca6aa87d00721c9be6ac63c13', 'https://github.com/knomit/knomit/pull/626']
---
# chi v5's Mux.Mount copies the parent's NotFound and MethodNotAllowed handlers into a subrouter that has none, so the shared origin-sessions subrouter keeps the handlers of whichever apiRouter mounted it first; harmless only while both apiRouters install the same ones

web builds ONE origin-sessions subrouter per Server (sync.Once) and mounts it in both Handler's apiRouter and OAuthHandler's. In chi v5 (module v5.2.5), Mux.Mount sets the parent's NotFound and MethodNotAllowed on a subrouter only if it has none of its own; the first parent to mount wins (Handler's, or OAuthHandler's if built first) and the second mount leaves them. Consequence: unmatched origin-session paths on the second listener are answered by the first listener's handlers. This is harmless only while every apiRouter installs the SAME problem+json NotFound and MethodNotAllowed (at the top of apiRouter); a per-listener handler would be silently ignored on one listener. Documented in the comment on originSessionsRouter in router.go. Third-party behavior: pin chi v5.2.5; re-check mux.go Mount after a chi upgrade.
