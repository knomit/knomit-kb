---
type: observation
domain: [web, store, git, history, provenance]
confidence: 0.95
sources: 0
entities: [CommitLogEntry, CommitLogRow, LogEntryWithTags, CommitDetailResult, commitItem, commit_log.author_name, commit_log.author_email, c.Author.Name, c.Author.Email]
refs: ['src://7b4887ce51d9/internal/engine/store/internal/vcs/fact_history.go@0610c20083fcef9e84f42ab0311a14708bcb6193:3be7372cd59f8bf0a5997e384f9e86e1729af58e', 'src://7b4887ce51d9/internal/engine/store/git/commitlog.go@0610c20083fcef9e84f42ab0311a14708bcb6193:6a211b394fd99da45f507815ab41e6c8dbde04b6', 'src://7b4887ce51d9/internal/engine/store/types.go@0610c20083fcef9e84f42ab0311a14708bcb6193:63b60548375ae420095d944b61e51d7f256e32bf', 'https://github.com/knomit/knowkb/commit/d84f9d352f986b241148a61580b0ab03a476ed14']
---
# Fact-history API exposes the commit AUTHOR (name+email, as-is) as a nested object, not committer

Decision: the fact-history / commit-list REST API will surface the git AUTHOR identity (name + email) verbatim ("as-is"), as a nested `author: {name, email}` object. The `operation` field stays separate (it is already derived from the author email's +subaddress).

**Options considered:**
1. Identity with the +operation subaddress stripped from the email (cleanest, no redundancy with `operation`).
2. Raw author email verbatim.
3. Both author and committer.
4. (shape) nested `author` object vs flat `author_name`/`author_email` fields.

**Rationale — why AUTHOR, name+email, as-is, nested:**
Real-repo investigation of knomit/knowkb (trunk origin) showed two commit shapes. Agent fact-writes: author = `<agent-id> <<agent-id>+<op>@agents.knomit.io>`, committer = bare `<agent-id>@agents.knomit.io` (drops the operation). PR merges to main: author = `knomit <k@knomit.io>` (the human who merged), committer = `GitHub <noreply@github.com>`.
- AUTHOR, not committer, is the faithful "who made it": on merges the committer is just GitHub; on agent commits the committer loses the operation tag.
- Must store the author NAME, not just parse the email: the clean identity lives in the author name (agent-id for agents, `knomit` for merges) and is NOT derivable from the email for human merges (name `knomit` != email localpart `apbogdan`). So stripping/parsing the email is lossy — store name + email as-is.
- Committer was dropped from scope: user explicitly preferred "author as-is"; committer can be added later without breaking the nested shape.
- Nested `author` object chosen over flat fields: matches GitHub's commit API shape and leaves room to add committer.

**The choice:** capture `c.Author.Name` (currently thrown away in commitEntries) alongside the already-captured `c.Author.Email`; add an `author_name` column to commit_log beside `author_email`; thread both through CommitLogRow / LogEntryWithTags / CommitDetailResult / the web commitItem view; emit JSON `author: {name, email}`. Existing repos repopulate author_name on the next commit-log rebuild (git is source of truth, DB is rebuildable cache), so no SQL data backfill is required.
