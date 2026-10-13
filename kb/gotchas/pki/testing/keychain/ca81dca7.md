---
kind: pragmatic
type: policy
domain: [pki, testing, security]
confidence: 0.9
sources: 1
entities: [keyring.MockInit, keyring.MockInitWithError, pki.OSKeychain, pki.Keychain, rootKeychain, useKeychain, stdinIsTerminal, onTerminal, TestMain]
motifs: [global-provider-swap]
refs: ['src://7b4887ce51d9/internal/pki/main_test.go@8e86308d84b35c8225020d0827fffb561908ad12:9eac204609bd4a2c1ed79dd21112a2924cb2b0b2', 'src://7b4887ce51d9/cmd/main_signer_test.go@8e86308d84b35c8225020d0827fffb561908ad12:1cfa5ea0def29163102df392cbfcf06b52d81c6e', 'src://7b4887ce51d9/cmd/identity_custody_test.go@8e86308d84b35c8225020d0827fffb561908ad12:7fc18e23f9ff3f0e2bf27967c6aa6da2bd2b9187', 'src://7b4887ce51d9/internal/pki/custody.go@8e86308d84b35c8225020d0827fffb561908ad12:2f577c0d343400fedd2f063e5e62c08892266d8d', 'https://github.com/knomit/knomit/pull/512']
---
# No test may touch the real OS keychain: internal/pki and cmd TestMain call keyring.MockInit(), pki tests inject a fake pki.Keychain, and cmd tests swap the rootKeychain variable; a new test binary that reaches pki.OSKeychain needs the same TestMain line

`pki.OSKeychain()` calls github.com/zalando/go-keyring, whose provider is a PROCESS-GLOBAL variable. `keyring.MockInit()` swaps it for an in-memory map; nothing else stops a test from writing items into the developer's real macOS Keychain, Windows Credential Manager or Secret Service.

WHAT IS IN PLACE (PR #512, merge 8e86308d). `internal/pki/main_test.go` and `cmd/main_signer_test.go` call `keyring.MockInit()` in TestMain; they are the only `keyring.` uses outside internal/pki/custody.go. pki custody tests pass a `fakeKeychain` (injectable Get/Set/Delete errors). cmd tests replace `rootKeychain` via `useKeychain(t, kc)`, and `stdinIsTerminal` via `onTerminal(t)` because export-root refuses off a terminal.

CONSEQUENCE. Tests must NEVER touch the real keychain. Any OTHER package whose tests run `knomit identity init-master`/`export-root`/`import-root` or `oauth --sign` through cmd, or call `pki.CreateRoot`/`OpenRoot`/`ExportRoot`/`ImportRoot` with `pki.OSKeychain()`, must add `keyring.MockInit()` to its TestMain first. Fixtures built with `pkitest`/`pki.NewRoot` are file-only and never touch a keychain.

SIDE EFFECT. Because cmd tests' default rootKeychain is the go-keyring mock, `init-master` in a cmd test stores into the mock keychain and enroll no longer needs a passphrase; a test that means to exercise the passphrase file must make the keychain refuse (TestIdentity_EnrollNeedsAPassphraseAndTheRightOne does).
