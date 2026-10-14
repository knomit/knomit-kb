---
kind: pragmatic
type: policy
domain: []
confidence: 0.9
sources: 1
entities: [repos.ErrStoredAgentIDInvalid, 'knomit://instance URI SAN', control.db]
motifs: [no-unshipped-migration]
refs: ['src://7b4887ce51d9/internal/repos/fleet_identity.go@3e2e83b082a450210e5de0111fa55774ccb891e6:64d6b86d6c8fedd380df05f3c5b59b6c963d58b9', 'src://7b4887ce51d9/internal/pki/verify.go@3e2e83b082a450210e5de0111fa55774ccb891e6:a524c42b53f2cf4794f89679c80ac8bb03fffc28', 'kb://3ec012f5b4d2/kb/decisions/identity/opaque-persisted-agent-id/cd3a110e.md', 'https://github.com/knomit/knomit/pull/514']
---
# PR #514 builds NO legacy support: no fleet repositories predate the fleet preset change, certificates issued before it are not migrated (they now read as operator certificates), and a stored agent id that is not lower case stops boot with a fresh-start error instead of being rewritten

User rulings (O2, round 3 ruling 3): the fleet feature is new, so no fleet repositories exist to upgrade, and shipping compatibility code for state nobody has is a liability in a security gate. OPTIONS: (a) accept old certificates (knomit://instance/<label>) as instances, lowercase stored ids, migrate stored fleet ontologies: REJECTED; (b) refuse loudly and name the remedy: CHOSEN.

WHAT HAPPENS: a certificate with only the old URI SAN reads as an OPERATOR certificate (no implicit read or push:own): re-issue it with `knomit identity enroll --agent-id`. A control.db whose agent id fails pki.ValidAgentID stops boot with repos.ErrStoredAgentIDInvalid and says to delete control.db (a fresh start; the instance must register with its fleet again); the id is never rewritten. An existing repository built from an older fleet preset keeps its stored ontology (no refresh).

MISREADING: this is not a statement that migrations are never wanted; it is that this state had no users at the time.
