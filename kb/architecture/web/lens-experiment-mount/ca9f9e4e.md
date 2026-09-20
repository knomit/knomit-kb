---
type: observation
domain: [web, mcp, lens, routing, experiments]
confidence: 0.95
sources: 1
entities: [LensExperimentMiddleware, '/lenses/{lens}/experiments/{name}/mcp', paramClass, TestRouter_NoStaticSiblingsOfUserNamedParams, NewBindingOfLensOnExperiment, experimentReconnectURL, SessionScoped]
motifs: [endpoint-is-the-selection, mirrored-routes-diverge]
refs: ['kb://3ec012f5b4d2/kb/architecture/web/096bb34b.md', 'kb://3ec012f5b4d2/kb/invariants/mcp/session-binding/handle-is-the-only-router/5dd2cf17.md', 'kb://3ec012f5b4d2/kb/decisions/experiments/lens/6c6692e6.md', 'kb://3ec012f5b4d2/kb/decisions/experiments/mcp-surface/e0064db5.md', 'src://7b4887ce51d9/internal/web/lens_middleware.go@55cd390dcdfbad97f9d5f6774cfab66ff783bd2e:1b4006973fd9e7c0456e0b03096fd7521046a718', 'src://7b4887ce51d9/internal/web/router.go@55cd390dcdfbad97f9d5f6774cfab66ff783bd2e:de53c238f343196cf3644213c855c7e9c37d830f', 'src://7b4887ce51d9/internal/web/router_static_children_test.go@55cd390dcdfbad97f9d5f6774cfab66ff783bd2e:09d920ff5e19cb523f62e6fb8851bcd94c902c10', 'src://7b4887ce51d9/internal/mcp/experiment.go@55cd390dcdfbad97f9d5f6774cfab66ff783bd2e:534c79e989af8f19d28a648a71e944af3676c757', 'https://github.com/knomit/knomit/pull/236']
---
# The lens mount carries an experiment as a URL SEGMENT because a handle is refused there on presence — an unknown experiment is a 404 rather than a heal, and `open` on any URL-scoped mount returns the URL to reconnect on

A lens pins a branch per member, so it has no single branch segment to carry an experiment the way `/repos/{repo}/branches/exp:<name>/mcp` does. It gets its own: `/lenses/{lens}/experiments/{name}/mcp`, which re-pins ONLY the write member.

WHY A URL SEGMENT AND NOT PER-SESSION STATE. The lens MCP mount is URL-scoped, so `repos.SessionScoped` is false and a `binding` argument is refused ON PRESENCE, empty string included (rule 3 of handle-is-the-only-router). There is therefore no per-call value that could name the caller, and the only ambient key left is Mcp-Session-Id — the key the 2026-09-17 incident removed. Putting the experiment in the path makes it part of the endpoint's identity, as the branch segment already is, and keeps the server stateless about who is calling.

A 404, NOT A HEAL, when the experiment is unknown or this instance may not write it. The session-scoped mount heals because the caller is mid-session and its call must still run; here the experiment IS the endpoint, so serving a different branch than the URL names would be the silent misroute the design exists to remove. The two rules differ because the question differs.

MIRRORED SURFACE, applied deliberately rather than discovered as drift (architecture/web/096bb34b). The new middleware sets the SAME two context values LensMiddleware does — the Binding and the write repo as the context RepoInstance. The repo mount needs no middleware at all, because BindingFromContext synthesizes the lens-of-one from the RepoInstance plus the {branch} segment; that asymmetry is deliberate, so both mounts are tested rather than made identical.

TWO CONSEQUENCES FOR ANYONE ADDING ROUTES HERE. (1) `{name}` is classified userNamed in paramClass: experiment names are chosen by whoever calls `open`, so NO static route may ever sit beside it — TestRouter_NoStaticSiblingsOfUserNamedParams fails an unclassified subtree on purpose rather than defaulting either way. (2) Because a URL-scoped mount cannot move the caller, `knomit_experiment open` there creates the experiment and RETURNS the reconnect URL, stating plainly that the live session is not inside it; an agent told only "created" keeps writing to the agent branch believing otherwise. The prefix is spelled in internal/mcp because internal/web imports it and the dependency cannot run the other way, so the tests CALL the URL they were handed rather than string-matching it.
