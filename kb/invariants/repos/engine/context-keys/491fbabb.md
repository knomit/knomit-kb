---
kind: pragmatic
type: policy
domain: [repos, engine, context, expert]
confidence: 0.9
sources: 1
entities: [declaredContextKeys, contextFor, Ontology.ContextSpec, reason_run, Knomit-Trace, TestReason_UndeclaredContextKeyNeverSent]
refs: ['src://7b4887ce51d9/internal/repos/engine.go@69be5c9863582e2b4d82914b68bf169a1af912b1:449cb5a15b6ae31ee6ea4e05969573492713c0f0', 'src://7b4887ce51d9/internal/repos/engine_reason.js@69be5c9863582e2b4d82914b68bf169a1af912b1:6795f5046cccf2c6bd99fbb07be7929bf2d66656', 'src://7b4887ce51d9/internal/mcp/reason_test.go@69be5c9863582e2b4d82914b68bf169a1af912b1:22b938c47aef7aa954c64e9b42ff760e0bbafec9']
---
# The engine puts reason_run into an answer fact's context ONLY when the ontology declares that key on the fact's topic walk — no topic declares it by default and there is no root context block — otherwise the fact carries no context and is found by its Knomit-Trace trailer (the run key) and refs

fact.ValidateContext refuses every key a topic's ontology walk does not declare, and knomit has no root-level context block; the real KBs declare none. So Engine.Run passes the script a frozen global `ontology.context`: for every declared topic path (lowercased, '/'-joined, children included) the keys Ontology.ContextSpec allows there (parents' declarations folded in). reason.js resolves the answer's topic/category against it by deepest declared prefix and adds context {reason_run} only if the key is in that list. It never sends an undeclared key and never retries on a refusal. Every commit of the run carries Knomit-Trace: <reason_run> (store.WithTrailers set once by the host), so the facts stay findable with git log --grep='^Knomit-Trace: reason-…' whatever the ontology says.

Consequence: a KB that wants engine facts queryable by context declares reason_run (type string) under the relevant topic. Misreading to avoid: an answer fact with no context is not a failed write or an unidentified fact.
