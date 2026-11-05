---
type: concept
domain: [llm, chat, config]
confidence: 0.9
sources: 1
entities: [Caps.StablePrefix, Registry.Adaptive, AnthropicChat.Caps, OllamaChat.Caps, adaptive_tools, LLMProviderConfig.AdaptiveTools, TestLoop_StablePrefixRequestsAreByteIdentical, ErrWindowWithStablePrefix, TestLoop_RefusesWindowWithStablePrefix]
motifs: [capability-over-kind]
refs: ['kb://3ec012f5b4d2/kb/invariants/llm/history/append-only/80e22727.md', 'src://7b4887ce51d9/internal/engine/llm/chat.go@2199e0370f837652478a0e4f95596f37c04c3dcd:302c8b27cb4ba006d5241541e5b6b6f9d47443e2', 'src://7b4887ce51d9/internal/engine/llm/stableprefix_test.go@2199e0370f837652478a0e4f95596f37c04c3dcd:0c205a1d7d347a31c97ed6611784a913b44ece15']
---
# Caps.StablePrefix means changing the tool list or editing earlier history between requests is costly or invalid for the provider; Anthropic sets it true, Ollama false, and a chat is adaptive only when it is false and adaptive_tools is not off

Caps.StablePrefix is a capability of the adapter, read by the chat wiring, not a provider-kind check. True means a byte-stable prefix is required: either the prompt cache is keyed on it, or (Claude 5.1+) thinking-block signatures bind system, tools and earlier messages. AnthropicChat.Caps sets it; OllamaChat.Caps leaves it false (no cache to protect). WrapChat and the tracer delegate Caps unchanged, so the flag survives wrapping.

Config: [llm.providers.<name>] adaptive_tools is empty or auto (adaptive when !StablePrefix) or off (always every tool). There is no on: adaptive_tools = "on" for an anthropic provider is refused at config load with the approved sentence (its requests must keep a stable prefix), and any other unknown value is refused naming auto and off. Registry.Adaptive(provider, model, caps) = adaptive_tools != off && !caps.StablePrefix; an unknown provider or model is never adaptive.

Consequence: a provider with StablePrefix true must have Loop.Window nil, so its requests are Builder.Snapshot() byte for byte. This is enforced, not only intended: Loop.RunTurn returns ErrWindowWithStablePrefix before adding or sending anything when Window is set and Caps().StablePrefix is true (TestLoop_RefusesWindowWithStablePrefix). Whoever wires the Window (the chat/edge layer) must still use Registry.Adaptive to decide, because the guard fails the turn rather than choosing for you. TestLoop_StablePrefixRequestsAreByteIdentical pins a SHA-256 of the rendered Anthropic requests taken from the code before adaptive tools existed.

Does NOT mean: adaptive is chosen by the string "ollama". A new provider without a cache becomes adaptive by leaving StablePrefix false.
