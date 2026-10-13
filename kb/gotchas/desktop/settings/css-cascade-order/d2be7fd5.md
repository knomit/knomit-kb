---
type: observation
domain: [desktop, ui, css]
confidence: 0.85
sources: 1
entities: [settings.tsx, index.css, App.css, '.k-select:disabled']
motifs: [import-order-cascade, shorthand-resets-longhand]
refs: ['src://7b4887ce51d9/tools/desktop/ui/src/settings.tsx@545994a2740bf7b48696604331f114eab1f2aa3f:edbf7f1f9c74cba0644ad696cc2134ef6e29d126#L3-L7', 'src://7b4887ce51d9/tools/desktop/ui/src/index.css@545994a2740bf7b48696604331f114eab1f2aa3f:5c0f287affa6e3093ba051542b84924715ed6a13#L149-L153', 'src://7b4887ce51d9/tools/desktop/ui/src/App.css@545994a2740bf7b48696604331f114eab1f2aa3f:082b189553e09f3d6f348735f7b58e302e50be44#L368-L381']
---
# In the desktop Settings bundle index.css loads AFTER App.css (settings.tsx imports SettingsApp, which imports App.css, before './index.css'), so an index.css rule of equal specificity wins — its `.k-select:disabled { background: ... }` shorthand silently drops a background-image set in App.css

tools/desktop/ui/src/settings.tsx imports './SettingsApp.tsx' (which imports './App.css') before './index.css', so the primitives file is LATER in the cascade than the component file, the opposite of what its 'tokens first' role suggests. Any App.css rule with the same specificity as an index.css rule loses. Observed 2026-10-13 in Chromium: the env-owned Log level <select> lost its chevron because index.css `.k-select:disabled` sets `background: #0e0e0e` (shorthand, resets background-image) and outranked App.css's `.ctl > .k-select` chevron for background-image. Fixed in PR #510 by restating the image in a more specific `.ctl > .k-select:disabled` rule.

CONSEQUENCE: when an App.css override seems ignored, check for an index.css rule of equal or higher specificity, and especially for a shorthand (`background`, `border`, `font`) there. Raise specificity in App.css or restate the longhands; do not reorder the imports without checking every primitive.

NOT MEANT: it does not say index.css should be imported first; nobody has decided that.
