---
kind: pragmatic
type: policy
domain: [repos, store, subscription, lifecycle, invariant]
confidence: 0.95
sources: 1
entities: [Service.SetReadOnly, repoHandler.readOnly, Manager.SwapStore, rewireStore, repoBuilder.openStore, Service.SetOntologyRoot, internal/repos/swapstore.go, internal/store/service.go]
motifs: [derived-state-liability, reconstruction-drops-configuration]
refs: ['src://7b4887ce51d9/internal/store/service.go@a1be0404ff37b9c342a90cb9241e17eab053d9f0:64f7e899c1f7d2832debbc93026d25f8b45693ad', 'src://7b4887ce51d9/internal/repos/swapstore.go@a1be0404ff37b9c342a90cb9241e17eab053d9f0:12a2e4758ffe3a92864190375541f1e09d312952', 'src://7b4887ce51d9/internal/repos/builder.go@a1be0404ff37b9c342a90cb9241e17eab053d9f0:b8ede3f32b31e8c834332134889968227ef34fa3', 'https://github.com/knomit/knomit/pull/186']
---
# The store's read-only flag does NOT survive store.Open — SwapStore must re-apply it, exactly like SetOntologyRoot, or a subscription silently becomes writable

`Service.SetReadOnly(bool)` sets `repoHandler.readOnly`, the structural half of a subscription's read-only guarantee. It is PROCESS state, not persisted state: `store.Open` rebuilds the Service from disk and knows nothing about it.

Therefore every path that constructs or reconstructs a Service for a repo must re-apply it from `ri.subscribed`. Today that is two places: `repoBuilder.openStore` (before anything can write) and `rewireStore` on the SwapStore path.

**Consequence for a consumer:** a swap that omits the call leaves a subscription WRITABLE, with no error and no log line — the same failure shape as forgetting `SetOntologyRoot`, which is why the two sit together. If you add a third construction path, it inherits nothing; wire it explicitly.

The flag is set at build/swap time before the store is published and never mutated afterwards, which is why the bool carries no lock. That is a property of the current call sites, not of the field — mutating it on a live store would be a data race.
