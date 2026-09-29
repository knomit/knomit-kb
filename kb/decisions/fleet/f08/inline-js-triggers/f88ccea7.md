---
type: observation
domain: [fleet, decisions, triggers, ontology]
confidence: 0.95
sources: 1
entities: [js, script, compileInlineJS, TriggerCache, recipes, R9]
refs: ['kb://3ec012f5b4d2/kb/architecture/fact/ontology/triggers/inline-js/09638689.md', 'src://7b4887ce51d9/internal/fact/triggers.go@53d102e998b25671eb2922a9ba25b064652ec6fd:e680371729faed7cbe3d77ff2e627b2e1b76cf9b', 'https://github.com/knomit/knomit/pull/348']
---
# Trigger scripts may be inline under `js:` (exclusive with `script:`), compiled with the ontology and cached by source hash; recipes stay files

User ruling R9 (2026-09-29, verbatim): "one more thing: currently to run a script you have to have a javascript file regardless if your script is a one liner or not. How about allowing inline scripts ?" → ""js" works. caching works. don't worry about older binaries, we've not release this yet. ok." → "recipes stay files"

OPTIONS CONSIDERED
- Key shape: a new key `js:` mutually exclusive with `script:` (CHOSEN); overloading `script:` to hold either a name or code (REJECTED: ambiguous; the kebab-case name rule makes a name a path).
- When the code compiles: with the ontology in compileTrigger, so bad code makes the trigger `invalid` (CHOSEN, like `if`); lazily per fire, so bad code becomes a `script-error` row on every fire (REJECTED: the code is right there; judge it with the entry).
- Caching: by source hash, so an ontology edit elsewhere recompiles nothing (CHOSEN, "caching works"); only by ontology blob (REJECTED: every unrelated edit recompiles every inline script).
- Older binaries: they mark a `js:` trigger invalid ("unknown key"); the user ruled this is not a concern ("we've not release this yet").
- Inline recipes: REJECTED ("recipes stay files"): a recipe may exec programs, and its trust tier is WHERE the file is (main, or this machine's home), which an inline ontology value would blur.

RATIONALE
One-liners such as the R2 expiry trigger (`js: "knomit.retract(change.path)"`) should not need a file; the sandbox, host API, trailers, loop guard and rate cap are the file script's, unchanged.

THE CHOICE
Merged in PR #348 (merge commit 53d102e9). Details: kb/architecture/fact/ontology/triggers/inline-js/09638689.md.

NON-SCOPE: this does not make recipes inline, and it adds no new sandbox capability.
