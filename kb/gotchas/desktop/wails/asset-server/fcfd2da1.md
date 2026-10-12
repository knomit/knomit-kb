---
type: observation
domain: [desktop, wails, http]
confidence: 0.9
sources: 1
entities: [apiProxy, httputil.ReverseProxy, http.Request.ContentLength, assetserver_webview.go, TestAPIProxy_BodyWithoutLengthReachesTheServer, WKWebView]
motifs: [silent-body-drop]
refs: ['src://7b4887ce51d9/tools/desktop/uiproxy.go@2104fd6d5e0fe0d6a1bdfbad4c75a973a539a62d:6ded8f9875a4f55a81b30e335072b6c57653d03a', 'src://7b4887ce51d9/tools/desktop/uiproxy_test.go@2104fd6d5e0fe0d6a1bdfbad4c75a973a539a62d:4dcd9c095bb5d2f328b7e5900008842a8e9a3354', 'https://github.com/wailsapp/wails/blob/v3.0.0-beta.28/v3/internal/assetserver/assetserver_webview.go', 'src://7b4887ce51d9/go.mod@0486a84f8041838216822ade9e29b2afbdd998ed:8e64003482fa0910ad39f0147d2a5d07793d3c71']
---
# A reverse proxy behind the Wails asset handler silently drops POST bodies unless ContentLength is set to -1 — Wails fills ContentLength only from a Content-Length header, which a webview custom-scheme POST does not send

Wails (v3.0.0-beta.28, internal/assetserver/assetserver_webview.go around L196-200) sets the request's ContentLength only by parsing a Content-Length header. A POST from a WKWebView custom-scheme page carries none, so the handler sees ContentLength == 0 with a non-empty Body. httputil.ReverseProxy sends NO body when ContentLength is 0, so the upstream saw an empty body: the first symptom was every POST /api/v1/repos answering 400 "the request body is empty".

FIX (tools/desktop/uiproxy.go, apiProxy): before proxying, `if r.ContentLength == 0 && r.Body != nil && r.Body != http.NoBody { r.ContentLength = -1 }` (-1 means unknown, so the transport sends it chunked). Pinned by TestAPIProxy_BodyWithoutLengthReachesTheServer; the sabotage (removing the line) turns it red, and the bodied POST /api/v1/repos returns 202 in the real macOS webview with the fix.

CONSEQUENCE: any handler placed in front of Wails' asset handler that forwards the request (a proxy, a recorder, a size limiter) must treat ContentLength 0 with a non-NoBody Body as "unknown", not "empty". WHAT THIS DOES NOT MEAN: it is not a ReverseProxy bug and not specific to POST (any method with a body); it was observed on macOS, and whether Windows/Linux webviews send a Content-Length is unverified, so keep the guard unconditional.
