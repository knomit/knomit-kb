---
type: reference
domain: [architecture, build, artifacts, documentation]
confidence: 0.9
sources: 1
entities: [knomit, knomit-bridge, knomit-desktop, fetchlibs, calibrate, drone]
refs: ['src://knomit/main.go', 'src://knomit/tools/bridge/main.go', 'src://knomit/tools/desktop/main.go']
---
# knomit ships three binaries: knomit (server), knomit-bridge, knomit-desktop

The build produces three user-facing binaries. **knomit** (main.go) is the core HTTP server: REST /api/v1 + the MCP endpoint + the embedded React web UI + optional git smart-HTTP; CGO build (ONNX Runtime + graphqlite). **knomit-bridge** (tools/bridge/main.go) is a pure-Go stdio<->HTTP MCP proxy that MCP clients spawn, and also hosts Claude Code scaffolding/hooks via `claude init`/`claude hook`. **knomit-desktop** (tools/desktop/main.go) is a Wails v3 tray/desktop app that boots the server in-process on a looknomitck port (default 19278) and shows the web UI in a native webview — packaged as Knomit.app on macOS, a binary + .desktop launcher on Linux (headless, no systray). The web UI (web/) is built to static assets and embedded into knomit and knomit-desktop; there is no separately deployed frontend. Build-only, not shipped: fetchlibs (downloads native libs), calibrate (derives per-model retrieval thresholds), drone (runs plans unattended).
