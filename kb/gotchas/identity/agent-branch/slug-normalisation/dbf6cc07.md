---
type: observation
domain: [identity, git, branches, fleet]
confidence: 0.95
sources: 1
entities: [app.AgentSlug, agentBranch, sanitizeHostname, internal/app/identity.go, agent_branch_owner]
motifs: [assumed-upstream-normalisation]
refs: ['src://7b4887ce51d9/internal/app/identity.go@a7bed6733bd534cbc30f181e12c3d9612559432d:e9239c524d5dfeb140611ea7c831680c5f59329d', 'kb://3ec012f5b4d2/kb/decisions/identity/agent-branch-from-key-fingerprint/3ee0414b.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/agent-branch-divergence-is-by-design/5478e46e.md']
---
# sanitizeHostname does NOT lowercase or map dots — it only replaces the chars git rejects in a ref, so `mindev.local` survives whole into the agent branch

`agentBranch(fp)` builds `"agent/" + sanitizeHostname(hostname) + "-" + fp`. sanitizeHostname replaces exactly the characters git rejects in a ref name — space, tilde, caret, colon, question mark, asterisk, open bracket, backslash — and falls back to "local" on an empty hostname. It does NOT touch dots, underscores, or uppercase letters.

So a macOS host gives `agent/mindev.local-8ef0cd32`, dot intact. Anything deriving a key, slug, directory name or URL segment from the branch and assuming sanitizeHostname already normalised it will carry that dot (and any uppercase or underscore) straight through.

As of F01 the ONLY place a branch name is normalised to `[a-z0-9-]` is `app.AgentSlug(branch)`: it strips the `agent/` prefix first so the separator does not survive as a hyphen, lowercases, maps every remaining rune outside `[a-z0-9]` to `-`, collapses runs, and trims. `agent/mindev.local-8ef0cd32` becomes `mindev-local-8ef0cd32`. It is exported precisely because skills and the fleet executor must compute the same slug this process does — two derivations that disagreed would not error, they would address two different categories and one side's messages would never be read.

What this does NOT mean: do not "fix" sanitizeHostname to lowercase or strip dots. Its output is a git ref component, the branch name is already recorded in agent_branch_owner and in every existing branch, and changing it renames live agent branches. Normalise at the point of use, in AgentSlug.

Written during branch `feat/signal-type` (F01); the identity.go ref below pins the sanitizeHostname half, which is unchanged at HEAD. AgentSlug is new on that branch and its ref must be pinned to the merge commit before this fact leaves the experiment.
