---
type: observation
domain: [desktop, mcp, integrations, build, bridge]
confidence: 0.95
sources: 0
entities: [tools/desktop/app.go, tools/desktop/bridge.go, tools/bridge/main.go, Makefile, config.Config.Home, knomit-bridge]
refs: ['src://knomit/Makefile@c83980f', 'src://knomit/tools/desktop/app.go@c83980f', 'src://knomit/tools/bridge/main.go@c83980f', 'src://knomit/internal/config/config.go@c83980f']
---
# macOS .app ships knomit-bridge and symlinks it to <home>/bin on launch for stable MCP wiring

How the packaged macOS desktop app makes the knomit-bridge MCP integration usable.

**Context:** knomit-bridge (tools/bridge) is a pure-Go stdio↔HTTP MCP adapter that stdio-only clients (Claude Code/Desktop, VS Code) launch as a subprocess; it discovers the running server via the server.json lockfile and proxies MCP. The desktop app already runs the server in-process and writes server.json, but never shipped the bridge — so MCP clients had nothing to launch.

**Options considered:** (a) bundle the binary only, clients reference /Applications/Knomit.app/Contents/MacOS/knomit-bridge by full path; (b) bundle + on launch install a stable symlink at <home>/bin/knomit-bridge; (c) bundle + symlink + auto-write a client's MCP config. **Chosen: (b).** (a) breaks the moment the .app moves or updates (the in-bundle path changes); (c) is too opinionated — it must guess repo/source/profile and which client, and MCP config is project-specific (the repo's own .mcp.json passes --repo/--source/--profile). (b) gives a fixed command path that survives app moves/updates while leaving the per-project config to the user / `knomit-bridge claude init`.

**Implementation:** Makefile desktop-app-macos builds the bridge (plain `go build`, no CGO/-tags — bridge has no onnx/tokenizers/graphqlite deps) into Contents/MacOS/knomit-bridge next to knomit-desktop. On launch the desktop app calls installBridgeSymlink(cfg.Home): resolves os.Executable() (EvalSymlinks), finds knomit-bridge in the same dir, and idempotently refreshes a symlink at <home>/bin/knomit-bridge (home = config.Home, default ~/.knomit, overridable via KNOMIT_HOME). Best-effort: failures are logged, startup continues; skipped when no bridge sits next to the exe (e.g. `go run` in dev). MCP configs then use `~/.knomit/bin/knomit-bridge` as the command.
