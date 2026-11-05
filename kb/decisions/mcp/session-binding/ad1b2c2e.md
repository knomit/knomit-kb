---
type: observation
domain: [mcp, binding, chat]
confidence: 0.85
sources: 1
entities: [knomit_bind, BindingInstructions, BindingInstructionsDelta, WithInstructionsHeld, InstructionsHeld, unboundAddendum]
motifs: [duplicated-context-cost]
refs: ['src://7b4887ce51d9/internal/edge/mcp/bind.go@1da2f7078baaf9b497f76c1dc2d12b9838accf20:05a32b13b307865d84ded4346495773ade289a81', 'src://7b4887ce51d9/internal/edge/mcp/instructions.go@1da2f7078baaf9b497f76c1dc2d12b9838accf20:a47efeb72e7fc95483e782ea0a2833d563f0b68c']
---
# knomit_bind re-sends the full server instructions to external clients on purpose; the in-process chat gets only the per-binding delta

On the UNSCOPED mount, initialize can only carry the instructions built from the default ontology (no topic list) and the code profile plus unboundAddendum, which tells the client to treat the knomit_bind result 'as if it were these instructions'. So the bind result carries the bound base's full instructions: this is deliberate, and clients that ignore initialize instructions depend on it. The in-process chat is the exception: its system prompt already holds the initialize instructions (hook 3), and requestContext marks its calls with WithInstructionsHeld, so knomit_bind returns BindingInstructionsDelta instead: ontology root, topic list, profile addendum and (for a lens) the mount table and lens addendum. Measured on a lens of three repos: 23776 bytes external, 5419 bytes chat.

CONSEQUENCE: anything that becomes bound-base-specific in the instructions must live in BindingInstructionsDelta too, or the chat never sees it. The server-wide text (tools, fact fields, motifs) is not repeated to the chat.

WHAT THIS DOES NOT MEAN: the delta is not for external MCP clients; do not drop the full text for them, since an external client may ignore initialize instructions. URL-scoped endpoints already send no instructions in bind (scopedHandleBanner).
