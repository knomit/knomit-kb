---
type: observation
domain: [repos, config, naming]
confidence: 0.95
sources: 0
entities: [config.DefaultRepoName, internal/config/config.go, internal/repos/manager.go, cmd/reset.go, cmd/verify.go, tools/bridge/main.go, internal/store/service.go]
refs: ['src://knomit/internal/config/config.go@841c68e', 'src://knomit/internal/repos/manager.go@841c68e', 'src://knomit/tools/bridge/claude/helpers.go@841c68e', 'src://knomit/internal/store/service.go@841c68e', 'src://knomit/internal/config/config.go@8c1b3dd', 'src://knomit/web/src/App.tsx@8c1b3dd', 'src://knomit/web/src/RepoManager.tsx@8c1b3dd']
---
# Default repo/KB named "core" (was "trunk", originally "knomit"); only the repo name, not the MCP server name or git identity

The default repository/knowledge-base name is "core". It was originally "knomit" but that collided with the product name (product=knomit AND default KB=knomit). It was first renamed to "trunk" (2026-06), then to "core" (2026-06-28) because "trunk" read as git trunk-based-development / branch terminology and was confusing.

Why "core": plain, neutral, universally-understood "the primary one"; no git/tree collision (unlike trunk/root/stem); distinct from the product name; and "-kb" suffix was rejected as redundant since every repo here IS a knowledge base. Reads cleanly everywhere: core.db, /repos/core, "the core repo".

The codebase has THREE distinct "knomit" identifiers that look the same but mean different things: (a) the default REPO/KB name — the only confusing one, now "core"; (b) the MCP SERVER name (the .mcp.json block key + the lookup cfg.MCPServers["knomit"] in tools/bridge/claude/helpers.go), which determines the mcp__knomit__* tool namespace that EVERY /knomit-* skill, hook, and template depends on — renaming it would break the whole Claude integration, so it STAYS knomit; (c) the git COMMITTER identity (knomit / knomit@local) — product identity, STAYS. Also unchanged: the binary name (cmd/root.go Use:"knomit"), NewMCPServer("knomit",…), and app-state dirs (~/.knomit, Library/Application Support/knomit).

Single source of truth: const config.DefaultRepoName in internal/config (a leaf package importable everywhere incl. the bridge), threaded through every default-repo site (manager boot opens <DefaultRepoName>.db, cmd init/reset/verify flag defaults, bridge --repo default, manager-boot tests). The repo name is derived from the .db filename basename in internal/store/service.go, so the .db filename IS the name with no logic change. The web frontend hardcodes the literal name in three production spots (App.tsx fallback, RepoManager canArchive + tooltip) — a pre-existing duplication of the Go constant, kept in sync manually. Two web tests now reference config.DefaultRepoName to stay drift-proof. No data migration: existing installs hand-rename the .db while the server is stopped (pre-release).
