---
kind: pragmatic
type: policy
domain: [build, desktop, packaging, ci]
confidence: 0.9
sources: 1
entities: [Makefile, make dist, make server, make build, make desktop, release-server, E2E_BUILD, e2e/helpers/binary.ts, desktop-deps]
motifs: [target-name-drift]
refs: ['src://7b4887ce51d9/Makefile@7cae543c22b005cb429753ae31ae1593ba69b298:2f4aff6ffb415d4bcad4acad49874680b24f7da8#L281-L281', 'src://7b4887ce51d9/Makefile@7cae543c22b005cb429753ae31ae1593ba69b298:2f4aff6ffb415d4bcad4acad49874680b24f7da8#L343-L343', 'src://7b4887ce51d9/Makefile@7cae543c22b005cb429753ae31ae1593ba69b298:2f4aff6ffb415d4bcad4acad49874680b24f7da8#L350-L350', 'src://7b4887ce51d9/Makefile@7cae543c22b005cb429753ae31ae1593ba69b298:2f4aff6ffb415d4bcad4acad49874680b24f7da8#L469-L469', 'src://7b4887ce51d9/Makefile@7cae543c22b005cb429753ae31ae1593ba69b298:2f4aff6ffb415d4bcad4acad49874680b24f7da8#L760-L760', 'src://7b4887ce51d9/e2e/helpers/binary.ts@7cae543c22b005cb429753ae31ae1593ba69b298:28175f48e207cf958c3009c386effdb0dfd063f0', 'kb://3ec012f5b4d2/kb/conventions/build/desktop/2ae77359.md', 'kb://3ec012f5b4d2/kb/gotchas/testing/e2e/7d273c30.md', 'https://github.com/knomit/knomit/pull/508']
---
# `make dist` is the end-to-end build (web UI, server, desktop app) on every platform and needs the desktop toolchain; server-only callers (CI release-server, the e2e harness, Docker) use `make server`, the app alone is `make desktop`

Layout requested by the user on 2026-10-13 and landed in PR #508 (merge 7cae543c): `server: web tokenizers-lib download-ort` builds the web UI, native libs and the `knomit`, `kb` and `knomit-okf` binaries into `$(DIST)`; `build: server` is a compatibility alias; `desktop: server desktop-ui` adds the desktop app on top; `dist: server desktop` is both, end to end, on EVERY platform; `release-server: server`. Before #508 `dist` was only the server build (`download-ort tokenizers-lib build`).

Who uses which: the app alone or a full local distribution is `make desktop` or `make dist`; server-only callers use `make server` (CI release-server, the e2e harness through `E2E_BUILD=1`, anything that previously ran `make build` or the old server-only `make dist`). Docker does not call make. The server binary is compiled once in every target (see kb/conventions/build/desktop/2ae77359.md).

**Consequence:** `make dist` now needs the native desktop toolchain, including GTK4/WebKitGTK on Linux (`make desktop-deps`); a Linux box or CI job that only has the server toolchain fails at `dist`. Do not use `dist` in a job that only needs the server; use `make server`.

**What this does NOT mean:** `dist` is not the server tarball and not a release step (releases go through `release-server` and `release-desktop`); `make build` was not removed (it is `server`), so old scripts keep working, but new text should say `make server`.
