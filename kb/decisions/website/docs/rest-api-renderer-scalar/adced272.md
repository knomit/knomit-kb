---
type: observation
domain: [website, docs, openapi]
confidence: 0.95
sources: 0
entities: [astro.config.mjs, src/generated/openapi.yaml, starlight-openapi, '@scalar/astro', starlightOpenAPI, openAPISidebarGroups]
refs: ['src://knomit-io/astro.config.mjs@e19ab32', 'src://knomit-io/src/generated/openapi.yaml@e19ab32']
---
# Render the REST API reference as a single Scalar page, replacing starlight-openapi

The knomit.io docs site (Astro + Starlight) renders the REST API reference. starlight-openapi generated one route per operation (61 operations across 13 tags), which the user found bare and fragmented. Decision: render the whole API as a single page, themed to the subdued knomit dark palette.

**Options considered** (via AskUserQuestion):
1. Scalar, themed — drop starlight-openapi, render the spec with Scalar as one scrollable page, themed via Scalar CSS variables to the knomit palette. Reads the OpenAPI spec directly so it stays in sync.
2. Swagger UI default — the literal Swagger look (one column, tag sections, expandable rows, Try-it-out), but its chrome resists matching the subdued theme.
3. Custom-built native renderer — single page in our own components/tokens; perfect match but most work and ongoing upkeep as the spec grows.

**Rationale** — Scalar won: modern and interactive, themeable to the dark palette via documented CSS variables (matches the whole session's goal of a subdued, on-brand look), and low-maintenance because it consumes src/generated/openapi.yaml directly. Swagger UI lost on theme-match (dated chrome). Custom lost on effort + maintenance burden for OpenAPI edge cases.

**The choice** — Scalar, themed dark, as a single page; remove the starlightOpenAPI plugin and its openAPISidebarGroups; the Starlight sidebar gets a single REST API entry pointing at the Scalar page.
