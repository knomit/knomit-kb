---
kind: pragmatic
type: policy
domain: [web, routing, spa]
confidence: 0.95
sources: 0
entities: [Server.Handler, newSPAHandler, StaticHandler, APIBase]
refs: ['src://knomit/internal/web/server.go@307b67d']
---
# Handler mount order: /git → /docs → /api/v1 → /assets/* → /* SPA fallback

Server.Handler() mounts paths in this strict order (chi matches in registration order): (1) /git — optional smart git HTTP if s.GitHandler != nil; (2) /docs — Swagger UI for the OpenAPI spec; (3) APIBase (= /api/v1) — the HAL API router mounted via r.Mount; (4) /assets/* — static embedded UI bundle via StaticHandler(); (5) /* — SPA catch-all via newSPAHandler(staticHandler) which serves index.html for any unmatched path (so client-side routes survive a page refresh). Order matters: /api/v1/* and /assets/* must be matched BEFORE the catch-all /* SPA handler or it intercepts everything. Anti-pattern: registering the SPA fallback before the API routes — every API call would return index.html.
