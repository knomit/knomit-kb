---
type: observation
domain: [auth, web, testing]
confidence: 0.95
sources: 1
entities: [httptest.NewRequest, RemoteAddr, isLoopback, fromLoopback, withoutAuthForTests, authDisabled, writeGate, AuthMiddleware, mcpRoutePattern, APIBase, NewAPIRouter]
motifs: [test-default-diverges, tempting-fix-widens-boundary]
refs: ['src://7b4887ce51d9/internal/web/auth_middleware.go@e4158f1fd9fc4e2fee5a3b2bf44ed358511adb43:8dc48207941b3ccbcadccb284305e61238fcf387', 'src://7b4887ce51d9/internal/web/auth_testaddr_test.go@e4158f1fd9fc4e2fee5a3b2bf44ed358511adb43:f9a0305371aaf6c4642423931603979829d7551a', 'src://7b4887ce51d9/internal/web/auth_disable_test.go@e4158f1fd9fc4e2fee5a3b2bf44ed358511adb43:dfb3bd798e7fd45cf73745c5183f7ac0920c52f6', 'src://7b4887ce51d9/internal/web/readonly.go@e4158f1fd9fc4e2fee5a3b2bf44ed358511adb43:db1c242d3824cd1767059cd3b68514ff4f1e210c']
---
# httptest.NewRequest's RemoteAddr is 192.0.2.1:1234 — documentation space, NOT loopback — so a mutating request built with it carries no principal and the write gate refuses it

`httptest.NewRequest` sets `RemoteAddr` to `192.0.2.1:1234`. That is TEST-NET-1 documentation space and is deliberately NOT loopback, so `AuthMiddleware` attaches no principal to such a request, and `writeGate` then refuses any mutating one with 403 "Permission denied". Adding the write gate turned 353 pre-existing tests in `internal/web` red for exactly this reason and nothing else.

THERE ARE TWO SANCTIONED FIXES and it matters which one you reach for.

FORM A, `fromLoopback(req)` — put `127.0.0.1:1` on the request, so it runs as the anonymous loopback principal with the default set. This is the DEFAULT choice: use it whenever the test does not care about auth. 155 of the 156 collateral sites took this form.

FORM B, `withoutAuthForTests(t, s)` — set the UNEXPORTED `Server.authDisabled`, which makes `AuthMiddleware` attach the anonymous principal whatever the address is and makes `writeGate` and the MCP permission gate no-ops. Only when Form A CANNOT work, which is narrower than it sounds. Two cases: the test ASSERTS on a non-loopback RemoteAddr (changing it breaks the thing under test), or the test drives `NewAPIRouter()` DIRECTLY instead of `Handler()`.

THAT SECOND CASE IS THE NON-OBVIOUS ONE. `mcpRoutePattern` — which exempts the MCP dispatch routes from method-based gating, because `initialize` and `knomit_bind` are themselves POSTs — is anchored with `"^" + regexp.QuoteMeta(APIBase)`. In production the API router is always mounted under `/api/v1`, so `r.URL.Path` carries the prefix and the exemption matches. A test that calls `NewAPIRouter()` directly produces paths with NO prefix, the anchor misses, and an MCP POST gets method-gated. That is a property of the test harness, not a production hole — but it looks exactly like one until you check where the router was mounted.

THE FIX THAT MUST NEVER BE TAKEN: widening `isLoopback` beyond 127/8 and ::1. It is tempting because it makes every one of these failures vanish at once, and it would hand every caller on the LAN the anonymous principal's full permission set. `authDisabled` is unexported and written only from `_test.go` for the same reason — an exported field there would be an off switch for the whole permission layer.
