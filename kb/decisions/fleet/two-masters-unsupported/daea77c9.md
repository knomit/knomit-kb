---
type: observation
domain: [fleet, decisions, consensus]
confidence: 0.95
sources: 1
entities: [isFleetMaster, PublishUpstream, fleetPeers]
motifs: [unsupported-by-ruling]
refs: ['src://7b4887ce51d9/internal/repos/fleet_master.go@9db8c106a98becf61a007d9ff2bc04fa472e65b4:4cf2ba83049f64ef497f399ad075ece5c8c7df49', 'kb://3ec012f5b4d2/kb/invariants/fleet/baked-ontology/master-refresh/d62746d2.md', 'https://github.com/knomit/knomit/pull/524']
---
# DECISION (user ruling): TWO fleet masters at once are NOT supported — it is the user's responsibility; knomit never elects itself master, and no code detects or fences a second master

RULING (user, PR #524): two masters at once are not supported. Which instance is the master is the user's responsibility. knomit never self-elects a master, and builds no detection, no lease and no fencing for a second one.

WHAT 'MASTER' IS (verified at merge commit 9db8c106): the instance for which RepoInstance.isFleetMaster holds: the fleet root key file (<home>/fleet/root.key) is a regular file AND the local root.crt equals artifacts/fleet/root.crt on the fleet repository's main. It is decided once per round / per merger run, so copying the key in or out takes effect on the next round. Copying the root key to a second machine makes a second master.

WHAT THE CODE DOES WITH TWO MASTERS: nothing special. Both would fetch and judge agent/*, merge into their own agent branches and publish main without force. The second to publish gets a non-fast-forward push error, fetches, merges and retries, so they do not clobber each other's main, but they race and can create redundant merge commits. This is a consequence, not a supported mode; nothing here logs or refuses it (the header comment of fleet_master.go says so).

MISREADING: 'the non-force publish makes two masters safe' is false as a promise: the ruling is that it is unsupported, and the behaviour above is incidental.
