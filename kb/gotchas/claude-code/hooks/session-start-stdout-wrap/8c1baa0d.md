---
type: observation
domain: [claude-code, hooks]
confidence: 0.85
sources: 2
entities: [SessionStart, PostToolUse, PreCompact, Stop, UserPromptSubmit, hookSpecificOutput.additionalContext, hookSessionStart, hookPreCompact, newCustomInstructions, tools/bridge/claude/hook_session_start.go]
refs: ['src://knomit/.claude/plans/2026-05-20-knomit-cc-memory-integration-design.md@0938d83', 'src://7b4887ce51d9/tools/bridge/claude/hook_session_start.go@9702a2832f22d45b6be003aee929746c43a0787a:0dd7f9cc6765f24ce837929cd764d6f11f8bd660#L20-L22', 'src://7b4887ce51d9/tools/bridge/claude/helpers.go@9702a2832f22d45b6be003aee929746c43a0787a:9448f5c376df25abcc4666dea27ef0bb6285a856#L217-L272', 'kb://3ec012f5b4d2/kb/gotchas/claude-code/hooks/additional-context-union/eb35b8d9.md', 'https://github.com/knomit/knomit/pull/76']
---
# CC SessionStart auto-wraps plain stdout as a system reminder — but "every OTHER hook injects via JSON" is FALSE: PreCompact must also emit plain text, and each event routes stdout somewhere different

**Still true, and verified at HEAD:** Claude Code's SessionStart hook treats plain stdout as a system reminder automatically. Emit plain text — no wrapper tags, no JSON envelope. `hookSessionStart` does exactly this and says so in its doc comment.

**CORRECTED (2026-08-10).** This fact previously claimed that *all* other context-injecting hooks — naming PostToolUse, **PreCompact**, Stop, UserPromptSubmit — must emit `{"hookSpecificOutput":{"additionalContext":"…"}}`, and that plain text from PreCompact yields nothing injected. **PreCompact was wrong, and acting on it caused a real bug.** `hookSpecificOutput` is a discriminated union and PreCompact carries NO `additionalContext` variant: a JSON envelope from PreCompact fails validation and CC discards the hook's entire result. PreCompact's channel is **plain stdout**, which CC reads as `newCustomInstructions` into the summarizer's prompt.

PostToolUse, Stop and UserPromptSubmit — the other three named here — do carry the variant and are unaffected by this correction. For the full twelve-event union and the two distinct ways to fail it, see `kb/gotchas/claude-code/hooks/additional-context-union/eb35b8d9.md`.

**The generalisation to actually carry away:** there is no global "JSON hooks vs stdout hooks" split. Whether an event accepts an `additionalContext` envelope, and where its plain stdout is routed if it does not, is **per-event**. SessionStart's stdout becomes a system reminder; PreCompact's becomes summarizer instructions; other events route theirs elsewhere or nowhere. Check the specific event before assuming either channel works.

**UNVERIFIED at CC 2.1.226:** the original claim that emitting JSON *from SessionStart* yields literal JSON in the preamble. SessionStart IS in the additionalContext union at 2.1.226, so a well-formed envelope naming `hookEventName: "SessionStart"` would be expected to validate — which suggests that claim described an older CC. Not re-tested. The bridge keeps emitting plain text from this hook regardless, because it is simpler and known-good; do not read this paragraph as a reason to change it.
