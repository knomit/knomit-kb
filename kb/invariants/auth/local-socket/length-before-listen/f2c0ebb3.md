---
kind: pragmatic
type: policy
domain: [auth, unix, config]
confidence: 0.9
sources: 1
entities: [auth.ListenLocal, listenLocal, auth.ErrPathTooLong, auth.SunPathCap, syscall.RawSockaddrUnix, EINVAL, TestListenLocal_PathLengthBoundaryIsTheSunPathCap]
motifs: [validate-before-side-effects, named-error-over-errno]
refs: ['src://7b4887ce51d9/internal/auth/listen_unix.go@c5af471f43db503a3d090d3240d7730afcff441e:c6de0f69ee7f3e4dd0e0e5d4e8f6e0bb9b2b4cf1', 'src://7b4887ce51d9/internal/auth/socketdir_unix.go@c5af471f43db503a3d090d3240d7730afcff441e:ce0802b1548a8471b7cf89cf67a39fac707a4d31', 'src://7b4887ce51d9/internal/auth/listen.go@0a275b92e72af75393601df1b7a5ef7ad61bf518:06020d128d9cc4dfcc7d08c62b2a414795fe65eb', 'src://7b4887ce51d9/internal/auth/listen_unix_test.go@c5af471f43db503a3d090d3240d7730afcff441e:5e0d36b789e18a590800eafcbddc1f910da44b83', 'https://github.com/knomit/knomit/issues/253', 'https://github.com/knomit/knomit/pull/268', 'kb://3ec012f5b4d2/kb/invariants/auth/local-listener-single-site/9cf869d3.md']
---
# ListenLocal refuses a path of SunPathCap bytes or more with ErrPathTooLong as its FIRST step — before the .lock, the fallback dir or net.Listen exist — comparing against the struct's size, never parsing EINVAL

THE RULE. The FIRST statement of the unix `listenLocal` is `if len(path) >= SunPathCap() { return ErrPathTooLong }`. It runs before `prepareSocketDir`, before the `<path>.lock` is opened, and before `net.Listen`. The error names the path, its length and this platform's cap.

WHY FIRST. After the fact, `net.Listen` returns a raw EINVAL that no caller can tell apart from any other bind failure (so `serve` was fatal on it, knomit#253). By then a `.lock` file (and, in the fallback case, the directory) has also been created for an unusable path.

THE BOUNDARY IS PINNED on every platform: `TestListenLocal_PathLengthBoundaryIsTheSunPathCap` builds paths of exactly `SunPathCap()-1` bytes (listens AND dials) and `SunPathCap()` bytes (`ErrPathTooLong`, NOT EINVAL, no socket or lock file). Both lengths are computed from the struct, so the test means the same thing on darwin and linux. A hardcoded 108 fails it on darwin; a hardcoded 104 fails it on linux only, which a darwin-only run cannot see.

Caller handling (WARN, TCP only, `RequireLocalListener` still refuses under [auth].require) is in the single-site invariant's caller contract.

WHAT THIS DOES NOT MEAN: not that a long DATA ROOT reaches this branch. `config.localListenerName` gives it a short fallback first; only an explicit overlong `[socket]`/`KNOMIT_SOCKET` lands here. And not a Windows concern: the error is declared on every platform so callers compile, and only unix returns it.
