---
kind: pragmatic
type: policy
domain: [global]
confidence: 1
sources: 0
entities: [designer]
refs: []
---
# Every fact carries cryptographic provenance via its commit

Every fact write — learn, update, retract, subsume, sync — is a commit
authored under an identity-carrying address. Agent commits use
`<agent-id>+<operation>@agents.knomit.io` as author and the bare
`<agent-id>@agents.knomit.io` as committer. Human commits use the
human's email with `+<operation>` subaddressing. Commits are signed,
making the (who, what, when, operation) tuple cryptographically
verifiable.

Why: the KB is consumed by AI agents and merged across machines. The
graph is only trustworthy if every claim is attributable. Email
subaddressing turns the operation type itself into a queryable field
(`git log --author="+learn@"`), and signing prevents a tampered or
forged history from silently entering consensus. Without this, knomit
degrades to "facts a process wrote at some point" — indistinguishable
from a scratchpad.
