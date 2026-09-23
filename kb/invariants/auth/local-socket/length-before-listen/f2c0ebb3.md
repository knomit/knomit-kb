---
kind: pragmatic
type: policy
domain: [auth, unix, config]
confidence: 0.9
sources: 1
entities: [auth.ListenLocal, listenLocal, auth.ErrPathTooLong, auth.SunPathCap, syscall.RawSockaddrUnix, EINVAL, TestListenLocal_PathLengthBoundaryIsTheSunPathCap]
motifs: [validate-before-side-effects, named-error-over-errno]
refs: ['src://7b4887ce51d9/internal/auth/listen_unix.go@9d83dd7ad695909caf0c84b6e1bf4fd8ac983e21:ab322614f7b424c0e278bc15ab771e4501750686', 'src://7b4887ce51d9/internal/auth/socketdir_unix.go@9d83dd7ad695909caf0c84b6e1bf4fd8ac983e21:a4e14252e88fc09a4d221e0b78569b4d0e2c2868', 'src://7b4887ce51d9/internal/auth/listen.go@9d83dd7ad695909caf0c84b6e1bf4fd8ac983e21:06020d128d9cc4dfcc7d08c62b2a414795fe65eb', 'src://7b4887ce51d9/internal/auth/listen_unix_test.go@9d83dd7ad695909caf0c84b6e1bf4fd8ac983e21:5e0d36b789e18a590800eafcbddc1f910da44b83', 'https://github.com/knomit/knomit/issues/253', 'https://github.com/knomit/knomit/pull/268', 'kb://3ec012f5b4d2/kb/invariants/auth/local-listener-single-site/9cf869d3.md']
---
# ListenLocal refuses a path of SunPathCap bytes or more with ErrPathTooLong as its FIRST step — before the .lock, the fallback dir or net.Listen exist — comparing against the struct's size, never parsing EINVAL

THE RULE. The FIRST statement of the unix `listenLocal` is `if len(path) >= SunPathCap() { return ErrPathTooLong }`. It runs before `prepareSocketDir`, before the `<path>.lock` is opened, and before `net.Listen`. The error names the path, its length and this platform's cap.

WHY FIRST. After the fact, `net.Listen` returns a raw EINVAL that no caller can tell apart from any other bind failure (so `serve` was fatal on it, knomit#253). By then a `.lock` file (and, in the fallback case, the directory) has also been created for an unusable path.

THE BOUNDARY IS PINNED on every platform: `TestListenLocal_PathLengthBoundaryIsTheSunPathCap` builds paths of exactly `SunPathCap()-1` bytes (listens AND dials) and `SunPathCap()` bytes (`ErrPathTooLong`, NOT EINVAL, no socket or lock file). What makes the test meaningful is NOT that both lengths come from the struct. That is self-referencing: any cap no larger than the real limit is self-consistent, and a hardcoded 104 passed on linux that way (review F1, #268). The test starts with a raw `net.Listen`, bypassing the pre-check, that must ACCEPT `SunPathCap()-1` bytes and REFUSE `SunPathCap()` bytes. `net.Listen` is the independent oracle and the layer knomit listens through. A hardcoded 108 fails its accept half on darwin, and a hardcoded 104 fails its refuse half on linux (CI's linux leg). The oracle is Go's limit, not the kernel's: both kernels would bind one more byte.

Caller handling (WARN, TCP only, `RequireLocalListener` still refuses under [auth].require) is in the single-site invariant's caller contract.

WHAT THIS DOES NOT MEAN: not that a long DATA ROOT reaches this branch. `config.localListenerName` gives it a short fallback first; only an explicit overlong `[socket]`/`KNOMIT_SOCKET` lands here. And not a Windows concern: the error is declared on every platform so callers compile, and only unix returns it.
