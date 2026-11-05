---
type: observation
domain: [llm, ollama, context-window]
confidence: 0.7
sources: 1
entities: [qwen35, kvBytesPerToken, full_attention_interval, head_count_kv]
motifs: [measured-environment-not-property]
refs: ['src://7b4887ce51d9/internal/engine/llm/numctx_test.go@d9f976a76541cb23badbc702a6d963b97f3006a7:67dc6fc23f96b2baca718ef30954db6e779595b0']
---
# One machine's measurement: qwen35 (Qwen3.5-4B) reports 32 blocks, a full-attention layer every 4th, 4 KV heads and 256-wide keys and values, so about 32 KiB of f16 KV per token (about 8 GiB at its 262144 trained length); a dense 36-layer, 8-KV-head, 128-wide model would be about 144 KiB per token

Read live from Ollama 0.40.14 /api/show on the user's machine, 2026-11-05: qwen35.context_length 262144, block_count 32, full_attention_interval 4, attention.head_count_kv 4, key_length and value_length 256. KV per token = 8 full-attention layers x (256+256) x 4 heads x 2 bytes = 32768 bytes, so 262144 tokens is about 8 GiB and 32768 tokens about 1 GiB. The dense comparison (36 layers x 8 heads x (128+128) x 2 = 147456 bytes per token, about 36 GiB at 262144) is an illustration, not a measurement.

This is ONE model on ONE machine: it is a test fixture in numctx_test.go (hybridInfo, denseInfo), not a property of the code. No constant is derived from it. Does NOT mean the trained length is safe in general: it is safe for this hybrid model on a large machine and unsafe for dense models, which is why the window is derived per model and never taken as the trained length alone.
