---
type: synthesis
domain: [integrations, hermes, memory, skills, positioning]
confidence: 0.8
sources: 1
evidence_weight: 0.7222222222222222
entities: [Hermes Agent, MEMORY.md, USER.md, skill_manage, Curator, curator.consolidate, memory.provider, prefetch, pre_llm_call, on_memory_write, sync_turn]
motifs: [lifecycle-by-disuse, bounded-not-subsumed]
refs: ['kb://3ec012f5b4d2/kb/architecture/integrations/hermes/memory-provider/c7792d3f.md', 'kb://3ec012f5b4d2/kb/architecture/integrations/hermes/skills/65e8b73b.md', 'kb://3ec012f5b4d2/kb/architecture/integrations/hermes/market/95a561c4.md']
---
# Hermes curates accumulated knowledge by BOUNDS and AGE — a byte cap that errors, a disuse clock, an opt-in LLM rewrite — and never by subsumption or provenance, which is why its own users report contradictions accumulating in both memory AND skills

Three separately-verified descriptions (all snapshots of 2026-09-13, against the Hermes docs and the GitHub API) turn out to describe ONE mechanism operating on two different stores.

THE MECHANISM. Every place Hermes accumulates durable knowledge, it is managed by a bound or a clock — never by deciding that one statement REPLACES another:

- MEMORY.md is capped at 2,200 chars and USER.md at 1,375. On overflow the memory tool ERRORS rather than compacting. There is no read operation. Both are injected as a frozen snapshot at session start.
- Agent-authored skills are aged by DISUSE: active → stale at 14 days unused → archived at 30. Nothing examines whether a skill is right, only whether it was recently used.
- The one operation that could subsume is `curator.consolidate`, and it is OFF by default — and it is LLM REWRITING rather than subsumption when on. The market snapshot records the same thing from the outside: the consolidation rule is "a prompt not code".
- `forget` leaves an internal summary behind, so removal is not removal.
- Nothing anywhere records WHY a memory or a skill exists, or what evidence produced it.

WHY THE CONSEQUENCE FOLLOWS. A cap, a disuse clock and an opt-in rewrite are all PROXIES for relevance; none is a judgement about content. So two statements that contradict each other can both be recent, both be under the cap, and both survive indefinitely. That is exactly what Hermes's own users report — "memory intentionally constrained", "contradictions accumulate" — and the skills docs report the same shape independently: skills "duplicate and contradict over time". The two stores were built separately and failed the same way, which is what makes this a mechanism rather than a complaint about one feature.

CONSEQUENCE for a knomit adapter — and this is a claim about FIT, not a demonstrated win: subsumption-on-write, provenance refs and a review gate are the knomit mechanics that address exactly this, and an agent-authored skill maps onto a pragmatic methodology/heuristic fact. Any such adapter must work WITHIN three structural constraints the sources establish: (1) `memory.provider` selects EXACTLY ONE external provider, so a knomit provider competes with mem0/honcho/supermemory for that slot — ship the per-turn context injection ALSO as a general plugin via `pre_llm_call`, which coexists with whatever provider is active; (2) recalled context must arrive via `prefetch()` or `pre_llm_call`, NOT `system_prompt_block()`, because the system prompt is frozen mid-session to preserve prefix cache; (3) `prefetch` sits on the turn path of a chat agent used from phones — budget well under 1 s (the docs' own example uses a 3 s timeout).

CARRIED FROM THE SOURCES, because the synthesis must not smooth them away:

- A provider does NOT replace MEMORY.md/USER.md. The files keep being injected and provider writes are ADDITIONAL.
- Hermes is not hookless for non-providers: `pre_llm_call` is available to any plugin.
- Hermes skills are NOT only agent-authored — most installed skills are human-authored from the hub. A round-trip should own only `created_by: agent` skills unless the user opts in.
- The market figures are a dated snapshot and partly third-party: star count is not usage, and the OpenRouter token-share claim is unverified. Nothing here rests on those numbers.
- Scope: verified against the docs as of 2026-09-13, on a project shipping breaking minors (v0.21.0→v0.21.2 shipped state.db lock/corruption bugs). Pin any adapter to a version range and re-verify before relying on an interface detail.

Each source carries the full interface signatures, frontmatter schemas and figures this summary does not reproduce — consult them before acting on any single clause here.
