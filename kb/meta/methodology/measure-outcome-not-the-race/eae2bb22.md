---
type: methodology
domain: [testing, methodology, review, concurrency]
confidence: 0.9
sources: 1
entities: [TestManagerClose_DrainsInFlightCreate, Manager.Close, Manager.StartCreate, drainCreates, ErrManagerStopped]
motifs: [intervention-perturbs-measurement, test-mode-hides-condition]
refs: ['kb://3ec012f5b4d2/kb/meta/methodology/guard-setup-outruns-hazard/2b230458.md', 'kb://3ec012f5b4d2/kb/meta/methodology/fixture-discrimination-precondition/f88dc045.md', 'kb://3ec012f5b4d2/kb/meta/methodology/prove-the-regression-test-fails/d321b280.md', 'src://7b4887ce51d9/internal/repos/create_job_test.go@fbb62045665c69729c6af1150fa0303282f5a749:c6899c42dae67f5210e48740ea2bcf2806ac4014', 'src://7b4887ce51d9/internal/repos/manager.go@fbb62045665c69729c6af1150fa0303282f5a749:f4f754dd420fb0bee6b8b46702b4bf7bebf8d8e0', 'https://github.com/knomit/knomit/pull/167']
---
# When the bug's own effect ALTERS the timing you would measure, a timing assertion cannot be made reliable by widening the margin — change the question to the OUTCOME, and treat 'widening made it worse' as the tell

**The rule.** Some concurrency guards can only be tested by a question about TIMING — "had B finished when A returned?" — and the usual way to make such a test reliable is to widen the margin between A and B. That remedy FAILS, silently, whenever the bug's own effect changes how long B takes. Then the margin is not the variable, no fixture is big enough, and the test stays partly inert however much you feed it. Ask about B's OUTCOME instead: outcomes are not races, so the assertion becomes deterministic and usually needs a SMALLER fixture, not a bigger one.

**THE DIAGNOSTIC, and it is the part worth carrying:** if widening the margin makes the test WORSE, stop tuning the fixture. That is not noise — it is evidence that the thing you are varying is not the thing that decides the result, and it should redirect you from the fixture to the question.

**How it presented (knomit, PR #167).** `Manager.Close` had to drain in-flight detached repo creates before releasing the control-db handles. The obvious test: start a create, call `Close`, assert the job was terminal when `Close` returned. Measured red-runs out of 30 with the drain removed:

- preset create (~12ms) vs Close (~6ms), a 2x margin ...... **27/30**
- clone create, 1 fact (~65ms), ~10x margin ............... **28/30**
- clone create, 200 facts (~275ms), ~45x margin ........... **20/30**

The 200-fact remote made it WORSE, which is what exposed the mechanism: **Close BREAKING the create is what makes the create finish quickly.** Teardown nils the handles and closes the database, the create's next control-db touch fails immediately, and it terminates — so in exactly the runs where teardown won the race hardest, the job was terminal by the time Close returned and the timing assertion passed. The symptom masks itself, and it masks itself MORE the worse the bug is.

**The rewrite.** Ask what happened to the create rather than when. With the drain removed it is destroyed every time, in two roughly even flavours — `repo manager is not running` (handles nilled under it) and `registry: sql: database is closed` (db closed under it). With the drain it reports `context canceled`: asked to stop, stopped. Asserting "the terminal error is not a teardown casualty" is **30/30 red without the drain and 0/20 with it, on the CHEAPEST fixture**. The 200-fact helper was built, measured, and deleted rather than kept as decoration — a fixture that no longer earns its cost is not evidence of rigour.

**Relation to guard-setup-outruns-hazard.** That fact is the case where the test's own SETUP finishes the hazardous work before the hazard can occur, and its remedy is exactly the one that fails here: choose a fixture that holds the guarded window open, and assert it. Both remedies are right in their own case, and the distinguishing question is **who moves the timing** — if it is the setup, re-choose the fixture; if it is the BUG, re-choose the question. Keep the in-flight precondition assertion from that fact either way: it is what makes a fixture that stops discriminating fail loudly instead of going quiet.

**What this does NOT mean.** It is not "timing assertions are bad" — the one here was kept as a secondary check, because it can never fail while the drain works and it costs nothing. It is not a licence to skip measuring the red rate: every number above came from actually removing the fix and counting, and the whole finding exists because the first version LOOKED fine and was 10-35% inert.
