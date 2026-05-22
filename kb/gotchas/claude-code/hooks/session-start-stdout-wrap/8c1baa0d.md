---
type: observation
domain: [claude-code, hooks]
confidence: 0.85
sources: 0
entities: [SessionStart, PostToolUse, PreCompact, Stop, hookSpecificOutput.additionalContext]
refs: ['src://knomit/.claude/plans/2026-05-20-knomit-cc-memory-integration-design.md@0938d83']
---
# CC SessionStart auto-wraps stdout as system reminder; other hooks use JSON

Claude Code's SessionStart hook treats plain stdout as a system reminder automatically — emit plain text, no wrapper tags or JSON needed. All OTHER hooks that need to inject context (PostToolUse, PreCompact, Stop, UserPromptSubmit) must emit JSON of the form {"hookSpecificOutput":{"additionalContext":"…"}}. Mixing these up: emitting JSON from SessionStart yields literal JSON in the preamble; emitting plain text from PreCompact/Stop yields nothing injected.
