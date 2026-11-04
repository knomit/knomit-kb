---
kind: pragmatic
type: policy
domain: [llm, tools, error-handling]
confidence: 0.85
sources: 1
entities: [llm.ToolUse, ToolUse.RawArgs, ToolUse.ArgsErr, InvalidToolArgsError, ErrInvalidToolArgs, finishBlock, Snapshot.CanonicalPrefix, TestAnthropicChat_InvalidToolArgsAreTyped]
motifs: [validate-at-the-boundary]
refs: ['src://7b4887ce51d9/internal/engine/llm/history.go@2e1e4dbcbb7604c4e9e1285c8953fbca4a8ebaa0:98d583a45cbc8b622668112f98a8f0900ce76b30#L60-L95', 'src://7b4887ce51d9/internal/engine/llm/anthropic_v2.go@2e1e4dbcbb7604c4e9e1285c8953fbca4a8ebaa0:42bbfc3ddf1f6d89a794f9a58e895de2b0887e0c', 'src://7b4887ce51d9/internal/engine/llm/anthropic_v2_test.go@2e1e4dbcbb7604c4e9e1285c8953fbca4a8ebaa0:e40894e3a4c27e74b78763b5cd9881db491828e9', 'https://github.com/knomit/knomit/pull/757']
---
# llm.ToolUse.Args is always valid JSON: arguments a model streams as invalid JSON are stored as Args {} with the received bytes in RawArgs, and ToolUse.ArgsErr() returns a typed *InvalidToolArgsError the loop must answer as a tool error

AnthropicChat's finishBlock checks json.Valid on the concatenated input_json deltas of each tool_use block. Valid bytes become ToolUse.Args (empty becomes {}). Invalid bytes are NOT stored in Args: Args is {} and the bytes go to ToolUse.RawArgs (json raw_args). ToolUse.ArgsErr() is nil for a valid call and otherwise an *InvalidToolArgsError{ToolUseID, Name, Raw} that Is ErrInvalidToolArgs. The EventToolUse carries the same *ToolUse.

Why: invalid RawMessage in a snapshot made CanonicalPrefix panic and json.Marshal of the entry and of the Anthropic request fail (reviewer N5). With Args always valid, the entry marshals, canonicalizes and renders (input {}). Pinned by TestAnthropicChat_InvalidToolArgsAreTyped.

Consequence: the agent loop (PR4) must check ArgsErr() before calling the host and answer such a call with an IsError tool_result naming the problem, never run it with {}. What it does NOT mean: Args {} is not the model's intent, and a call with ArgsErr != nil must never be executed. Any other adapter (Ollama, PR3) must keep the same contract.
