---
type: observation
domain: [web, ui, shell, server]
confidence: 0.85
sources: 1
entities: [serverPagesFor, accessTabsFor, AccessPage, AccessTab, useOAuthAvailability, ServerPages, SecretsTab, AuthPage]
refs: ['src://7b4887ce51d9/web/src/facets.ts@b421c6cb73226b64efba5f7f3a9541016292fc4a:78675fee1deb328ae76192dbf163e88f5aae8dcd', 'src://7b4887ce51d9/web/src/AccessPage.tsx@b421c6cb73226b64efba5f7f3a9541016292fc4a:fe31eac3607c6710faacbd7dc9c5a9b40c9bc878', 'src://7b4887ce51d9/web/src/App.tsx@b421c6cb73226b64efba5f7f3a9541016292fc4a:34c23ae627eadacba4a8ce06ddffef92eefc0fe2', 'kb://3ec012f5b4d2/kb/decisions/web/shell/facets/server-pages-behind-gear/911dd19a.md']
---
# The gear's server pages are Logs and Access (serverPagesFor); Access has tabs Secrets (always) and Authorizations (only with an OAuth issuer, accessTabsFor in facets.ts), and App owns the Access tab so useOAuthAvailability knows when Authorizations polls for itself

`serverPagesFor(serverReadOnly)` returns `[]` on a read-only server (no gear) and `['logs', 'access']` otherwise; it no longer takes the OAuth flag. `accessTabsFor(oauthAvailable)` in web/src/facets.ts returns `['secrets']` or `['secrets', 'auth']`; a missing tab is omitted, never disabled, and AccessPage falls back to Secrets when asked for a tab that is gone. AccessPage (web/src/AccessPage.tsx) renders its own small tab strip and then SecretsTab or AuthPage (ManageOAuth.tsx). The tab state lives in App (`accessTab`), because `useOAuthAvailability`'s `pageOpen` must be true exactly while Authorizations is on screen (`view.server === 'access' && accessTab === 'auth'`); that page polls and reports its count itself. The waiting-authorizations badge is on the server tab (`server-tab-access-badge`) and on the inner tab (`access-tab-auth-badge`).

Consequence: a later tab (Connected accounts) is added to AccessTab and accessTabsFor, not as a new ServerPage. Test ids `server-tab-auth` and `server-tab-auth-badge` no longer exist.
