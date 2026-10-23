---
type: observation
domain: [synthesize, discovery, motif, testing]
confidence: 0.9
sources: 1
entities: [TestDerivedWriters_NoLocalMotifValidation, TestDerivedWriters_DropRatherThanDiscard, TestDerivedWriters_NoMechanicalSeedStamping, readDiscoverySources, motifGateCallSites, DropInvalidMotifs, TestPrompts_NeverInterpolateContext]
motifs: [vacuous-guard-key]
refs: ['src://7b4887ce51d9/internal/corpus/synthesize/motif_writers_test.go@b50d817eb4147f128c8e319c4b61d17aaf1308bb:6f5490cbec01fa12a1c64c972e1404d9e917132e', 'src://7b4887ce51d9/internal/corpus/synthesize/context_test.go@b50d817eb4147f128c8e319c4b61d17aaf1308bb:38bfab8b70c09bd02aa4ef9138daf711ea008bd2', 'src://7b4887ce51d9/internal/domain/fact/motif_conformance_test.go@b50d817eb4147f128c8e319c4b61d17aaf1308bb:c9f1b59ca7118326e4ba95700c49d3c55e198884', 'kb://3ec012f5b4d2/kb/decisions/synthesize/motif/derived-writer-rules/aba23e53.md', 'kb://3ec012f5b4d2/kb/gotchas/testing/motif-guards/4fee58d5.md', 'kb://3ec012f5b4d2/kb/gotchas/testing/archtest-prompt-context-scan/2035ad06.md', 'https://github.com/knomit/knomit/pull/608']
---
# The MN11 derived-writer guards (TestDerivedWriters_NoLocalMotifValidation, _DropRatherThanDiscard, _NoMechanicalSeedStamping) scan every non-test file of the discovery directory, but the MN4 DropInvalidMotifs call-site key still names the single file discovery/discovery.go

At b50d817e (PR #608) the three derived-writer guards in internal/corpus/synthesize/motif_writers_test.go run over derivedWriterSources: decision.go plus the concatenated text of EVERY non-test .go file under internal/corpus/synthesize/discovery/ (readDiscoverySources globs the directory, so a file added to the child is scanned; the walk asserts at least one file, so it cannot go vacuous). TestPrompts_NeverInterpolateContext (context_test.go) likewise globs every non-test discovery/*.go for a .Context selector in prompt text, because renderDiscoverPrompt lives there.

Two things stay keyed to a NAMED file: internal/domain/fact/motif_conformance_test.go motifGateCallSites lists "internal/corpus/synthesize/discovery/discovery.go" as the one discovery file allowed to call fact.DropInvalidMotifs. Consequence: a new discovery file may be scanned by the guards for free, but moving the DropInvalidMotifs call (or any motif-gate helper call) into another discovery file fails the MN4 test until that key is changed, and the failure is loud, not silent.

DOES NOT MEAN: that the MN4 allowlist is directory-wide. Only the MN11 writer guards and the prompt-context scan are glob-based.
