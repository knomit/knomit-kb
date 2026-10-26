---
type: reference
domain: [llm, cost, materialize, pricing]
confidence: 0.55
sources: 1
entities: [Claude Haiku 5.5, OpenRouter, CometAPI, Requesty, AIMLAPI, AWS Bedrock, Google Vertex, Azure Foundry]
motifs: [dated-snapshot]
refs: ['kb://3ec012f5b4d2/kb/meta/research/materialize/haiku-cost/27c962bf.md', 'https://platform.claude.com/docs/en/about-claude/pricing.md', 'https://openrouter.ai/pricing', 'https://openrouter.ai/api/v1/models', 'https://www.cometapi.com/pricing/', 'https://www.requesty.ai/models/anthropic/claude-haiku-5-5', 'https://www.anthropic.com/legal/commercial-terms']
expires: "2026-12-01T00:00:00Z"
---
# As of 2026-10-26: no legitimate gateway sells Claude Haiku 5.5 below Anthropic list ($0.10 in / $0.50 out per MTok up to 100k tokens); OpenRouter passes list plus a 5.5% credit fee; CometAPI lists 20% below list but was not confirmed as Anthropic-authorized

DATED SNAPSHOT, fetched 2026-10-26 (materialize research spike, round 6; evidence in the git-ignored .claude/plans/materialize-spike/round6-local-models/GATEWAYS.md). Prices change; re-check before reusing, which is why the fact carries an expiry. No paid API was called.

Anthropic list for Haiku 5.5, prompts up to 100k tokens: $0.10 input / $0.50 output per MTok; 5-minute cache write $0.125; 1-hour cache write $0.20; cache read $0.01; batch -50%; over 100k tokens $0.50 / $2.50. Minimum cacheable prefix 512 tokens. OpenRouter's endpoints for Anthropic, Bedrock, Vertex and Azure all showed $0.10 / $0.50 (EU/US-pinned endpoints $0.11 / $0.55). OpenRouter charges a 5.5% fee on credit purchases, so credits cost 1.055x list; its BYOK has no fee on the first $25,000 per month. Requesty: list plus 5% on pay-as-you-go. AIMLAPI: $0.13 / $0.65 per its own listing as seen in a search snippet (above list, not verified on its page). CometAPI: $0.08 input on its pricing page ('20% discount') and $0.08 / $0.40 on a product page; searches found no evidence that it is an Anthropic-authorized reseller, and Anthropic's commercial terms prohibit reselling except as expressly approved, so its supply chain and data handling are unknown. Together, DeepInfra, Fireworks, Groq and Cerebras did not offer Haiku 5.5 (they appeared on OpenRouter only for open models).

WHAT THIS DOES NOT MEAN: 'no gateway is cheaper' means none was found that is both below list and demonstrably authorized; it is not a claim that CometAPI's price is false, or that a buyer using it breaks a rule (the reading of Anthropic's terms was the spike's own inference, not legal advice). The AWS, Google and Azure rows came from OpenRouter's passthrough view, not from the clouds' own pricing pages; batch availability on those clouds was not verified. Retention and training policies of Requesty, CometAPI and AIMLAPI were not found.
