---
type: observation
domain: [repos, engine, script, expert]
confidence: 0.9
sources: 1
entities: [engineHost, engineHost.functions, toolHost, Engine.Run, engine_reason.js, fact.RunScriptResult, TestEngine_HostMapIsExactlySixNoExec]
motifs: [capability-by-host-map]
refs: ['kb://3ec012f5b4d2/kb/invariants/repos/triggers/recipes/exec/dc5951b0.md', 'kb://3ec012f5b4d2/kb/architecture/repos/triggers/script-host/44ca7482.md', 'src://7b4887ce51d9/internal/repos/engine_host.go@3462f5a42e5ede59bc23f049427b2c8edf7f1989:d0e97da24789f6ff3a1d68c1b3b057cc4204d61f', 'src://7b4887ce51d9/internal/repos/engine.go@3462f5a42e5ede59bc23f049427b2c8edf7f1989:449cb5a15b6ae31ee6ea4e05969573492713c0f0', 'src://7b4887ce51d9/internal/repos/trigger_script.go@3462f5a42e5ede59bc23f049427b2c8edf7f1989:13ab8060239aaa46913201e850d3f594f642b484', 'src://7b4887ce51d9/internal/repos/engine_test.go@3462f5a42e5ede59bc23f049427b2c8edf7f1989:766b212a6839384c3f3057e32fbb7cbe1a53def4']
---
# The reasoning engine's host object is a THIRD host map of exactly six names — query, explain, learn (the script host's own toolHost closures), incoming, experiment, model — with no exec, emit, push, run, update or retract; reason.js is embedded in internal/repos with a <home>/engine/reason.js override

internal/repos/engine_host.go builds the engine tier's `knomit` object per run: query, explain and learn are the SAME closures the `do: script` and recipe tiers use (toolHost, split out of scriptHost in trigger_script.go so all three tiers share one implementation and one set of private/artifact-path refusals); incoming, experiment and model are new. It runs on the trigger sandbox through fact.RunScriptResult (same frozen globals, interrupt and host binding as RunScript, but returning the completion value).

Why no exec: the model is reached by a host-made HTTP call (engine_model.go), so nothing in the engine starts a process; the exec invariant (kb/invariants/repos/triggers/recipes/exec) still holds — os/exec stays in the recipe tier only. No update/retract, and the engine only ADDS answer facts inside its own experiment. At 69be5c98 that was NOT true: learn was the shared toolHost.learn unguarded, so a learn before `experiment open` committed onto the asker's branch and `learn([], {retract: [...]})` deleted facts by the move form. Since 4900270a it is enforced in engineHost.engLearn: a learn while no experiment this run opened is current is refused, and any opts.retract is refused, each thrown to the script with a Warn `reason step` line carrying reason_run, before the tool is called (TestEngine_Learn_OnlyInsideItsExperimentAndNeverRetracts: the agent head does not move, no learn reaches the tools, the named fact survives). experiment has open, rollback and list but NOT commit: accepting the answer is the asker's (knomit_experiment commit).

Where the script comes from: the embedded internal/repos/engine_reason.js (go:embed), unless <home>/engine/reason.js exists, which replaces it (development and gating tests). It is NOT resolved from a KB's .knomit/ like a recipe: the engine must run on subscriptions and held-out KBs, and a KB's own copy deciding its answers is a hazard for a measured model.

Consequence: a new engine host function goes in engineHost.functions() and TestEngine_HostMapIsExactlySixNoExec must change with it; a function that starts a process does not belong here. Misreading to avoid: 'reuses the script host' does NOT mean the engine is a recipe or a trigger script — it has its own on-demand entry (Engine.Run / knomit_reason) and no trigger starts it.

Re-pinned to merge commit 3462f5a4 after PR #498 merged into expert-spike; claim re-read there and holds.
