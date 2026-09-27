---
kind: pragmatic
type: policy
domain: [verify, store, cmd, fleet, forensics]
confidence: 0.9
sources: 1
entities: [store.Audit, AuditInput, AuditReport, knomit verify audit, cmd.ExitCodeError, exitCodeOf, AuditNeverMember, AuditOtherAgent, AuditRevoked]
refs: ['src://7b4887ce51d9/internal/store/verify_audit.go@d4205ad457f087b8a714c17f4745a5f7bea3675c:e7e8d9aa36959936705761f81695f0ad3ace9561', 'src://7b4887ce51d9/cmd/verify_audit.go@d4205ad457f087b8a714c17f4745a5f7bea3675c:43ca7e9fb93a5fa1ce241a60e1ab1e938673cdc4']
---
# knomit verify audit re-checks a WHOLE branch against EVERY version of the fleet's member records (the fleet's git history), so rotated keys and departed agents are attributed, not flagged; it is on demand only and never in a write path

store.Audit(AuditInput{KBDir, Branch, FleetDir, FleetRev, FleetRoot, Key}) walks the fleet repository's history from FleetRev collecting every (agent, state) each key had across record versions (history = the fact's versions, user ruling), then walks the whole KB branch. Findings: unsigned (an unsigned merge passing M3 excepted), never-member (key never on any record), other-agent (key belonged to a different agent than the author id), revoked (key held by the author's agent in a version whose state was revoked). --key <fingerprint or prefix> lists every commit that key signed: the breach tool from the transitive-trust ruling (look at everything a key signed from the day of detection backwards).

knomit verify audit is a subcommand of the integrity verify command (which is unchanged); it takes plain checkouts (--dir, --fleet, --branch, --fleet-rev, --key, --json) and returns cmd.ExitCodeError: 0 clean, 1 findings, 2 could not run (main.go exitCodeOf honours it; commands never os.Exit).

The contrast with the gate is deliberate: the gate admits only the CURRENT key of an ACTIVE agent (acceptance is now), the audit asks whether history is attributable.
