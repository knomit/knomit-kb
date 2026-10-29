---
kind: pragmatic
type: policy
domain: [web, ui, layout, settings]
confidence: 0.9
sources: 1
entities: [FacetColumns, indexMaxWidth, SettingsPage, SETTINGS_INDEX_MAX_PX, knomit.leftPanelWidth, leftPanelWidth, Library, BrowseFacet]
motifs: [opt-in-exception]
refs: ['kb://3ec012f5b4d2/kb/decisions/web/shell/facets/index-column-width/60dbb4f4.md', 'src://7b4887ce51d9/web/src/FacetColumns.tsx@6e98835813f6f77638b58b9dd786597513da3c09:25353a6e259cd8fa41a89fdfc8847140958c7461', 'src://7b4887ce51d9/web/src/SettingsPage.tsx@6e98835813f6f77638b58b9dd786597513da3c09:b14f19002e4c2c057d927c8d480356780258d833', 'https://github.com/knomit/knomit/pull/678']
---
# The Settings section nav is capped at 240px (FacetColumns indexMaxWidth), chosen over the shared draggable Library width; Settings has no drag handle and its divider sits at a different x than Browse

Options: (A) keep the Settings section nav at the shared, draggable Library width (35% of the window by default, observed 315-448px), so its divider sits at the same x as Browse's; (B) cap the Settings nav at 240px.

Choice: B, the 240px cap. A user ruling dated 2026-10-29. Implemented as the opt-in prop indexMaxWidth on FacetColumns; SettingsPage passes SETTINGS_INDEX_MAX_PX = 240 (PR #678). It narrows kb/decisions/web/shell/facets/index-column-width/60dbb4f4.md for Settings only.

Rationale: the nav labels are short section names (Description, Remote, Pushed branches...), so 240px fits them, and the content panel gains about 200px. The same PR removed the 760px max-width on the settings content, so the content panel now fills the rest of the pane.

Cost the user accepted: while the cap applies there is no drag handle on Settings (the splitter is not offered, because dragging could not move a capped column), and the Settings divider sits at a different x than on Browse.

Non-scope: Browse and the wizards (CreateRepoWizard, CreateLensForm, FleetWizard) do not pass indexMaxWidth and keep the shared, draggable Library width. The persisted knomit.leftPanelWidth is untouched by the cap; the cap is applied only at render, and only when the Library width exceeds it.

Does NOT mean: that every facet index is now capped, or that Settings ignores the Library width setting. A different facet that wants a capped index must opt in with indexMaxWidth itself.
