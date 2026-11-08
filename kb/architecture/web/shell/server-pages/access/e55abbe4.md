---
type: observation
domain: [web, ui, shell, server]
confidence: 0.85
sources: 1
entities: [serverPagesFor, accessTabsFor, AccessPage, AccessTab, useOAuthAvailability, ServerPages, SectionRail, AuthList]
refs: ['kb://3ec012f5b4d2/kb/decisions/web/shell/facets/server-pages-behind-gear/911dd19a.md', 'src://7b4887ce51d9/web/src/facets.ts@69c7f5daf9344d86df562fc7f2241d9f557d5eb3:50309fdaa7d589e5c5f752eb5d9f1473cddf677b', 'src://7b4887ce51d9/web/src/AccessPage.tsx@69c7f5daf9344d86df562fc7f2241d9f557d5eb3:898805fe89e86f96278ef712b2668f8eb2be9216', 'src://7b4887ce51d9/web/src/App.tsx@b421c6cb73226b64efba5f7f3a9541016292fc4a:34c23ae627eadacba4a8ce06ddffef92eefc0fe2']
---
# The gear's server pages are Access and Logs, in that order (serverPagesFor); Access has sections Secrets (always) and Authorizations (only with an OAuth issuer, accessTabsFor in facets.ts), and App owns the Access section so useOAuthAvailability knows when Authorizations polls for itself

`serverPagesFor(serverReadOnly)` returns `[]` on a read-only server (no gear) and `['access', 'logs']` otherwise (Access first since feat/access-layout; the gear still OPENS on Logs, see kb/architecture/web/shell/server-pages/order); it does not take the OAuth flag. `accessTabsFor(oauthAvailable)` in web/src/facets.ts returns `['secrets']` or `['secrets', 'auth']`; a missing section is omitted, never disabled, and AccessPage falls back to Secrets when asked for one that is gone. AccessPage (web/src/AccessPage.tsx) draws them as a SectionRail (test ids `access-section-secrets`, `access-section-auth`), then an item list and a detail panel (kb/architecture/web/shell/server-pages/access/layout). The section state lives in App (`accessTab`), because `useOAuthAvailability`'s `pageOpen` must be true exactly while Authorizations is on screen (`view.server === 'access' && accessTab === 'auth'`); that section polls and reports its count itself. The waiting-authorizations count is on the server tab (`server-tab-access-badge`) and on the rail's Authorizations row (a pill labelled `<n> waiting`).

Consequence: a later section (Connected accounts) is added to AccessTab and accessTabsFor plus a SectionEntry and a list/detail pair, not as a new ServerPage. Test ids `server-tab-auth`, `access-tab-*`, `secrets-tab` and `auth-page` no longer exist.
