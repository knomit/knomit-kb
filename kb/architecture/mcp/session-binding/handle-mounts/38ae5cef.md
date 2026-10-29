---
type: reference
domain: [mcp, store, control-db, sessions, migrations]
confidence: 0.95
sources: 1
entities: [handle_mounts, binding_handles, MintBindingHandleOnMount, Store.HandleMount, Store.Purge, resolveHandle, 000018_handle_mounts]
motifs: [derived-state-liability]
refs: ['https://github.com/knomit/knomit/pull/664', 'https://github.com/knomit/knomit/pull/689', 'kb://3ec012f5b4d2/kb/decisions/mcp/experiments/handle-keyed-sibling-table/d0f37357.md', 'kb://3ec012f5b4d2/kb/architecture/mcp/client-sessions/binding-set/29c0fb3a.md', 'kb://3ec012f5b4d2/kb/invariants/mcp/session-binding/scoped-handle-pin-is-mount/ada49bc0.md', 'src://7b4887ce51d9/internal/engine/store/migrate/control/000018_handle_mounts.up.sql@267a0b1239d290bd5a8beb1ae14699777256aa41:c58f4b9a6e7af85e36b3e2a410087bf2ee31ce53', 'src://7b4887ce51d9/internal/engine/store/migrate/control/000018_handle_mounts.down.sql@267a0b1239d290bd5a8beb1ae14699777256aa41:eba8f0d325c66b896cc2f3038b9e8a9d42ffc061', 'src://7b4887ce51d9/internal/foundation/clientsessions/bindings.go@267a0b1239d290bd5a8beb1ae14699777256aa41:c43cec784c8da26e152a6a4e88c1fa3c80718374', 'src://7b4887ce51d9/internal/foundation/clientsessions/bindings_test.go@267a0b1239d290bd5a8beb1ae14699777256aa41:caeb3dbb7c0193b9348aa590ee64734e759951b7', 'src://7b4887ce51d9/internal/edge/mcp/binding_gate.go@267a0b1239d290bd5a8beb1ae14699777256aa41:3a1d913fab88e3d98863675dcb81c8b50a5dbe7f', 'src://7b4887ce51d9/internal/edge/web/scoped_mount_middleware.go@267a0b1239d290bd5a8beb1ae14699777256aa41:bc4da305e8f66cf3d06690d810cfe717a2f66bda']
---
# handle_mounts (control migration 000018) ties a binding handle to the URL-scoped mount that minted it: no row means minted on the unscoped endpoint, so old handles work on /api/v1/mcp and are refused on scoped mounts

SCHEMA: handle_mounts(handle PRIMARY KEY, mount_uid, created_at), migration 000018_handle_mounts. mount_uid is the scoped mount's pin (repo:<uid> or lens:<uid>), Binding.PinID() as ScopedMountMiddleware records it (mount_experiments, which once used the same key, is no longer read or written since PR #686). NO ROW means the handle was minted on the unscoped endpoint, which is every handle that existed before the table: they keep working on /api/v1/mcp and are REFUSED on a scoped mount; a scoped handle is refused on the unscoped endpoint (errHandleFromScopedEndpoint) and on another scoped mount (errHandleFromOtherEndpoint).

ATOMIC. The row is written in the SAME transaction as binding_handles (MintBindingHandleOnMount): a scoped handle that lost its row would read as unscoped and be accepted on /api/v1/mcp. Store.HandleMount returns an error, not "", on a read failure, for the same reason (fail closed). Pinned by TestMintBindingHandleOnMount_IsAtomic.

PURGE. Store.Purge deletes binding_handles first, then handle_mounts rows by anti-join, so no live handle ever loses its row (TestPurge_CollectsHandleMountsWithTheirHandles).

WHY A SIBLING TABLE: the control chain is replayed as one script and admits only CREATE … IF NOT EXISTS, so a column on binding_handles is impossible (the handle_experiments precedent, kb/decisions/mcp/experiments/handle-keyed-sibling-table).

DOWN MIGRATION drops the table; every surviving scoped handle then reads as unscoped and is accepted on /api/v1/mcp, routing to its repo or lens at the default branch plus any handle_experiments row. Accepted and documented in the down file: handles are not authorization and down migrations are manual. It never routes to a different knowledge base than the handle's pin.

THE MOUNT IS THE PIN (PR #689, merged 2026-10-29 as 267a0b12). MintBindingHandleOnMount(ctx, handle, mountUID, now) takes the mount alone and writes it as binding_handles.binding too, so a scoped handle's pin always equals its mount_uid. resolveHandle therefore checks only this row; the old "Unreachable ... Checked anyway" pin != mount check from PR #664 was deleted. See kb/invariants/mcp/session-binding/scoped-handle-pin-is-mount/ada49bc0.md. Do not reintroduce a two-value mint.
