---
kind: pragmatic
type: policy
domain: [integrations, claude-code, bridge, skills]
confidence: 0.95
sources: 1
entities: [tools/bridge/skills/templates, knomit-bridge claude init, mergeClaudeMd, blockMarkerCurrent, skills.FS]
motifs: [fix-stranded-in-repo, no-marker-no-detection]
refs: ['src://7b4887ce51d9/tools/bridge/skills/skills.go@629d0315068ffec8b3b930d1acdeff295d2b88e6:042e87f88792371db0e308c66331311d7a55f923', 'src://7b4887ce51d9/tools/bridge/claude/init.go@629d0315068ffec8b3b930d1acdeff295d2b88e6:1764ddd14cac812cb91ddbe0f8faa2beccd2aff9', 'src://7b4887ce51d9/tools/bridge/claude/init_merge.go@629d0315068ffec8b3b930d1acdeff295d2b88e6:1efb7146a89a5e9d6266d4171519f48cefd22f56', 'https://github.com/knomit/knomit/pull/238']
---
# A skill fix does not reach a running agent: skills are EMBEDDED in the bridge binary, so a template change needs a rebuild, an init and a restart — and NOTHING signals that an installed skill is stale

Demonstrated the hard way on 2026-09-21. A rule forbidding a behaviour was correct in the repo while an agent kept doing it, because that machine's `kb` was three days old and its embedded set had TEN skills, not eleven — the skill carrying the rule did not exist in it at all.

The chain is: templates live in `tools/bridge/skills/templates/`, are embedded in the bridge binary by `//go:embed`, and are written into a project's `.claude/skills/` by `knomit-bridge claude init`, which OWNS and overwrites that directory. So a fix reaches an agent only after rebuild + init + agent restart.

THE CLAUDE.md BLOCK SOLVES EXACTLY THIS AND SKILLS DO NOT. `mergeClaudeMd` returns an existing block unchanged when its marker matches the template's, so bumping `<!-- knomit:integration vN -->` makes a stale block detectable and replaceable in place. Skills have no equivalent, so a stale one is invisible to everyone — including the agent running it, which has no way to know its instructions are out of date.

This is a DISTRIBUTION property and it outlives whatever rule the skill happens to carry.

DIAGNOSE IT: `strings $(which kb) | grep -c "<a phrase from the new skill>"`. Zero means the binary predates the change, whatever the repo says.
