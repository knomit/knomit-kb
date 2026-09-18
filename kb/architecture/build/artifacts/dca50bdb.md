---
type: reference
domain: [architecture, build, artifacts, documentation]
confidence: 0.9
sources: 1
entities: [knomit, kb, knomit-bridge, knomit-desktop, knomit-okf, fetchlibs, calibrate, drone]
refs: ['src://knomit/main.go', 'src://knomit/tools/bridge/main.go', 'src://knomit/tools/desktop/main.go', 'https://github.com/knomit/knomit/pull/218', 'kb://3ec012f5b4d2/kb/decisions/tools/bridge/rename-bridge-binary-to-kb/1f2bc3e7.md']
---
# knomit ships three user-facing binaries: knomit (server), kb (the bridge), knomit-desktop

The build produces three user-facing binaries. **knomit** (main.go) is the core HTTP server: REST /api/v1 + the MCP endpoint + the embedded React web UI + optional git smart-HTTP; CGO build (ONNX Runtime; GraphQLite was removed in PR #28 and is no longer linked). **kb** (tools/bridge/main.go; the executable was named `knomit-bridge` until PR #218 on 2026-09-18, and `knomit-bridge` is still the program's descriptive name in its help header, User-Agent and log file) is a pure-Go stdio<->HTTP MCP proxy that MCP clients spawn, and also hosts Claude Code and Antigravity scaffolding/hooks via `kb claude init`/`kb claude hook` and `kb antigravity init`. **knomit-desktop** (tools/desktop/main.go) is a Wails v3 tray/desktop app that boots the server in-process on a loopback port (default 19278) and shows the web UI in a native webview — packaged as Knomit.app on macOS, a binary + .desktop launcher on Linux (headless, no systray). The web UI (web/) is built to static assets and embedded into knomit and knomit-desktop; there is no separately deployed frontend. Also shipped alongside: knomit-okf (OKF export CLI). Build-only, not shipped: fetchlibs (downloads native libs), calibrate (derives per-model retrieval thresholds), drone (runs plans unattended).
