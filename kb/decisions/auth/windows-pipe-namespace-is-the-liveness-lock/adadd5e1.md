---
type: observation
domain: [auth, windows, desktop, bridge, security]
confidence: 0.9
sources: 1
entities: [auth.ListenLocal, auth.ErrSocketInUse, isPipeNameTaken, winio.ListenPipe, NtCreateNamedPipeFile, FILE_CREATE, ERROR_ACCESS_DENIED, ERROR_PIPE_BUSY, TestListenLocal_LivePipeIsNotStolen, TestListenLocal_ForeignOwnerIsNotStolenAndMapsToInUse]
motifs: [os-supplies-the-lock, probe-cannot-distinguish-cases]
refs: ['src://7b4887ce51d9/internal/foundation/auth/listen.go@0610c20083fcef9e84f42ab0311a14708bcb6193:992995eaebeaa6241bf0600555ad2ef14da32b04', 'src://7b4887ce51d9/internal/foundation/auth/listen_windows.go@0610c20083fcef9e84f42ab0311a14708bcb6193:3e7190724f39eeaa0606ff53953e4d9d42e174bf', 'src://7b4887ce51d9/internal/foundation/auth/listen_windows_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:07c40011270b69122af00f1fd6a4721f9c600bb8', 'kb://3ec012f5b4d2/kb/decisions/auth/socket-liveness-flock/b27c556d.md', 'kb://3ec012f5b4d2/kb/decisions/auth/windows-local-listener-is-a-named-pipe/5b6e19d0.md', 'kb://3ec012f5b4d2/kb/invariants/auth/require-needs-a-bound-listener/0f2335f4.md', 'https://github.com/knomit/knomit/pull/251']
---
# A named pipe needs no lock file to decide liveness — the NAMESPACE is the lock: FILE_CREATE refuses a second instance of a live name, and the refusal is ERROR_ACCESS_DENIED from ANY foreign owner, which auth maps to ErrSocketInUse

On unix, auth.ListenLocal decides "is another instance alive here?" with an exclusive flock on <path>.lock, because a unix socket FILE outlives its owner and a dial probe cannot tell a leftover from a live server whose accept backlog is full.

On Windows that machinery is unnecessary and DELIBERATELY ABSENT. A pipe name exists only while an instance of it is open — the kernel reaps it with the process — so nothing can be left behind for a successor to clear up. And winio.ListenPipe creates the FIRST instance with the FILE_CREATE disposition on NtCreateNamedPipeFile (pipe.go:378-381 in v0.6.2), which fails outright if the name already exists; that is the effect the Win32 API spells FILE_FLAG_FIRST_PIPE_INSTANCE, but winio does NOT use that flag, so do not go looking for it.

MEASURED: the refusal is ERROR_ACCESS_DENIED. isPipeNameTaken maps it (and ERROR_PIPE_BUSY) onto auth.ErrSocketInUse, which is the class cmd/serve.go and tools/desktop/boot.go match with errors.Is to log at Warn and serve TCP only. Getting the class wrong turns "the desktop already owns this" into a dead server.

THAT BENIGN BRANCH IS BENIGN ONLY WHILE `[auth].require` IS OFF. With it on there is no anonymous path, so serving TCP only would mean answering every request 403 while looking healthy — auth.RequireLocalListener, called straight after listening, fails the boot instead. Do not compress this decision to "name taken -> ErrSocketInUse -> carry on": that is true for half the configurations. See the require-needs-a-bound-listener invariant.

CONSEQUENCE WORTH KNOWING: ERROR_ACCESS_DENIED is what the OS returns for ANY foreign owner, not just another knomit. TestListenLocal_ForeignOwnerIsNotStolenAndMapsToInUse holds a name with a SYSTEM-only ACL and proves the mapping; TestListenLocal_LivePipeIsNotStolen logs the errno verbatim, so a Windows build that changed it fails loudly instead of turning every desktop-plus-serve boot fatal. Both are in internal/foundation/auth/listen_windows_test.go.
