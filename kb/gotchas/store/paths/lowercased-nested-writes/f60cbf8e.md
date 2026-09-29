---
type: observation
domain: [store, paths, skills]
confidence: 0.9
sources: 1
entities: [internal/store/fact_write.go, writeFileExact, WriteFact, WriteRootFile, BatchWriteFacts, fact.NormalizePath, skillFileEntry, SKILL.md]
motifs: [normalization-breaks-external-convention]
refs: ['src://7b4887ce51d9/internal/store/fact_write.go@a72455f980a33e0afbfd0ab7f51424876287d9cc:2e76224e55cfed12bb21583c1ee5dd61268fa29f', 'src://7b4887ce51d9/internal/store/skills.go@a72455f980a33e0afbfd0ab7f51424876287d9cc:602ba9e7509d1420c0566075d2614dc96ddcac23']
---
# Every knomit write door lowercases a nested path (WriteFact, WriteFactIfUnchanged, BatchWriteFacts), so `.knomit/skills/x/SKILL.md` written THROUGH knomit lands as `skill.md`; only git keeps SKILL.md, and the skills reader therefore matches SKILL.md case-insensitively

internal/store/fact_write.go: writeFile, WriteFactIfUnchanged and batchWrite call strings.ToLower on the path before writeFileExact (a fact-path rule, fact.NormalizePath). The only case-preserving exported door is WriteRootFile, and it refuses any path containing "/". So nothing outside the store package can commit an upper-case nested path through knomit — a file a convention requires to be upper-case (Agent Skills' SKILL.md) arrives lower-cased when written by a knomit tool, a test fixture, or the UI, while a `git push` keeps the case.

Consequences:
- store.skillFileEntry (internal/store/skills.go) accepts SKILL.md in any case, preferring the exact spelling, and excludes that entry from the bundled files; S19 in the PR C sabotage run shows an exact-match reader loses a knomit-written skill.
- Tests outside internal/store that need an exact-case nested file cannot write it through the Service API; inside the store package use fi.writeFileExact (skills_test.go putRaw).
- Bundled skill files written through knomit are lower-cased too (their URIs follow).

What this does NOT mean: the store is not case-insensitive — git trees are case-sensitive and a SKILL.md and a skill.md can coexist in one folder (the exact spelling wins; the other is a bundled file).
