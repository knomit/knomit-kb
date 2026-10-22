---
type: observation
domain: [store-git, kv]
confidence: 0.95
sources: 0
entities: [kv table, kvGet, kvSet, ConfigStorer, IndexStorer, ShallowStorer, ModuleStorer, 'index:modtime']
refs: ['src://7b4887ce51d9/internal/engine/store/git/kv.go@0610c20083fcef9e84f42ab0311a14708bcb6193:7ae56d4f9e656c93120c230f68dd7f210c028192']
---
# kv table backs 4 go-git storer interfaces: Config, Index, Shallow, Module

The kv table is the catchall key-value store backing FOUR go-git storer interfaces: ConfigStorer ('config' key), IndexStorer ('index' key + 'index:modtime' for the round-trip-lost ModTime), ShallowStorer ('shallow' key, newline-separated hash list), ModuleStorer ('module:<name>' keys recording submodule presence). kvGet returns (nil, nil) for missing keys (not an error) so each interface's default (empty Config, empty Index{Version: 2}, nil Shallow) is constructed in the caller. SetIndex always sets idx.ModTime = time.Now() and persists the nanoseconds in a SEPARATE kv entry because the encode/decode round-trip drops the field — without the separate storage, every Index() call would return zero ModTime. Anti-pattern: a key conflict (e.g. 'module:config') — ALL kv keys must be namespaced (literal 'config'/'index'/'shallow' or 'module:<name>'/'index:modtime' prefix).
