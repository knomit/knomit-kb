---
kind: pragmatic
type: policy
domain: [verify, store, cmd, fleet, forensics]
confidence: 0.9
sources: 1
entities: [store.Audit, AuditInput, AuditReport, knomit verify audit, cmd.ExitCodeError, exitCodeOf, AuditNeverMember, AuditOtherAgent, AuditRevoked]
refs: ['src://7b4887ce51d9/internal/engine/store/verify/audit.go@0610c20083fcef9e84f42ab0311a14708bcb6193:34ca448d17fe957968a1427d40ebd4a79b91f7bd', 'src://7b4887ce51d9/cmd/verify_audit.go@0610c20083fcef9e84f42ab0311a14708bcb6193:e1d42afafc4b0ff34f473cf594de1c1dc8fb1fc7']
---
# knomit verify audit re-checks a WHOLE branch against EVERY version of the fleet's member records (the fleet's git history), so rotated keys and departed agents are attributed, not flagged; it is on demand only and never in a write path

verify.Audit(AuditInput{KBDir, Branch, FleetDir, FleetRev, FleetRoot, Key}) walks the fleet repository's history from FleetRev collecting every (agent, state) each key had across record versions (history = the fact's versions, user ruling), then walks the whole KB branch. Findings: unsigned (an unsigned merge passing M3 excepted), never-member (key never on any record), other-agent (key belonged to a different agent than the author id), revoked (key held by the author's agent in a version whose state was revoked). --key <fingerprint or prefix> lists every commit that key signed: the breach tool from the transitive-trust ruling (look at everything a key signed from the day of detection backwards).

knomit verify audit is a subcommand of the integrity verify command (which is unchanged); it takes plain checkouts (--dir, --fleet, --branch, --fleet-rev, --key, --json) and returns cmd.ExitCodeError: 0 clean, 1 findings, 2 could not run (main.go exitCodeOf honours it; commands never os.Exit).

The contrast with the gate is deliberate: the gate admits only the CURRENT key of an ACTIVE agent (acceptance is now), the audit asks whether history is attributable.
