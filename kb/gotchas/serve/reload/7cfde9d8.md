---
type: observation
domain: [serve, config, lockfile, security]
confidence: 0.4
sources: 1
entities: [reload_error, restart_needed, Listeners.Reload, server.json]
motifs: [inconsistent-redaction]
refs: ['src://7b4887ce51d9/internal/app/listeners_reload.go@4a9018c4eedc0991816c2152ca3700f656e5e9af:338bd825934eb798eb67b6630ab6e7c60356ea73', 'src://7b4887ce51d9/internal/app/listeners_reload_test.go@4a9018c4eedc0991816c2152ca3700f656e5e9af:3dec1c4aba83b15d88512e1b622d6f8d2dafc522', 'src://7b4887ce51d9/internal/serveraddr/lockfile.go@4a9018c4eedc0991816c2152ca3700f656e5e9af:d211b8f4810023d4cb9ef863febf0d2ba5eeff10', 'https://github.com/knomit/knomit/pull/497']
---
# `reload_error` carries the loader's message verbatim, and a TOML parse error can quote the offending token, so a mistyped secret-bearing key could land its fragment in server.json (0600) while restart_needed deliberately holds names only

UNVERIFIED reviewer taste note on knomit#497 (round 2); not reproduced with a real secret. The restart_needed list is names-only on purpose and a test pins that no config value reaches server.json. `reload_error` ('<RFC3339Nano UTC time>: <message>') is different: the message is whatever config.Load returned, and TOML parse errors can quote the offending token. If someone mistypes a secret-bearing key (for example an llm.api_key line), the quoted fragment is written to server.json, which is 0600 and owner-only, and to the server log, where the same error was already logged at ERROR before this field existed. So the exposure is no wider than the log; the inconsistency is with the names-only rule.

CONSEQUENCE: do not assume everything in server.json is value-free; if reload_error is ever shown in a UI or copied into a bug report, treat it like a log line. A possible fix is trimming the message to key and line. Settle it by feeding a malformed secret-bearing line to Reload and reading the file.
