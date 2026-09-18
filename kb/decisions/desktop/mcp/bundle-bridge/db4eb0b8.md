---
type: observation
domain: [desktop, mcp, integrations, build, bridge]
confidence: 0.95
sources: 0
entities: [tools/desktop/app.go, tools/desktop/bundled.go, installBridgeTool, bridgeExecName, tools/bridge/main.go, Makefile, config.Config.Home, kb, knomit-bridge]
refs: ['src://knomit/Makefile@c83980f', 'src://knomit/tools/desktop/app.go@c83980f', 'src://knomit/tools/bridge/main.go@c83980f', 'src://knomit/internal/config/config.go@c83980f', 'src://7b4887ce51d9/tools/desktop/bundled.go@b3d026204c2dbb1b3ea0d9cf0b1474560a6f1d49:8734d73785953f183a9ac19175efd4d48727053b', 'https://github.com/knomit/knomit/pull/218', 'kb://3ec012f5b4d2/kb/decisions/tools/bridge/rename-bridge-binary-to-kb/1f2bc3e7.md']
---
# macOS .app ships the bridge executable `kb` and symlinks it to <home>/bin on launch for stable MCP wiring

How the packaged macOS desktop app makes the knomit-bridge MCP integration usable. (Executable name: `kb` since PR #218, 2026-09-18 — it was `knomit-bridge` before; `knomit-bridge` remains the program's descriptive name.)

**Context:** the bridge (tools/bridge) is a pure-Go stdio↔HTTP MCP adapter that stdio-only clients (Claude Code/Desktop, VS Code) launch as a subprocess; it discovers the running server via the server.json lockfile and proxies MCP. The desktop app already runs the server in-process and writes server.json, but never shipped the bridge — so MCP clients had nothing to launch.

**Options considered:** (a) bundle the binary only, clients reference /Applications/Knomit.app/Contents/MacOS/kb by full path; (b) bundle + on launch install a stable symlink at <home>/bin/kb; (c) bundle + symlink + auto-write a client's MCP config. **Chosen: (b).** (a) breaks the moment the .app moves or updates (the in-bundle path changes); (c) is too opinionated — it must guess repo/source/profile and which client, and MCP config is project-specific (the repo's own .mcp.json passes --repo/--lens). (b) gives a fixed command path that survives app moves/updates while leaving the per-project config to the user / `kb claude init`.

**Implementation:** Makefile desktop-app-macos builds the bridge (plain `go build`, no CGO/-tags — bridge has no onnx/tokenizers/graphqlite deps) into Contents/MacOS/kb next to knomit-desktop. On launch the desktop app calls installBridgeTool(cfg.Home) (tools/desktop/bundled.go, `bridgeExecName = "kb"`): resolves os.Executable() (EvalSymlinks), finds `kb` in the same dir, and idempotently refreshes <home>/bin/kb (home = config.Home, default ~/.knomit, overridable via KNOMIT_HOME) — a symlink on macOS, a real copy on Linux/Windows (placeTool). Since PR #218 it first best-effort removes a leftover <home>/bin/knomit-bridge (and .exe) from a pre-rename install, in both symlink and copy shape; a failure there is logged and never aborts the install. Best-effort overall: failures are logged, startup continues; skipped when no bridge sits next to the exe (e.g. `go run` in dev — note that in that path the legacy removal still runs, so a developer running from source loses an old install without a replacement). MCP configs then use `~/.knomit/bin/kb` as the command.
