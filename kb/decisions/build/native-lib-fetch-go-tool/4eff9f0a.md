---
type: observation
domain: [build, tooling, cross-platform]
confidence: 0.95
sources: 0
entities: [tools/fetchlibs, ortSpec, tokenizersSpec, ortVersion, tokenizersVersion, Makefile, tools/calibrate]
refs: ['src://knomit/tools/fetchlibs/spec.go@d960e325', 'src://knomit/tools/fetchlibs/main.go@d960e325', 'src://knomit/Makefile@d960e325', 'https://github.com/knomit/knomit/pull/28']
---
# Native-lib downloads unified into one cross-platform Go tool (tools/fetchlibs), not three bash/Makefile fetchers

**Options considered** (resolved via AskUserQuestion): (1) Unify all three native-lib fetchers — ONNX Runtime, graphqlite, libtokenizers.a — into a single pure-Go stdlib tool `tools/fetchlibs`, invoked via `go run`. (2) Port only `scripts/fetch_tokenizers_lib.sh` to Go and leave the ort/graphqlite bash blocks in the Makefile.

**Rationale**: The driver was true cross-platform support (Windows). The blocker was never just the one script — the Makefile's `download-ort`/`download-graphqlite` targets and the script were ALL bash/`uname`/`curl`/`tar` based, and `make` itself isn't on Windows. Option 2 leaves the build non-cross-platform, so it doesn't meet the goal. Option 1 removes the entire shell/uname/make dependency from fetching and follows the existing `tools/calibrate` precedent (Go tool under `tools/`).

**The choice**: Created `tools/fetchlibs` (pure Go, stdlib only: net/http, archive/tar, compress/gzip, archive/zip, os/exec for darwin codesign). It is the single source of truth for native-lib versions and per-platform asset names. Handles per-OS archive differences (ONNX is .tgz on mac/linux but .zip on Windows). Idempotent: skips if dest file exists. Makefile `setup`/`download-ort`/`tokenizers-lib` delegate to `go run ./tools/fetchlibs [-only <id>] dist/lib`; the Makefile keeps only a minimal `ORT_LIB_NAME` switch for the `run` target's ORT_LIB_PATH. `scripts/` deleted (so `scripts/fetch_tokenizers_lib.sh` no longer exists — that is the point of this decision, not drift). Pure logic (per-platform spec resolution + archive extraction) is unit-tested via TDD.

**STATUS AT HEAD — TWO libs, not three.** `tools/fetchlibs/spec.go` declares only `ortVersion` (1.24.3) and `tokenizersVersion` (v1.27.0); the `id` field is `"ort" | "tokenizers"`.

WHAT THIS DOES NOT MEAN:
- **There is no graphqlite fetch any more.** `graphqliteSpec`, `graphqliteVersion`, and the Makefile `download-graphqlite` target are all GONE, removed with the GraphQLite dependency itself (PR #28). Do not expect `-only graphqlite` to work or look for a third libSpec. The "all three" framing above is the historical shape of the decision, not the current inventory.
- The unification decision itself still stands — dropping one library did not reintroduce shell fetchers.
