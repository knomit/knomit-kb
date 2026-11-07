---
kind: pragmatic
type: policy
domain: [web, chat, fact-links]
confidence: 0.9
sources: 1
entities: [turnPrefer, callTarget, boundAfter, POST /facts/resolve]
refs: ['src://7b4887ce51d9/web/src/chatFactLinks.ts@be0c8d295b20de60625a673a9da73aff96f62ac3:cd23aa5ce0a925d8e8ff0fb128e933c043bfe0ac', 'src://7b4887ce51d9/web/src/chatState.ts@be0c8d295b20de60625a673a9da73aff96f62ac3:a6522b30b2f7aed189a705bb66ac1dba5b654eb4', 'src://7b4887ce51d9/web/src/ChatFactLinks.test.tsx@be0c8d295b20de60625a673a9da73aff96f62ac3:0bf199e2d33f7c74016e4a42b4a314603689a4d5', 'src://7b4887ce51d9/internal/edge/web/api/kb/handlers_facts_resolve.go@be0c8d295b20de60625a673a9da73aff96f62ac3:f05ef658bb34e32fd9fdc5e41c9454a953d3e9f6', 'kb://3ec012f5b4d2/kb/architecture/web/api/kb/facts-resolve/ff1a3ed6.md']
---
# A failed call adds nothing to the fact links' prefer list, whichever tool it was: turnPrefer skips it for both its target and a kb:// store, and keeps the binding in force; a short id two repos hold, where only a failed call named one, stays code

User decision (2026-11-07), chosen over counting every call and over excluding only failed binds. turnPrefer (chatFactLinks.ts) skips every call with isError, for both callTarget and the store of a kb:// file it named. The binding in force is kept, and a failed bind does not move it (boundAfter). Why: the server answers a commit hash only when prefer comes down to exactly one reachable repo, so a refused query naming a second reachable repo stopped a commit chip, for a repo the turn never worked in.

ACCEPTED COST: a failed call earns its repo no preference either. A short id two reachable repos hold, where only a failed call named one of them, is ambiguous and stays plain code rather than linking.

Does NOT mean failed calls are invisible. callTarget does not look at isError and is unchanged, and the activity line still counts them ('Query in second-repo' for a refused query): the rule is the prefer list's alone. Do not move it into callTarget.

Checked on a real server: a bind to a repo, then a knomit_query naming a second real repo that the server refused (is_error, in_base, target second-repo): POST /facts/resolve carried prefer [the bound repo], the chip resolved, and the activity line read 'Bind ... · Query in second-repo'.
