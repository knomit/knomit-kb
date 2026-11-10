---
type: observation
domain: [web, ui, chat]
confidence: 0.9
sources: 1
entities: [unpackJson, JsonView, resultText]
refs: ['src://7b4887ce51d9/web/src/jsonViewText.ts@c094a1176b2c50ccbb46709f657232d79f96e197:26516ee843b69718a561a35cf1ffa6987b3bca29']
---
# In a chat result's JSON view, every string that is itself a JSON object or array is ALWAYS shown unpacked, recursively; there is deliberately no toggle (user decision, 2026-11-09)

jsonViewText.unpackJson replaces a string value with what it parses to when the string, trimmed, starts with '{' or '[' AND JSON.parse succeeds, then recurses into the result; it walks arrays and objects at every depth. A tool that returns JSON inside a string field (a review's `response`) is then read as structure instead of one escaped line. Strings that only look like JSON ("{not json}"), quoted scalars and numbers in strings stay strings.

Consequence: what the view (and a copy from it) shows is NOT byte-for-byte what the tool returned when a string held JSON; the copy is the exact pretty-printed text of the unpacked value. Do not add a raw/unpacked switch without a new decision: the user chose always-unpacked over the trial's toggle.

Keys are copied with Object.defineProperty, not `out[k] = ...` (review fix, PR #819): assigning "__proto__" on a plain object sets its prototype, so a result carrying a "__proto__" key lost it from the view. Consequence: any code that rebuilds parsed JSON objects key by key must define, not assign.
