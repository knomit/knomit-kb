---
type: observation
domain: [web, ui, shell, server]
confidence: 0.85
sources: 1
entities: [serverPagesFor, accessTabsFor, AccessPage, AccessTab, useOAuthAvailability, ServerPages, SectionRail, AuthList]
refs: ['kb://3ec012f5b4d2/kb/decisions/web/shell/facets/server-pages-behind-gear/911dd19a.md', 'src://7b4887ce51d9/web/src/facets.ts@f8ba30233d61af49fc1482eed100ba9404a019c8:3b8d52f0229ee7e85a1d189b92cd2c08dd503098', 'src://7b4887ce51d9/web/src/AccessPage.tsx@f8ba30233d61af49fc1482eed100ba9404a019c8:32871d0368c322060ac90f8438e69e47c5ed9d01', 'src://7b4887ce51d9/web/src/App.tsx@f8ba30233d61af49fc1482eed100ba9404a019c8:b4774823804762792989050eb43e9cbaaa031563', 'https://github.com/knomit/knomit/pull/729', 'https://github.com/knomit/knomit/pull/795']
---
# The gear's server pages are Access and Logs, in that order (serverPagesFor); Access has sections Secrets (always) and Authorizations (only with an OAuth issuer, accessTabsFor in facets.ts), and App owns the Access section so useOAuthAvailability knows when Authorizations polls for itself

`serverPagesFor(serverReadOnly)` returns `[]` on a read-only server (no gear) and `['access', 'logs']` otherwise (Access first since PR #795, and the gear OPENS on Access, see kb/architecture/web/shell/server-pages/order); it does not take the OAuth flag. `accessTabsFor(oauthAvailable)` in web/src/facets.ts returns `['secrets']` or `['secrets', 'auth']`; a missing section is omitted, never disabled, and AccessPage falls back to Secrets when asked for one that is gone. AccessPage (web/src/AccessPage.tsx) draws them as a SectionRail (test ids `access-section-secrets`, `access-section-auth`), then an item list and a detail panel (kb/architecture/web/shell/server-pages/access/layout). The section state lives in App (`accessTab`), because `useOAuthAvailability`'s `pageOpen` must be true exactly while Authorizations is on screen (`view.server === 'access' && accessTab === 'auth'`); that section polls and reports its count itself. The waiting-authorizations count is on the server tab (`server-tab-access-badge`) and on the rail's Authorizations row (a pill labelled `<n> waiting`).

Consequence: a later section (Connected accounts) is added to AccessTab and accessTabsFor plus a SectionEntry and a list/detail pair, not as a new ServerPage. Test ids `server-tab-auth`, `access-tab-*`, `secrets-tab` and `auth-page` no longer exist.
