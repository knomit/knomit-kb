---
type: observation
domain: [repos, engine, script, expert]
confidence: 0.9
sources: 1
entities: [engineHost, engineHost.functions, toolHost, Engine.Run, engine_reason.js, fact.RunScriptResult, TestEngine_HostMapIsExactlySixNoExec]
motifs: [capability-by-host-map]
refs: ['kb://3ec012f5b4d2/kb/invariants/repos/triggers/recipes/exec/dc5951b0.md', 'kb://3ec012f5b4d2/kb/architecture/repos/triggers/script-host/44ca7482.md', 'src://7b4887ce51d9/internal/repos/engine_host.go@69be5c9863582e2b4d82914b68bf169a1af912b1:b9f5de770c910fce84638da6c877140bad2884ca', 'src://7b4887ce51d9/internal/repos/engine.go@69be5c9863582e2b4d82914b68bf169a1af912b1:449cb5a15b6ae31ee6ea4e05969573492713c0f0', 'src://7b4887ce51d9/internal/repos/trigger_script.go@69be5c9863582e2b4d82914b68bf169a1af912b1:13ab8060239aaa46913201e850d3f594f642b484', 'src://7b4887ce51d9/internal/repos/engine_test.go@69be5c9863582e2b4d82914b68bf169a1af912b1:1b310d1c6f4f9e63512aff9294d072bce433f7c4']
---
# The reasoning engine's host object is a THIRD host map of exactly six names — query, explain, learn (the script host's own toolHost closures), incoming, experiment, model — with no exec, emit, push, run, update or retract; reason.js is embedded in internal/repos with a <home>/engine/reason.js override

internal/repos/engine_host.go builds the engine tier's `knomit` object per run: query, explain and learn are the SAME closures the `do: script` and recipe tiers use (toolHost, split out of scriptHost in trigger_script.go so all three tiers share one implementation and one set of private/artifact-path refusals); incoming, experiment and model are new. It runs on the trigger sandbox through fact.RunScriptResult (same frozen globals, interrupt and host binding as RunScript, but returning the completion value).

Why no exec: the model is reached by a host-made HTTP call (engine_model.go), so nothing in the engine starts a process; the exec invariant (kb/invariants/repos/triggers/recipes/exec) still holds — os/exec stays in the recipe tier only. No update/retract: the engine only adds answer facts into its own experiment. experiment has open, rollback and list but NOT commit: accepting the answer is the asker's (knomit_experiment commit).

Where the script comes from: the embedded internal/repos/engine_reason.js (go:embed), unless <home>/engine/reason.js exists, which replaces it (development and gating tests). It is NOT resolved from a KB's .knomit/ like a recipe: the engine must run on subscriptions and held-out KBs, and a KB's own copy deciding its answers is a hazard for a measured model.

Consequence: a new engine host function goes in engineHost.functions() and TestEngine_HostMapIsExactlySixNoExec must change with it; a function that starts a process does not belong here. Misreading to avoid: 'reuses the script host' does NOT mean the engine is a recipe or a trigger script — it has its own on-demand entry (Engine.Run / knomit_reason) and no trigger starts it.
