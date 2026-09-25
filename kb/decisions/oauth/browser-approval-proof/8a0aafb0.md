---
type: observation
domain: [oauth, web, security, csrf]
confidence: 0.85
sources: 1
entities: [web.browserProof, web.ownHost, web.KnomitClientHeader, X-Knomit-Client, Sec-Fetch-Site, Origin, http.LocalAddrContextKey, corsMiddleware, 'wails://localhost', '[auth].loopback_default', '[auth].require', auth.Admin]
motifs: [csrf-by-reachability, dormant-permission-goes-live]
refs: ['src://7b4887ce51d9/internal/web/oauth_pending_api.go@52176c9f3207df73a085430915d46f9b8f731c29:f662765ac5fe23e368201f60a6798663fed99687', 'src://7b4887ce51d9/internal/web/oauth_browser_gate_test.go@52176c9f3207df73a085430915d46f9b8f731c29:3c6254a1f8eeceded17b2da77c968b191c270bd2', 'src://7b4887ce51d9/internal/web/cors.go@52176c9f3207df73a085430915d46f9b8f731c29:152ebaed28cb8b4f3d67ec360a314c12c5354f81', 'kb://3ec012f5b4d2/kb/invariants/oauth/approval-local-principals-only/6d4a8514.md', 'https://github.com/knomit/knomit/pull/283', 'src://7b4887ce51d9/internal/web/oauth_browser_gate_test.go@8ca46cd87debaa7feb2245c500a203038a310bdf:0a8e58e6ffa76a202917653d1659eaec5c47cbf9', 'kb://3ec012f5b4d2/kb/invariants/auth/loopback-anonymous-host/4d629192.md', 'https://github.com/knomit/knomit/issues/281']
---
# The web UI approves OAuth requests with a same-origin PROOF, not a token or cookie: Host a loopback spelling with the arrival port, X-Knomit-Client: web, Sec-Fetch-Site same-origin or absent, Origin exactly http://<Host> (required on mutations), JSON Content-Type on mutations — and it makes loopback-anonymous `admin` live for the first time

CHOSEN (F19 phase 3b Task 1; master rulings R2, R3 and the own-origin correction, 2026-09-24). The pending panel runs in the web UI that the plain listener serves. On `knomit serve` it is same-origin with the API; knomit sets no cookies. The browser proof is what a page on ANOTHER origin cannot produce. For all requests:
- Host is exactly `localhost`, `127.0.0.1` or `[::1]`, with the port the connection ARRIVED on. That port is taken from http.LocalAddrContextKey, not config, because the bind may be 0.0.0.0 or port 0. A Host without a port means 80.
- `X-Knomit-Client: web` is present. A cross-origin page must preflight to send it, and the preflight is refused.
- Sec-Fetch-Site, when present, is `same-origin`.
- Origin, when present, is byte-exactly `http://<Host>`.

Mutations (approve/deny) must also carry:
- Origin, which is REQUIRED on a mutation;
- a Content-Type whose media type is application/json.

Each failure is a 403 whose text names the missing proof.

WHY EACH PART:
- **Host check.** This was the DNS-rebinding defence when 3b shipped: a rebound page is same-origin with ITSELF, its Origin matches its own Host, and only the Host names the attacker. Since #281 it is the SECOND line. AuthMiddleware refuses a loopback peer whose Host is a foreign DNS name with 421, before any principal exists (kb/invariants/auth/loopback-anonymous-host), so such a request never reaches this gate. This gate still refuses, with 403, the Host classes that check admits: other loopback IP literals such as 127.0.0.2, a port other than the arrival port, and a Host with no port on a listener that is not on 80. Keep both. They ask different questions: 'may this request be the local user' versus 'is this the page this listener served'.
- **GET needs no Origin.** Browsers omit Origin on same-origin GET/HEAD. Requiring it would refuse the panel's own list call (reviewer B2).
- **JSON type on mutations.** An HTML form can send only urlencoded, multipart or text/plain, and never a custom header.
- **No CSRF token.** There is no ambient credential to ride: no cookie, no session. The only ambient authority is being on loopback, and the Host/Origin/header proof is exactly the evidence that the caller is the page this listener served.

OWN ORIGIN IS NEVER A CORS-ALLOWLISTED ORIGIN. The desktop allowlists `wails://localhost` and `http://wails.localhost` for CORS, and corsMiddleware now allows the X-Knomit-Client header for them so their preflights work. They still fail the Origin rule, because Chrome resolves *.localhost to loopback, so any local process on port 80 could present them. The desktop never opens the OAuth listener anyway.

PREFLIGHT ON SERVE IS 403, NOT 405. The gate runs as route middleware, before chi's method dispatch. What makes the preflight a refusal is the non-2xx status plus the absent Access-Control-Allow-Origin, and that is what the test asserts.

CONSEQUENCE TO KNOW (reviewer S7):
- Before 3b, auth.Admin was checked nowhere in production code, so 'loopback-anonymous holds admin' was inert.
- Now any local process of ANY OS user can send these headers with curl. It can approve a pending OAuth request and so turn a local foothold into a remote bearer token. Before 3b, only the socket's owner could.
- The mitigations are operator config: `[auth] loopback_default` without `admin`, or `require = true`. Either removes the browser path entirely: TestBrowserGate_AnonymousWithoutAdminRefused.
- A DNS-rebound page reaching every OTHER plain-listener route as loopback-anonymous was the wider gap filed as #281. It is closed by the AuthMiddleware Host check.
- Five TestBrowserGate_EachMissingProofRefuses rows (rebinding, rebinding GET, 127.0.0.1.nip.io, localhost., sub.localhost) now expect 421 from that earlier check, and still assert that the request stays undecided.

MISREADING TO AVOID: the proof is not authentication. It says 'a browser page this listener served is asking'; it says nothing about which human. It is the anonymous principal's admin that authorises.
