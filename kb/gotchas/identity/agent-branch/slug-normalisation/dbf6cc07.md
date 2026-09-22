---
type: observation
domain: [identity, git, branches, fleet]
confidence: 0.95
sources: 1
entities: [app.AgentSlug, agentBranch, sanitizeHostname, internal/app/identity.go, agent_branch_owner]
motifs: [assumed-upstream-normalisation]
refs: ['src://7b4887ce51d9/internal/app/identity.go@f814273519db70f7c1afc7b07b4708d3d77e9801:0b55d36ae486ed9d44002928afd8cd35dc312f45', 'kb://3ec012f5b4d2/kb/decisions/identity/agent-branch-from-key-fingerprint/3ee0414b.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/agent-branch-divergence-is-by-design/5478e46e.md', 'https://github.com/knomit/knomit/pull/247']
---
# sanitizeHostname does NOT lowercase or map dots — it only replaces the chars git rejects in a ref, so `mindev.local` survives whole into the agent branch

`agentBranch(fp)` builds `"agent/" + sanitizeHostname(hostname) + "-" + fp`. sanitizeHostname replaces exactly the characters git rejects in a ref name — space, tilde, caret, colon, question mark, asterisk, open bracket, backslash — and falls back to "local" on an empty hostname. It does NOT touch dots, underscores, or uppercase letters.

So a macOS host gives `agent/mindev.local-8ef0cd32`, dot intact. Anything deriving a key, slug, directory name or URL segment from the branch and assuming sanitizeHostname already normalised it will carry that dot (and any uppercase or underscore) straight through.

As of F01 the ONLY place a branch name is normalised to `[a-z0-9-]` is `app.AgentSlug(branch)` (identity.go:142). It lowercases FIRST and strips the `agent/` prefix second — that order is load-bearing, because strings.TrimPrefix is case-sensitive, so trimming first would leave "Agent/host-abc" carrying its prefix and slugging to "agent-host-abc", a different key for the same branch. It then maps every remaining rune outside `[a-z0-9]` to `-`, collapses runs, and trims. `agent/mindev.local-8ef0cd32` becomes `mindev-local-8ef0cd32`.

It is exported precisely because skills and the fleet executor must compute the same slug this process does — two derivations that disagreed would not error, they would address two different categories and one side's messages would never be read.

AgentSlug returns "" for an empty branch and for one with no `[a-z0-9]` rune at all. A caller building a path such as `inbox/<slug>/` must treat "" as invalid and refuse, rather than address the parent directory.

What this does NOT mean: do not "fix" sanitizeHostname to lowercase or strip dots. Its output is a git ref component, the branch name is already recorded in agent_branch_owner and in every existing branch, and changing it renames live agent branches. Normalise at the point of use, in AgentSlug.

Shipped in PR #247, merged to dev as f8142735. The ref below is pinned at the merge commit and the claim was re-read there.
