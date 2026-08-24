---
type: observation
domain: [claude-code, skills, knomit, integrations]
confidence: 0.95
sources: 1
entities: [/knomit-bootstrap, knomit-bootstrap, /knomit-remember, .claude/skills/knomit-bootstrap/SKILL.md, tools/bridge/claude/templates/skills/knomit-bootstrap/SKILL.md, tools/bridge/claude/templates/CLAUDE-md-block.txt, tools/bridge/claude/init_test.go, fact.Ontology.ValidatePath, internal/fact/ontology_default.yaml, internal/fact/ontology_code.yaml]
refs: ['src://7b4887ce51d9/.claude/skills/knomit-bootstrap/SKILL.md@0478b099c5764e01283cfaec8429c941b570ae46:51939413df76df6088ea2bc73c955322a5c79337', 'src://7b4887ce51d9/internal/fact/ontology_default.yaml@58d3df2280d851ae9d0b54c7d6b5128e37148feb:949ca99cd0c303d129fd0cb9c77778fc54f240a6', 'src://7b4887ce51d9/internal/fact/ontology_code.yaml@58d3df2280d851ae9d0b54c7d6b5128e37148feb:6c57699f000b55117d95cc6a84eb34fc56a2cb8e', 'src://7b4887ce51d9/internal/mcp/learn.go@58d3df2280d851ae9d0b54c7d6b5128e37148feb:c03f83bde73b655ccad229d8e5dec90b826b898e', 'src://7b4887ce51d9/internal/fact/ontology.go@58d3df2280d851ae9d0b54c7d6b5128e37148feb:331a061bdc1861dafca160146b49a6a798925a7e', 'kb://3ec012f5b4d2/kb/invariants/tools/bridge/skills-deployment/e6a2465c.md']
---
# /knomit-bootstrap deleted outright, nothing folded into /knomit-remember — never used, a third of it a stale fork, and its topic list was the wrong preset

On 2026-08-15, on branch `fix/session-start-toc-and-recall-path` (commit 58d3df22), `/knomit-bootstrap` was DELETED — both `.claude/skills/knomit-bootstrap/` and `tools/bridge/claude/templates/skills/knomit-bootstrap/`. Eleven `/knomit-…` slash commands became ten.

## Options considered

1. **Delete outright, fold nothing into another skill** — CHOSEN.
2. **Delete, fold ~10 lines into /knomit-remember** as a "seeding an empty area" section carrying the strict trigger, the topic-spread checklist and the approve-before-writing gate. This was the recommendation offered; the user rejected the fold specifically.
3. **Keep and fix properly** — replace the hardcoded topic list with a pointer to the MCP server instructions, cut the duplicated ref section down to a cross-reference.
4. **Keep, minimal fix only** — correct the false topic sentence and nothing else.

## Rationale

Three independent findings, each verified before the decision:

**It had never been used.** Across all six fact DBs in `~/.knomit/repos`, no commit carried the `bootstrap <area>` moment the skill prescribed at its step 4. The only "bootstrap" moments at all — 12, in one repo — read "Documentation bootstrap: user-facing reference facts", a hand-written batch that did not go through the skill.

**A third of it had forked from /knomit-remember and drifted.** Its "Producing refs" section duplicated remember's ref section, minus the worked example, minus the "NEVER write bare paths" warning, minus the explanation of why the blob makes a citation durable. Its body-authoring guidance delegated to /knomit-remember outright. The file's own structure conceded that remember was the real skill.

**Its central claim was false in a way that broke the one job it had.** It said knomit's ontology has "fixed top-level topics (architecture, conventions, decisions, gotchas, incidents, invariants, meta)". That is the `code` preset (`internal/fact/ontology_code.yaml`). The preset actually named `default` (`internal/fact/ontology_default.yaml`) is a general-knowledge ontology — people, technology, science, society, culture, geography, history, health, philosophy, religion, business, reference, meta — overlapping on `meta` and nothing else; and a repo may supply its own via `.knomit/ontology.yaml`. Because `knomit claude init` ships skills into arbitrary repos, an agent bootstrapping a default-preset repo would draft every fact under a topic `Ontology.ValidatePath` rejects (`learn.go:234` → `ontology.go:280`, "unknown topic"). The seeding skill would fail on every fact in exactly the situation it existed for. knomit's own repo runs the code preset, which is why this never surfaced.

What remained unique — read the area, spread drafts across topics, show them before writing — is a prompt, not a skill.

## Consequence for a consumer

Do not reintroduce a bootstrap skill on the argument that "seeding an empty area needs a recipe". The recipe was written, shipped, and drew zero uses over the corpus's whole life. Seeding an empty area is done with a direct prompt plus `/knomit-remember`. If someone revives the idea, the burden is evidence of use, not plausibility.

## WHAT THIS DOES NOT MEAN

This does NOT mean the approve-before-writing gate was judged wrong — it was judged not worth a skill of its own, and the user explicitly declined to relocate it into `/knomit-remember`. Anyone who later wants that gate should propose it on its own merits, not cite this decision as precedent for it.

It also does NOT generalise to the other ten skills. The three findings here — zero measured uses, a third of the file a degraded fork, and a load-bearing false claim — were established for this skill specifically. Deleting another skill requires establishing them again for that skill.

## Non-scope

This decision covers the skill only. It does not touch `knomit_learn`, the ontology presets, or `ValidatePath`. Two existing facts still name `knomit-bootstrap`: `kb/conventions/integrations/cc-memory/confidence-calibration/9d5aea63.md` (refs the now-deleted template) and `kb/decisions/integrations/cc-memory/recall-trigger-expansion/b75a1f47.md` (names it as an entity). Both remain substantively correct; their references are now historical.
