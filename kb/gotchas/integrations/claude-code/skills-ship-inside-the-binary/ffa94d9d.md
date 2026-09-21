---
kind: pragmatic
type: policy
domain: [integrations, claude-code, bridge, skills]
confidence: 0.95
sources: 1
entities: [tools/bridge/skills/templates, knomit-bridge claude init, mergeClaudeMd, blockCurrent, blockMarkerCurrent, skills.FS, skills_test, CLAUDE-md-block.txt]
motifs: [fix-stranded-in-repo, no-marker-no-detection, generated-output-edited]
refs: ['src://7b4887ce51d9/tools/bridge/skills/skills.go@629d0315068ffec8b3b930d1acdeff295d2b88e6:042e87f88792371db0e308c66331311d7a55f923', 'src://7b4887ce51d9/tools/bridge/claude/init.go@629d0315068ffec8b3b930d1acdeff295d2b88e6:1764ddd14cac812cb91ddbe0f8faa2beccd2aff9', 'src://7b4887ce51d9/tools/bridge/claude/init_merge.go@629d0315068ffec8b3b930d1acdeff295d2b88e6:1efb7146a89a5e9d6266d4171519f48cefd22f56', 'src://7b4887ce51d9/tools/bridge/claude/templates/CLAUDE-md-block.txt@629d0315068ffec8b3b930d1acdeff295d2b88e6:61c7954f30a42d446a6e6482208be822bbf80a8a', 'https://github.com/knomit/knomit/pull/238']
---
# Skills and the CLAUDE.md block are GENERATED: the canonical source is the bridge's templates, a fix reaches a running agent only after rebuild + init + restart, and nothing signals that an installed skill is stale

TWO GENERATED ARTIFACTS, one canonical source each, and one of them has no staleness signal at all.

WHERE THEY COME FROM. Skill templates live in `tools/bridge/skills/templates/<name>/SKILL.md`, are embedded in the bridge binary by `//go:embed`, and are written into a project's `.claude/skills/` by `knomit-bridge claude init`, which OWNS and overwrites that directory. A skill authored only in `.claude/skills/` is therefore not embedded, reaches no other machine, and vanishes on the next init; `skills_test` asserts the exact list, so a new one must be registered at the source.

A FIX DOES NOT REACH A RUNNING AGENT until rebuild + init + agent restart. Demonstrated the hard way on 2026-09-21: a rule forbidding a behaviour was correct in the repo while an agent kept doing it, because that machine's `kb` was three days old and its embedded set had TEN skills, not eleven — the skill carrying the rule did not exist in it at all.

THE CLAUDE.md BLOCK SOLVES THIS AND SKILLS DO NOT. `mergeClaudeMd` returns an existing block UNCHANGED when its marker equals the template's (`case blockCurrent`), so changing the block's text without bumping `<!-- knomit:integration vN -->` leaves every install already on that version with the old wording permanently — and the count in its first line ("Eleven `/knomit-…` slash commands") is part of that text and drifts the same way. That marker is what makes a stale block detectable and replaceable in place. Skills have no equivalent, so a stale skill is invisible to everyone, including the agent running it, which has no way to know its instructions are out of date.

ONE MARKER, NOT TWO: only the claude template carries it. The antigravity template is a companion file with no block markers, so a change there needs no bump and must not be described as if it did.

The distribution property outlives whatever rule a skill happens to carry.

DIAGNOSE IT: `strings $(which kb) | grep -c "<a phrase from the new skill>"`. Zero means the binary predates the change, whatever the repo says.
