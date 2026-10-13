---
type: observation
domain: [desktop, settings, ui, design-language]
confidence: 0.9
sources: 1
entities: [SettingsForm, Row, PAGES, PAGE_OF, pagesWithErrors, settingsWindowOptions, settingsWindowName, icons.tsx, FleetIdentitySection]
motifs: [transaction-boundary-over-layout, reversed-design-ruling]
refs: ['kb://3ec012f5b4d2/kb/invariants/desktop/fleet/install-surface/a46bbb69.md', 'kb://3ec012f5b4d2/kb/principles/ui/motif/4b9101b3.md', 'kb://3ec012f5b4d2/kb/decisions/desktop/settings/http-port-scope/ad18a6d0.md', 'src://7b4887ce51d9/tools/desktop/ui/src/SettingsForm.tsx@545994a2740bf7b48696604331f114eab1f2aa3f:ea58706205bb2facd1f97de21d82776d4307eebb#L159-L159', 'src://7b4887ce51d9/tools/desktop/ui/src/SettingsForm.test.tsx@545994a2740bf7b48696604331f114eab1f2aa3f:a100df12062fa264f1648dde91d24db1f226daf7#L124-L124', 'src://7b4887ce51d9/tools/desktop/windows.go@545994a2740bf7b48696604331f114eab1f2aa3f:4eb7f2921b456a260f595cdd0111964e2564ce2d#L41-L62', 'https://github.com/knomit/knomit/pull/510']
---
# The desktop Settings window is a 680x500 sidebar of four pages (General, Network, Fleet, Logs) with ONE transactional Cancel/Save across all of them; the user chose this (variant A) over an instant-apply layout and a status-first layout, and it reverses the earlier 'one grouped list, not icon-headed sections' ruling

DECISION (2026-10-13, PR #510 against dev, branch feat/desktop-settings-sidebar). The user reviewed three mockups of the desktop Settings window and picked variant A, 'Orb-faithful, transactional'.

OPTIONS:
- A, transactional (CHOSEN): sidebar General / Network / Fleet / Logs with glyphs copied from web/src/icons.tsx; cards of rows (label + muted sublabel left, control right); the Cancel/Save footer and the restart offer stay pinned under every page, so SaveSettings still validates every field together in one call.
- B, instant-apply: no footer, each control commits on change. Rejected cost: every commit is a SaveSettings call, i.e. a knomit.toml write AND a server reload (Off -> This computer -> All networks reloads three times), no Cancel, and Go would have to accept single-field saves.
- C, status-first: the server state as a card at the top of the sidebar, both listeners on one 'Listeners' page, trust material on its own page, a save bar that appears only when edits are pending ('Revert' instead of Cancel). Rejected: one more concept (pending bar) and a renamed Cancel.

RATIONALE for A: keeps the save model and its tests (one transaction, Save/Cancel names, restart as an outcome of Save) while giving four landmarks to navigate. Its known costs, accepted: edits on a page you have left are invisible until Save; a refusal can sit on a page not in view (mitigated, see below); the Fleet listener ADDRESS lives on Network while its live STATUS lives on Fleet; the bundle Install acts at once, not through Save.

REVERSAL: SettingsForm.tsx used to carry a comment recording the opposite ruling ('One grouped list, not four icon-headed sections... the scaffolding outweighed the content'). That ruling is consciously reversed by the user's choice; the comment was replaced at the nav.

CONSEQUENCES a later change must keep: (1) the window Name stays settingsWindowName (it gates the bindings; see the install-surface invariant); (2) pages stay MOUNTED and are hidden with the `hidden` attribute, so form edits and an open fleet-root confirmation survive a look at another page; (3) a refused field marks its sidebar item with a red dot (accessible name 'Network has an error') and a failed Save switches to the first page holding an error unless the page in view holds one (PAGE_OF maps field -> page; a new validated field needs an entry or its error can be invisible); (4) the selection pill is neutral, red is the only accent the sidebar uses, and only for an error; (5) a missing section (no fleet props) is omitted from the sidebar, never shown disabled.

WHAT THIS DOES NOT MEAN: it is not a move to per-control saving; Install bundle was already immediate and still is. Nor does 'icon-headed sections' license icons elsewhere in the window: only the four sidebar items and the Copy public key link carry glyphs.
