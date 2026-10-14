---
type: observation
domain: [repos, engine, expert, evaluation]
confidence: 0.8
sources: 1
entities: [judge, knomit.model, modelCall, temperature, llama-server, LFM2.5-2.6B]
motifs: [unpinned-randomness-in-eval]
refs: ['kb://3ec012f5b4d2/kb/conventions/repos/engine/judgments/4d7ec12c.md', 'kb://3ec012f5b4d2/kb/decisions/repos/engine/model-call/269bd726.md', 'src://7b4887ce51d9/internal/repos/engine_reason.js@913286afe36d27be2c391f0c51a990d01ac5fa7b:bce63402236cfb2a9dbbd725c6c9808577a02727', 'src://7b4887ce51d9/internal/repos/engine_model.go@913286afe36d27be2c391f0c51a990d01ac5fa7b:1a2d3272bb5b8c47e62178cd73dbf368889f45a7']
---
# The engine sends no temperature, so every eval arm decodes with its server's default sampling: two 2.6B validation runs that differed only in a schema bound (it touched 3 of 68 chains) disagreed on 12 of 68 outcomes, with outcome-right +0.09 [0.00, +0.19] between them

reason.js judge() passes schema, max_tokens, system and user to knomit.model and never a temperature, and modelCall sends `temperature` only when the script passes one. So every judgment is decoded with the serving runtime's default sampling: llama-server's own default for the local arms, and the claude CLI's for Opus.

Measured in the Phase 2 re-run on the validation slice (68 questions, mode H, LFM2.5-2.6B Q4_K_M on llama-server). Build e82577d1 and build 913286af differ only in fix/expert-schema-bounds, which changed the outcome of 3 chains (6 truncated support replies). Yet 12 of the 68 outcomes differ between the two runs, and outcome right moved from 0.65 to 0.74 (paired difference +0.09 [0.00, +0.19]; correct +0.01 [-0.03, +0.07]).

Consequence: at n=68, run-to-run variance is of the same order as the differences between arms that a gating bar is meant to read. A bar fixed on one run inherits it. Before fixing a bar, either pass temperature 0 (or a fixed seed) through judge() for eval runs, or run each arm more than once and report the spread. Comparisons between two builds of the same arm need the same treatment.

What this does NOT mean: the difference between the two builds is not evidence that the bound improved the 2.6B. Most of it is sampling noise, because the bound changed only 3 chains.
