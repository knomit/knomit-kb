---
type: observation
domain: [repos, config, naming]
confidence: 0.95
sources: 0
entities: [config.DefaultRepoName, internal/config/config.go, internal/repos/manager.go, cmd/init.go, cmd/reset.go, cmd/verify.go, tools/bridge/main.go, internal/store/service.go]
refs: ['src://knomit/internal/config/config.go@841c68e', 'src://knomit/internal/repos/manager.go@841c68e', 'src://knomit/tools/bridge/claude/helpers.go@841c68e', 'src://knomit/internal/store/service.go@841c68e']
---
# Default repo/KB renamed knomit→trunk; only the repo name, not the MCP server name or git identity

The default repository/knowledge-base name was changed from "knomit" to "trunk" to stop it colliding with the product name (the confusing case: product=knomit AND default KB=knomit).

Options considered (AskUserQuestion):
1. Rename the default repo/KB name ONLY — chosen.
2. Rename everything including the .mcp.json MCP server block (would change the tool namespace mcp__knomit__* → mcp__trunk__*).

Rationale: the codebase has THREE distinct "knomit" identifiers that look the same but mean different things: (a) the default REPO/KB name — the only confusing one, renamed to trunk; (b) the MCP SERVER name (the .mcp.json block key at templates/mcp.json.tmpl + the lookup cfg.MCPServers["knomit"] in tools/bridge/claude/helpers.go), which determines the mcp__knomit__* tool namespace that EVERY /knomit-* skill, hook, and template depends on — renaming it would break the whole Claude integration, so it STAYS knomit; (c) the git COMMITTER identity (knomit / knomit@local in internal/store/repo.go + remote_reconcile_rebase.go) — product identity, STAYS. Also unchanged: the binary name (cmd/root.go Use:"knomit"), NewMCPServer("knomit",…) in internal/mcp/server.go, and app-state dirs (~/.knomit, Library/Application Support/knomit).

The choice: introduce a single source of truth const config.DefaultRepoName="trunk" in internal/config (a leaf package importable everywhere incl. the bridge) and thread it through every default-repo site (manager boot opens <DefaultRepoName>.db, cmd init/reset/verify flag defaults, bridge --repo default, and all manager-boot tests). The repo name is derived from the .db filename basename in internal/store/service.go, so renaming knomit.db→trunk.db auto-derives "trunk" with no logic change. No data migration: existing installs hand-rename knomit.db→trunk.db.
