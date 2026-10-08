---
kind: pragmatic
type: policy
domain: [ci, testing, windows, performance]
confidence: 0.9
sources: 1
entities: [.github/scripts/go-test-capture.sh, .github/workflows/tests.yml, actions/upload-artifact, gotest-json, RUNNER_TEMP, build-test, slow-tests, windows-2025]
motifs: [observability-before-optimization]
refs: ['src://7b4887ce51d9/.github/scripts/go-test-capture.sh@19a6a6dae44ea3b76ba11a953e207bdd448318b7:2ef25e73e1d7805c7c3dfc216f1ac4b97e368a8e#L49-L60', 'src://7b4887ce51d9/.github/workflows/tests.yml@19a6a6dae44ea3b76ba11a953e207bdd448318b7:708a2367a1287d40d8cbc03801949170f75733df#L408-L422', 'src://7b4887ce51d9/.github/workflows/tests.yml@19a6a6dae44ea3b76ba11a953e207bdd448318b7:708a2367a1287d40d8cbc03801949170f75733df#L512-L522', 'https://github.com/knomit/knomit/pull/441', 'https://github.com/knomit/knomit/pull/440', 'https://github.com/knomit/knomit/actions/runs/37834083557']
---
# Per-test CI timings: every go-test-capture.sh step prints its 25 slowest top-level tests in a '25 slowest tests (<name>)' log group, and Windows legs of build-test and slow-tests upload their full go test -json streams as artifacts 'gotest-json build-test (windows-2025)' / 'gotest-json slow-tests (<leg>)', kept 14 days

Since #441 (merged as 19a6a6da), this is where to read which tests are slow in CI. Before #441 the logs gave only per-package Elapsed, and Windows per-test timing could not be obtained at all.

CONSOLE: .github/scripts/go-test-capture.sh already keeps the full `go test -json` stream in $RUNNER_TEMP/gotest-<name>.jsonl. After the per-package lines and the skip lines, it now prints a `::group::25 slowest tests (<name>)` block. Each line is `<Elapsed>s<TAB><pass|fail><TAB><package><TAB><test>`, top-level tests only, sorted by Elapsed. Subtests are excluded because their time is already inside the parent's. The block is informational and ends in `|| true`; the step's exit status is still go test's own.

ARTIFACTS: on windows-2025 only, `actions/upload-artifact@v4` runs with `if: matrix.os == 'windows-2025' && !cancelled()`, so a red leg still uploads. It uploads ${{ runner.temp }}/gotest-*.jsonl with retention 14 days:
- build-test uploads `gotest-json build-test (windows-2025)`, holding the stream of every capture step on that leg;
- each slow-tests Windows leg uploads `gotest-json slow-tests (windows-2025 synthesize|store|repos)`.
Fetch one with `gh run download <run-id> -R knomit/knomit -n '<artifact name>'`, then rank tests with jq over the Test/Elapsed fields.

WHEN IT EXISTS: since #440, PRs run Linux tests only. So the Windows artifacts and Windows slowest-tests output exist ONLY on dev pushes, nightly runs and dispatch with full=true, never on a PR run. A PR run has ubuntu slowest-tests output only. The upload step's first real run was the dev push after #441, so if no artifact shows up there, suspect the name or the Windows glob first.

NOT MEANT: the 25-line list is not a complete profile. It is the top of one step. For anything deeper, read the artifact (Windows) or rerun locally with `go test -json`.
