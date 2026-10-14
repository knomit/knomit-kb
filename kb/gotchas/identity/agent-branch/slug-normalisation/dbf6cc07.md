---
type: observation
domain: [identity, git, branches, fleet]
confidence: 0.95
sources: 1
entities: [app.newIdentity, sanitizeHostname, internal/app/identity.go, agent_branch_owner]
motifs: [assumed-upstream-normalisation]
refs: ['src://7b4887ce51d9/internal/app/identity.go@f814273519db70f7c1afc7b07b4708d3d77e9801:0b55d36ae486ed9d44002928afd8cd35dc312f45', 'kb://3ec012f5b4d2/kb/decisions/identity/agent-branch-from-key-fingerprint/3ee0414b.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/agent-branch-divergence-is-by-design/5478e46e.md', 'https://github.com/knomit/knomit/pull/247', 'src://7b4887ce51d9/internal/app/identity.go@c9c599dee475771c4226212ac0889817c6b268d4:ba7fa0481b46dfa22ee0a903b45811942f84faf5', 'https://github.com/knomit/knomit/pull/513']
---
# sanitizeHostname does NOT lowercase or map dots — it only replaces the chars git rejects in a ref, so `mindev.local` survives whole into the agent branch

`app.newIdentity(fp)` (it replaced agentBranch(fp) in PR #513) builds the identity a first boot persists: host = sanitizeHostname(hostname), agent id = host + "-" + fp, branch = "agent/" + id. sanitizeHostname replaces exactly the characters git rejects in a ref name — space, tilde, caret, colon, question mark, asterisk, open bracket, backslash — and falls back to "local" on an empty hostname. It does NOT touch dots, underscores, or uppercase letters.

So a macOS host gives `agent/mindev.local-8ef0cd32`, dot intact. Anything deriving a key, directory name or URL segment from the branch and assuming sanitizeHostname already normalised it will carry that dot (and any uppercase or underscore) straight through.

REMOVED: app.AgentSlug (PR #247 added it as the one place a branch was normalised to `[a-z0-9-]`) was deleted in PR #513: it turned a branch NAME into a category key by stripping agent/, which D18 (no identity inferred from a name) forbids, and it had no production caller. Do not reintroduce a branch-to-key helper; take an id from the persisted instance_identity instead.

What this does NOT mean: do not "fix" sanitizeHostname to lowercase or strip dots. Its output is a git ref component, the branch name is already recorded in agent_branch_owner and in every existing branch, and changing it renames live agent branches. The sanitized host is also persisted on its own (instance_identity_host), so nothing needs to be parsed back out of the branch.

Shipped in PR #247, merged to dev as f8142735; the AgentSlug half was removed in PR #513 (merge c9c599de).
