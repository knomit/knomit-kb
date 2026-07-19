---
type: observation
domain: [mcp, lens, federation]
confidence: 0.95
sources: 0
entities: [knomit_query, knomit_explain, knomit_repos, AfterInitialize, queryResponse]
refs: ['https://github.com/knomit/knomit/discussions/8']
---
# Lens responses carry NO federation/coverage metadata (no skipped[] block) — a lens answers exactly like a single repo

**Options considered:** (a) a `skipped[]` block in query/explain responses reporting read repos skipped for lacking the queried topic (RFC revisions 1–2); (b) no per-response metadata — topic-coverage skipping is a silent internal fan-out optimization.

**Rationale (user ruling 2026-07-16):** skip metadata has no actionable value to the consumer — querying a general-knowledge repo for code returns empty today with no explanation, and the agent's next action is identical either way. Worse, per-response skip reporting leaks the underlying repos, breaking the virtual-repo abstraction the lens exists to provide, AND diverges from single-repo behavior (a repo queried for a topic it doesn't have just returns empty). Skipping repos that don't declare the topic is therefore purely an internal optimization whose observable behavior must be indistinguishable from querying them and matching nothing.

**The choice:** No `skipped[]` (or any federation metadata) in MCP responses. Coverage — which mount declares which topics — is discoverable once per session via AfterInitialize instructions and the knomit_repos discovery tool, not per response.

**Non-scope:** This does not hide the mounts themselves — alias-qualified result paths and knomit_repos deliberately expose them; the ruling is specifically against per-response coverage/skip reporting.
