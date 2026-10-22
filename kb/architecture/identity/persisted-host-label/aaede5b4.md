---
type: observation
domain: []
confidence: 0.9
sources: 1
entities: [instance_identity_host, repos.resolveIdentity, repos.Identity, Deps.AgentHost, fact.TriggerIdentity]
refs: ['src://7b4887ce51d9/internal/store/migrate/control/000015_instance_identity_host.up.sql@c9c599dee475771c4226212ac0889817c6b268d4:8067a4cfae415f1711435466098eff57648dba9b', 'src://7b4887ce51d9/internal/repos/fleet_identity.go@c9c599dee475771c4226212ac0889817c6b268d4:24f7f5078ec357a484f072beebe894c6d33a813b', 'src://7b4887ce51d9/internal/repos/identity_noname_test.go@c9c599dee475771c4226212ac0889817c6b268d4:53a4020accd8ff3ca110b1620168aa193a0059af', 'src://7b4887ce51d9/internal/domain/fact/triggers_identity_test.go@48d32e95ccc2bb2b439f975479ae15d165e5049e:3ac0b7bc1ed16389eeea74207e4ebdddb50e6a9a', 'https://github.com/knomit/knomit/pull/513']
---
# The host label is persisted with the identity in control.db instance_identity_host (migration 000015), written in the same transaction as the first instance_identity row; an identity created before it has an empty host and {host} is then invalid on that instance

repos.resolveIdentity(db, fresh Identity, now): if a row exists it returns (agent_id, branch, host via LEFT JOIN instance_identity_host); else it inserts instance_identity and, when fresh.Host is non-empty, instance_identity_host(id=1, host) in ONE transaction. A separate table because the control chain admits only CREATE ... IF NOT EXISTS (TestControl_UpMigrationsAreIdempotentDDL).

CONSEQUENCES: Deps.AgentHost feeds fact.TriggerIdentity.Host. A hostname ending in -<8 hex> (ci-0badcafe) is used verbatim. A pre-existing control.db has no host row: reading never backfills one (no parse of the branch), so a trigger whose match names {host} compiles invalid there ('placeholder {host} has no usable value'); recreating control.db is the fresh start (pre-release, no legacy path).

Tests: TestResolveIdentity_PersistsTheHostLabel, TestTriggerIdentity_ComesFromThePersistedIdentity, TestSubstitutePlaceholders_HostIsVerbatimAndFp8IsGone.
