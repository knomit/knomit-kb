---
type: observation
domain: [local-models, materialize, evaluation, article-writing]
confidence: 0.75
sources: 1
entities: [LFM2.5-2.6B, Claude Haiku 5.5, does-not-mean caveats, blind grading]
motifs: [quote-true-claim-false, task-shape-fit]
refs: ['kb://3ec012f5b4d2/kb/meta/research/ingestion/lfm-fit/e7e851aa.md']
---
# LFM2.5-2.6B as an article WRITER from facts (even given an outline and per-fact claims) inverted 'does not mean' caveats, duplicated and truncated text, and ranked last in both blind-graded articles

Measured 2026-10-26 (materialize research spike, round 3; evidence in the git-ignored .claude/plans/materialize-spike/round3/ directory, so repo-relative, not a pinned ref). Two articles were written from fact sets: art1 = 25 facts (15.5k words of source) for 'latest trends in agentic development', art2 = 40 facts (3.4k words) for 'what's new in the last 14 days'. Three writers: W1 = Claude Haiku in a pipeline (per-fact claim extraction, outline, section writing), W2 = LFM2.5-2.6B Q4_K_M writing the sections from W1's extracted claims and W1's outline (temperature 0.1, num_ctx 16384, num_predict 1500, thinking bypassed), W3 = Haiku in one call over the raw fact bodies. A grader read the three variants blind and checked 30 randomly drawn cited sentences each against the cited fact.

W2 results: ranked LAST in both articles. Mechanical checks: art1 253 sentences of which only 180 distinct, 18 of 25 facts cited, caveat retention 0.742 (vs 0.996 for W1), 6 hard number violations (a number found in no parent fact); art2 217 sentences of which 153 distinct, 38 of 40 facts cited, 2 invalid fact ids, caveat retention 0.828 (W1 0.99), 15 hard number violations. 7 of its 15 LFM calls hit the 1500-token cap. Blind-grade defects: duplicated blocks, sentences truncated mid-way, sections whose content did not match the heading, citations drifting by one fact, a closing paragraph repeated about six times (art2), and inverted rules (art1: 'Routing does not add a failure point if one prompt handles all categories'; 'Multi-agent systems work best when subtask outputs must be mutually consistent'; art2: 'The advisory does not mean patching to the Oct 21 emergency releases does not remove this risk'; an opt-in turned into 'unless the user chooses not to share'). Usefulness for an agent reader: 1 of 5 in both.

The sampled-support rate hid the collapse: W2 scored 24 of 30 (art1) and 28 of 30 (art2) supported on the random sample while being unusable. Cheap guards the grader proposed: distinct-sentence ratio, a truncation check, a per-section topic check, and a targeted check of every negated or 'does not mean' sentence (negated caveats were the worst error in ALL six articles, Haiku's included, but at far lower rate).

Consequence: do not assign an LFM2.5 model of 2.6B or smaller the prose-writing step of a fact-to-article pipeline, even with extraction and outline done by a stronger model; if one is used anyway, gate its output on the mechanical guards above. WHAT THIS DOES NOT MEAN: it does not show that the 2.6B fails at the structuring steps (extraction here was Haiku's); round 2's table shows LFMs are weak at extraction too, but the classification-shaped jobs in the lfm-fit fact were not retested. Two articles, one prompt, one grader: a sample, not a benchmark.
