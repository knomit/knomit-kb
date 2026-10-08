---
type: observation
domain: [fleet, mcp, skills]
confidence: 0.95
sources: 1
entities: [F08, R6, R8, .knomit/skills, SKILL.md, MCP prompts, internal/mcp/prompts.go, kb claude init]
refs: ['src://7b4887ce51d9/internal/mcp/prompts.go@ad75940b36c05b3c88d312b130e7107a37d14acd:13bc50baa5e947aefd0d389956c9999f29f7547b', 'src://7b4887ce51d9/internal/fact/skill.go@ad75940b36c05b3c88d312b130e7107a37d14acd:e4b5f5740bf7baa7843c20bacf48356dbe8b1177', 'src://7b4887ce51d9/internal/store/skills.go@ea6836909036f558f80f7c90ff4e95dcf250b693:946e1ab5a0fe5e6a9fdb7e0a07b8b847bbda5c72', 'kb://3ec012f5b4d2/kb/architecture/mcp/skills/prompts-per-binding/25fa02e5.md', 'kb://3ec012f5b4d2/kb/gotchas/store/go-git/tree-entry-modes/206d1283.md', 'https://github.com/knomit/knomit/pull/351', 'https://github.com/knomit/knomit/pull/438']
---
# Repo instructions are Agent Skills folders in .knomit/skills/<name>/SKILL.md, served by knomit over MCP as prompts; knomit installs them nowhere (R6 + R8)

**User rulings, verbatim.**

R6: "I'm struggling with the skills a bit - I mean, if the instance is not connected to a fleet they will still be available right ? And what would those skills actually do, outside of what the MCP operations will be capable in doing ?" → "I'm actually up for maybe in addition to js files in the recipes maybe add a new .knomit folder which will store prompts, prompts which will contain instructions on how to do different things - stuff that skills would teach an agent. This way, we can point an agent to "go and do that""

R8: "stop for a second, instead of "prompts" maybe we should have a folder with "skills" ?" → "ok, middle path it is - in skills format, but mcp will deliver them as promts and anyone else could do whatever they decide with them"

**Options considered.**
1. Skills as bridge-installed files: copy them into the harness's folder (.claude/skills), as `kb claude init` does for the /knomit-* skills. REJECTED: harness-specific, a copy that drifts from the repo, and unavailable to any other MCP client; R8 says knomit delivers them and "anyone else could do whatever they decide with them".
2. Prompts only: a `.knomit/prompts/*.md` folder of plain prompt files (R6's first shape). SUPERSEDED by R8: the files use the Agent Skills format instead (frontmatter name + description, Markdown body, bundled files), so a SKILL.md written for a harness works unchanged.
3. CHOSEN, the "middle path": skills format in the repo, delivered by knomit's MCP server as MCP prompts (plus the knomit_skill tool, a separate ruling).

**The choice as built (#351).** `.knomit/skills/<name>/SKILL.md` + bundled files; each skill is an MCP prompt on its repo's mount (one optional `args` argument for $ARGUMENTS); bundled text files inlined up to 256 KiB in total, binaries listed by path; malformed skills skipped with one WARN per blob. Available whether or not the instance is in a fleet.

**Non-scope.** Nothing is copied into .claude/skills or any harness folder, and `kb claude init` is unchanged. MCP resources are not used for bundled files.

SYMLINKS (#438, merge ea683690, the follow-up to #437; user ruling 2026-10-08 "we do NOT want to follow symlinks, so 404"): SKILL.md and every bundled file must be REGULAR files (or executable). A folder whose SKILL.md is a symlink is no skill, and a regular skill.md beside such a symlink is the skill. A symlinked bundled file is neither inlined nor listed. The test is store.isSystemFileMode, shared with SystemFileAt; see kb/gotchas/store/go-git/tree-entry-modes/206d1283.md.
