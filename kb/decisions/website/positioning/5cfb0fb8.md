---
type: insight
domain: [website, positioning, competitors, git]
confidence: 0.9
sources: 1
entities: [Letta, Context Repositories, MemFS, REST, HTTP, git, commit, branch, merge]
refs: ['src://knomit-io/src/content/blog/git-is-the-substrate-for-agent-knowledge.mdx', kb/principles/philosophy/temporal-graph/5908f56a.md, kb/principles/philosophy/git-is-the-only-source-of-truth/8f74c514.md]
---
# Core differentiator vs git-journal competitors: knomit uses git operations for what they MEAN, like REST uses HTTP

Designer-stated positioning line (during blog article 01 drafting, 2026-07-02): what separates knomit from competitors that also store agent memory as markdown-in-git (Letta Context Repositories / MemFS, released Feb 2026) is NOT the substrate — it is that knomit uses git's operations for their intended semantics, the way REST builds on HTTP's verbs rather than tunneling RPC through POST.

The mapping: commit = assertion of belief (signed), branch = agent identity, merge = consensus, conflict = disagreement between peers surfaced for adjudication, checkout = time-travel, log = timeline of belief, blame = provenance. Because the semantics are preserved, the surrounding git ecosystem works on the corpus unmodified: a PR against kb/ is belief review, CI on the repo is fact validation.

Competitors auto-commit memory files: git as journal/transport. History accumulates but the operations carry no meaning — a branch is not anything, a conflict is a nuisance avoided by worktree isolation, commits are unsigned because they record byte changes rather than assert beliefs.

Use this framing in website/blog/compare content. Argue category, not feature parity: 'git as journal' vs 'git operations as knowledge semantics'. Do not name Letta in foundation essays; reserve head-to-head for a /compare page. First used in the blog post 'The best substrate for agent knowledge shipped in 2005' (section 'Putting files in git is not the same as using git').
