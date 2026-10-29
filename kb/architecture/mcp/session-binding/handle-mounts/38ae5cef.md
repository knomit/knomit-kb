---
type: reference
domain: [mcp, store, control-db, sessions, migrations]
confidence: 0.95
sources: 1
entities: [handle_mounts, binding_handles, MintBindingHandleOnMount, Store.HandleMount, Store.Purge, resolveHandle, 000018_handle_mounts]
motifs: [derived-state-liability]
refs: ['https://github.com/knomit/knomit/pull/664', 'kb://3ec012f5b4d2/kb/decisions/mcp/experiments/handle-keyed-sibling-table/d0f37357.md', 'kb://3ec012f5b4d2/kb/architecture/mcp/client-sessions/binding-set/29c0fb3a.md', 'kb://3ec012f5b4d2/kb/invariants/mcp/session-binding/scoped-handle-pin-is-mount/ada49bc0.md', 'src://7b4887ce51d9/internal/engine/store/migrate/control/000018_handle_mounts.up.sql@035390e6af66a9a68a54d7592e89915cf5368bb0:c58f4b9a6e7af85e36b3e2a410087bf2ee31ce53', 'src://7b4887ce51d9/internal/engine/store/migrate/control/000018_handle_mounts.down.sql@035390e6af66a9a68a54d7592e89915cf5368bb0:eba8f0d325c66b896cc2f3038b9e8a9d42ffc061', 'src://7b4887ce51d9/internal/foundation/clientsessions/bindings.go@cb6b27fc32cf6f9078f5877db6e432be812d8962:c43cec784c8da26e152a6a4e88c1fa3c80718374', 'src://7b4887ce51d9/internal/foundation/clientsessions/bindings_test.go@cb6b27fc32cf6f9078f5877db6e432be812d8962:caeb3dbb7c0193b9348aa590ee64734e759951b7', 'src://7b4887ce51d9/internal/edge/mcp/binding_gate.go@cb6b27fc32cf6f9078f5877db6e432be812d8962:3a1d913fab88e3d98863675dcb81c8b50a5dbe7f']
---
# handle_mounts (control migration 000018) ties a binding handle to the URL-scoped mount that minted it: no row means minted on the unscoped endpoint, so old handles work on /api/v1/mcp and are refused on scoped mounts

SCHEMA: handle_mounts(handle PRIMARY KEY, mount_uid, created_at), migration 000018_handle_mounts. mount_uid is the scoped mount's pin (repo:<uid> or lens:<uid>), the key mount_experiments uses. NO ROW means the handle was minted on the unscoped endpoint, which is every handle that existed before the table: they keep working on /api/v1/mcp and are REFUSED on a scoped mount; a scoped handle is refused on the unscoped endpoint (errHandleFromScopedEndpoint) and on another scoped mount (errHandleFromOtherEndpoint).

ATOMIC. The row is written in the SAME transaction as binding_handles (MintBindingHandleOnMount): a scoped handle that lost its row would read as unscoped and be accepted on /api/v1/mcp. Store.HandleMount returns an error, not "", on a read failure, for the same reason (fail closed). Pinned by TestMintBindingHandleOnMount_IsAtomic.

PURGE. Store.Purge deletes binding_handles first, then handle_mounts rows by anti-join, so no live handle ever loses its row (TestPurge_CollectsHandleMountsWithTheirHandles).

WHY A SIBLING TABLE: the control chain is replayed as one script and admits only CREATE … IF NOT EXISTS, so a column on binding_handles is impossible (the handle_experiments precedent, kb/decisions/mcp/experiments/handle-keyed-sibling-table).

DOWN MIGRATION drops the table; every surviving scoped handle then reads as unscoped and is accepted on /api/v1/mcp, routing to its repo or lens at the default branch plus any handle_experiments row. Accepted and documented in the down file: handles are not authorization and down migrations are manual. It never routes to a different knowledge base than the handle's pin.

THE MOUNT IS THE PIN (fix/resolvehandle-pin-guard, 2026-10-29). MintBindingHandleOnMount(ctx, handle, mountUID, now) takes the mount alone and writes it as binding_handles.binding too, so a scoped handle's pin always equals its mount_uid. resolveHandle therefore checks only this row; the old "Unreachable ... Checked anyway" pin != mount check from PR #664 was deleted. See kb/invariants/mcp/session-binding/scoped-handle-pin-is-mount/ada49bc0.md. Do not reintroduce a two-value mint.
