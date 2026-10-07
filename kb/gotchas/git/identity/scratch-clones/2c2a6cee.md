---
kind: pragmatic
type: policy
domain: [git, process, identity]
confidence: 0.95
sources: 1
entities: [git user.name, git user.email, knomit-playbooks, refs/pull]
refs: []
---
# A commit made in any clone outside the knomit checkout (scratchpad clones of knomit-playbooks or other repos) takes the machine's global git identity unless one is set explicitly: always commit as knomit <k@knomit.io> with -c user.name=knomit -c user.email=k@knomit.io

The knomit checkout sets user.name=knomit locally; a fresh clone elsewhere does not, and git falls back to the global identity. On 2026-10-06 three commits in knomit/knomit-playbooks (two by a worker, one by the coordinating session) were authored with the machine owner's personal account that way. Removing them required rewriting history and deleting and recreating the GitHub repo, because GitHub keeps every pull request's commits under refs/pull/N/head and a force-push does not remove them.

Rule: every brief for work that commits outside the knomit checkout tells the agent to commit with `git -c user.name=knomit -c user.email=k@knomit.io commit ...` (or set them in that clone's local config before the first commit), and to check `git log --format='%an <%ae> %cn <%ce>'` before pushing. Push with the `knomit` GitHub account (gh's active account), never another.

WHAT THIS DOES NOT MEAN: it is not enough that pushes go through the knomit account; the author and committer fields inside each commit are what GitHub shows and what history keeps.
