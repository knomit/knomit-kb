---
type: observation
domain: [testing, windows, synthesize, ci]
confidence: 0.75
sources: 1
entities: [TestFleetREST_FactWritesRefused, handleStartSynthesis, stub LLM, Windows fault guard]
motifs: [recovered-panic-still-fails]
refs: ['src://7b4887ce51d9/internal/web/handlers_jobs.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:afcf6e7b33b6a5b9b4c83f814f71c077722c4326', 'https://github.com/knomit/knomit/pull/520']
---
# A REST test that starts a real synthesis review with the stub LLM dereferences nil; the panic is recovered as a task error but the Windows fault guard still fails the leg ('a test took a real access violation') — stop such a test at the hub's 409 and never run the review

Seen in CI on PR #520 (a dispatched run on the pre-fix head failed build-test on windows-2025 only): the control row of the synthesis-runs test started a real review against the stub LLM; the review dereferenced nil, the panic was recovered and reported as a task error, but on Windows it is a real access violation and the suite's fault guard fails the whole leg ('a test took a real access violation'). Linux and macOS passed the same test, so a green non-Windows run proves nothing here.

Fix used: the control request now stops at the job hub's 409 ('Job already running'), which proves the request passed the fleet gate without any review running (TestFleetREST_FactWritesRefused). CONSEQUENCE: a handler test that needs to show a route is reachable should stop at a cheap deterministic refusal downstream, not start the real pipeline with the stub LLM. Evidence is the PR author's CI report; the nil dereference itself was not re-reproduced here.
