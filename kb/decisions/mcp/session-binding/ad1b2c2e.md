---
type: observation
domain: [mcp, binding, chat]
confidence: 0.85
sources: 1
entities: [knomit_bind, BindingInstructions, BindingInstructionsDelta, WithInstructionsHeld, InstructionsHeld, unboundAddendum]
motifs: [duplicated-context-cost]
refs: ['src://7b4887ce51d9/internal/edge/mcp/bind.go@e4b4ddbbb82cabf2b8d239e4478cc7798fca45aa:05a32b13b307865d84ded4346495773ade289a81', 'src://7b4887ce51d9/internal/edge/mcp/instructions.go@e4b4ddbbb82cabf2b8d239e4478cc7798fca45aa:2a1f9e118014541a025dcb41e33aaa476ba6f0b4', 'src://7b4887ce51d9/internal/edge/mcp/bind_test.go@e4b4ddbbb82cabf2b8d239e4478cc7798fca45aa:c69fc41bfc34dae23092555ada5c39ee85c56249']
---
# knomit_bind re-sends the full server instructions to external clients on purpose; the in-process chat gets only the per-binding delta

On the UNSCOPED mount, initialize can only carry the instructions built from the default ontology (no topic list) and the code profile plus unboundAddendum, which tells the client to treat the knomit_bind result 'as if it were these instructions'. So the bind result carries the bound base's full instructions: this is deliberate, and clients that ignore initialize instructions depend on it. The in-process chat is the exception: its system prompt already holds the initialize instructions (hook 3), and requestContext marks its calls with WithInstructionsHeld, so knomit_bind returns BindingInstructionsDelta instead: ontology root, topic list, the write repo's profile addendum ONLY when it differs from initializeProfile ("code", the profile the initialize text was built with; the same addendum is not repeated) and (for a lens) the mount table and lens addendum. Measured at the merge commit with TestBind_ChatGetsOnlyThePerBindingDelta (bindFixture lens "all": a write repo plus two read repos; the test logs both sizes): 23776 bytes external, 4148 bytes chat (about 4.1 KB). The 5419-byte chat figure before the repeated profile addendum was dropped is from an earlier state of the branch and was not re-measured. Sizes move with the fixture and the instruction text, so treat them as the order of magnitude: the chat gets roughly a sixth of the external text. TestBindingInstructions_Composition pins the external text to baseInstructionsText + profile addendum + lensInstructions.

CONSEQUENCE: anything that becomes bound-base-specific in the instructions must live in BindingInstructionsDelta too, or the chat never sees it. The server-wide text (tools, fact fields, motifs) is not repeated to the chat.

WHAT THIS DOES NOT MEAN: the delta is not for external MCP clients; do not drop the full text for them, since an external client may ignore initialize instructions. URL-scoped endpoints already send no instructions in bind (scopedHandleBanner).
