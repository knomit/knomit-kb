---
type: observation
domain: [identity, git, branches, fleet]
confidence: 0.95
sources: 1
entities: [app.newIdentity, sanitizeHostname, internal/app/identity.go, agent_branch_owner]
motifs: [assumed-upstream-normalisation]
refs: ['src://7b4887ce51d9/internal/app/identity.go@3e2e83b082a450210e5de0111fa55774ccb891e6:13145703058e7247bd6dc0b8c634a57d13e8b57b', 'src://7b4887ce51d9/internal/repos/fleet_identity.go@3e2e83b082a450210e5de0111fa55774ccb891e6:64d6b86d6c8fedd380df05f3c5b59b6c963d58b9', 'kb://3ec012f5b4d2/kb/decisions/identity/agent-branch-from-key-fingerprint/3ee0414b.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/agent-branch-divergence-is-by-design/5478e46e.md', 'kb://3ec012f5b4d2/kb/invariants/pki/agent-id-binding/efad10db.md', 'https://github.com/knomit/knomit/pull/247', 'https://github.com/knomit/knomit/pull/513', 'https://github.com/knomit/knomit/pull/514']
---
# sanitizeHostname LOWERCASES the hostname and replaces '+' and '@' and the chars git rejects in a ref (since PR #514: agent ids are lower case everywhere), but it still does NOT map dots, so `mindev.local` survives whole into the agent branch

`app.newIdentity(fp)` (it replaced agentBranch(fp) in PR #513) builds the identity a first boot persists: host = sanitizeHostname(hostname), agent id = host + "-" + fp, branch = "agent/" + id. Since PR #514 sanitizeHostname FIRST lowercases the hostname (agent ids must pass pki.ValidAgentID, which refuses any upper-case letter, so the host part of a minted id and the persisted host label are lower case) and then replaces space, tilde, caret, colon, question mark, asterisk, open bracket, backslash, '+' and '@' with '-' (the git-ref-illegal characters, plus the two an agent id may not hold). It falls back to "local" on an empty hostname. It does NOT map dots or underscores.

So a macOS host named `MinDev.local` gives `agent/mindev.local-8ef0cd32`, dot intact, case folded at MINTING. Anything deriving a key, directory name or URL segment from the branch and assuming sanitizeHostname produced only [a-z0-9-] will still carry that dot (and any underscore) straight through.

AN ID MINTED BEFORE #514 WITH AN UPPER-CASE HOSTNAME IS NOT REWRITTEN: control.db keeps it, boot validates it (pki.ValidAgentID) and refuses it with repos.ErrStoredAgentIDInvalid, whose message says a fresh start (delete control.db) is required. The persisted branch of such an instance is therefore also not renamed by this change; the instance re-mints a lower-case id after the fresh start.

REMOVED: app.AgentSlug (PR #247 added it as the one place a branch was normalised to `[a-z0-9-]`) was deleted in PR #513: it turned a branch NAME into a category key by stripping agent/, which D18 (no identity inferred from a name) forbids, and it had no production caller. Do not reintroduce a branch-to-key helper; take an id from the persisted instance_identity instead.

What this does NOT mean: do not "fix" sanitizeHostname to strip dots. The branch name is already recorded in agent_branch_owner and in every existing branch, and changing it renames live agent branches. The sanitized host is also persisted on its own (instance_identity_host), so nothing needs to be parsed back out of the branch. Do not add case folding anywhere else either: the judge, the reader and the audit compare ids exactly (kb/invariants/pki/agent-id-binding/efad10db.md).

Shipped in PR #247, merged to dev as f8142735; the AgentSlug half was removed in PR #513 (merge c9c599de); the lowercasing came with PR #514.
