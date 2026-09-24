---
type: observation
domain: [oauth, auth, web, security, decisions]
confidence: 0.85
sources: 1
entities: ['[oauth].addr', cmd/serve_oauth.go, openOAuthServer, web.Server.OAuthHandler, web.BearerMiddleware, AuthMiddleware, WWW-Authenticate, resource_metadata, oauth.ResourceMetadataURL]
motifs: [additive-listener-not-upgrade, proxy-looks-local]
refs: ['src://7b4887ce51d9/internal/web/oauth_router.go@9b98436ca0603d07cf111c858c8c78b53f9e953d:9a966a2efaadc2626da784ce68359f7c8a683b70', 'src://7b4887ce51d9/internal/web/bearer_middleware.go@9b98436ca0603d07cf111c858c8c78b53f9e953d:0489735e06c25965af684daadc2da70585896a23', 'src://7b4887ce51d9/cmd/serve_oauth.go@9b98436ca0603d07cf111c858c8c78b53f9e953d:67cc938c59c731b32af2b6e90600a5cb237f7c5f', 'src://7b4887ce51d9/internal/web/oauth_router_test.go@9b98436ca0603d07cf111c858c8c78b53f9e953d:f9117ea10e98afcfd896acc7cfb11141f9a5b83c', 'kb://3ec012f5b4d2/kb/decisions/auth/tls-separate-listener/34bcb7f1.md', 'kb://3ec012f5b4d2/kb/invariants/auth/one-principal-one-check/8d5cc79d.md', 'https://github.com/knomit/knomit/pull/278']
---
# Bearer tokens are judged on a THIRD listener, [oauth].addr, with its own router — not as a step in AuthMiddleware — and that listener's BearerMiddleware is the only 401 knomit sends: no Authorization header, or any header that does not verify

OPTIONS (F19 phase 3a; worker D1 + reviewer B1, master ruling R1/R2, 2026-09-23): (a) a bearer step inside AuthMiddleware on the plain listener, with the OAuth surface mounted there; (b) force [auth].require = true whenever an issuer is set; (c) a third listener that is bearer-only. CHOSEN: (c).

WHY NOT (a): a reverse proxy on the same host connects from 127.0.0.1, so with the default require = false every proxied request became the anonymous loopback principal, which holds loopback_default (read, write, push:own, operator, admin) — a caller needed no token at all and could reach an admin-gated approve endpoint to approve itself. WHY NOT (b): it breaks the web UI over loopback TCP on every OAuth-enabled instance. (c) changes nothing on the plain, socket/pipe and TLS listeners; a reverse proxy fronts [oauth].addr only.

SHAPE: [oauth].issuer and [oauth].addr together (both or neither). cmd/serve_oauth.go opens its own http.Server over web.Server.OAuthHandler with the plain server's timeouts and base context and NO ConnContext (nothing there reads peer credentials or a listener mark); a bind failure is fatal. OAuthHandler serves the public OAuth routes unauthenticated (so [auth].require cannot hide discovery or the token endpoint), the API and MCP tree under /api/v1 behind BearerMiddleware then Require(read), and nothing else — /git, /docs, the web UI and the operator's /api/v1/oauth/pending endpoints are absent; unknown paths are a 404 only AFTER authentication.

401 IS NARROW: exactly two cases, both with WWW-Authenticate: Bearer realm="knomit", resource_metadata="<issuer origin>/.well-known/oauth-protected-resource<issuer path><request path>" (the RFC 9728 path form for the requested resource, so a client that follows it asks for a token confined to that resource): no Authorization header (no error code, RFC 6750 §3.1); an Authorization header of any scheme or shape, or two of them, that does not verify — unknown, expired, revoked, not for this resource (error="invalid_token"). A verified token lacking a permission is 403, as everywhere else. A non-canonical request path is a 400 before any token is judged.

On every OTHER listener an Authorization header is IGNORED: a valid write token gives a read-only anonymous loopback caller nothing, and a garbage one refuses nothing (TestPlainListener_IgnoresAuthorization). auth_middleware.go's code is unchanged; its header comment says so.
