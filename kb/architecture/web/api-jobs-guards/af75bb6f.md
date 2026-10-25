---
type: reference
domain: [web, archtest, architecture]
confidence: 0.9
sources: 1
entities: [webKnownTypes, webAPIExportAllowlist, TestWebAPI_ExportsOnlyMount, layers_test.go, web_no_interfaces_test.go, api/jobs, jobEnvelope]
refs: ['src://7b4887ce51d9/test/archtest/web_split_test.go@97ae7c195e0f4ac76868f9059728e4dd2b950957:9f5221b3f5c0ff1fb6b74cdd7f3ac06d80bceefb', 'src://7b4887ce51d9/test/archtest/layers_test.go@97ae7c195e0f4ac76868f9059728e4dd2b950957:11d2a6b363c20d840806b36c4657f2d227ba1604', 'src://7b4887ce51d9/test/archtest/web_no_interfaces_test.go@97ae7c195e0f4ac76868f9059728e4dd2b950957:3687a3b8ec67557f98fcd8e23d846e71c88a1f1f', 'kb://3ec012f5b4d2/kb/invariants/testing/web-api-exports/ebaa68fd.md', 'kb://3ec012f5b4d2/kb/architecture/web/api-kb-guards/eb689679.md', 'https://github.com/knomit/knomit/pull/627', 'https://github.com/knomit/knomit/pull/630']
---
# api/jobs is registered in the web-split guards: webKnownTypes gets {jobEnvelope: handlers.go}, the layer map allows api/jobs -> hal and httpx only, and its webAPIExportAllowlist key (already present, empty) is now live; api/manage (12e) and api/kb (12f) are live too and nothing is pending

Verified at 97ae7c19 (PR #627). Guard changes that came with api/jobs, all under test/archtest: (1) web_no_interfaces_test.go: webKnownTypes gains the entry for internal/edge/web/api/jobs listing exactly one declared type, jobEnvelope, in handlers.go (mount.go declares none). (2) layers_test.go: the allowed edges for api/jobs are edge/web/hal and edge/web/httpx; any other import into another web-family package fails. (3) web_split_test.go (unchanged file; the same blob as at the 12d merge): the webAPIExportAllowlist key "api/jobs" was already present with an empty list, and is now scanned for real because go list returns the package. Its single export is Mount, with no results.

PENDING (historical, at 97ae7c19): TestWebAPI_ExportsOnlyMount's pending Logf then listed api/kb and api/manage. Since 12f (33eb0307) there is no pending state: all four keys are live and a key whose package go list does not return is a t.Errorf.

CONSEQUENCE: adding an exported name, a second type, or an import of another web-family package to api/jobs fails an archtest rather than a review.

DOES NOT MEAN (at 97ae7c19, superseded): that api/kb or api/manage were policed then; both are policed now.

UPDATE (12e, PR #628, merge a8bf8d7a; re-read there): api/manage was registered in the same guards (webKnownTypes entries in web_no_interfaces_test.go for the types it declares, layers_test.go edges edge/web/api/manage -> hal and -> httpx), and its allowlist key is live and empty. Only api/kb was still pending then.

UPDATE (12f, PR #630, merge 33eb0307; re-read there): api/kb landed and is registered in the same guards (kb/architecture/web/api-kb-guards/eb689679.md). All four api packages are live; nothing is pending.
