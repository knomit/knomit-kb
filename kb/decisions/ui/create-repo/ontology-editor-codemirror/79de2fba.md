---
type: observation
domain: [ui, web, ontology, fact, desktop]
confidence: 0.95
sources: 1
entities: [fact.ParseOntology, fact.ValidateOntologyYAML, Ontology, OntologyNode, Validation, GET /api/v1/ontologies/schema, 'POST /api/v1/ontologies:validate', OntologyEditor.tsx, CodeMirror, internal/fact/ontology.go, web/src/CreateRepoForm.tsx]
refs: ['src://7b4887ce51d9/internal/fact/ontology.go@b3f65a99179a88c37665f99f50403fef3af05a62:331a061bdc1861dafca160146b49a6a798925a7e', 'src://7b4887ce51d9/web/src/CreateRepoForm.tsx@b3f65a99179a88c37665f99f50403fef3af05a62:9b8e8a569089cb5dd4602c09e16ce10c96997b5a', 'kb://3ec012f5b4d2/kb/decisions/fact/per-repo-ontology/0be51fec.md', 'kb://3ec012f5b4d2/kb/decisions/repos/kb-repo-layout/ontology-dot-domains-fallback/318f619a.md']
---
# The custom-ontology editor is CodeMirror 6 with completions served FROM GO — a hand-maintained TypeScript copy of the 11 ontology field names is the version of this feature that was rejected

# Context

Custom ontology was a bare `<textarea>`. YAML is indentation-sensitive and Tab moves focus, so it offered no help with the one affordance the format most needs — plus no upload, no validation before submit, and no feedback until `Manager.Create` failed mid-stream.

# Options considered

1. **Upload only** — drop the editor; author YAML in your own editor. Smallest, but no way to tweak a preset in-app.
2. **Upload + plain textarea + explicit server-side Validate.** REJECTED — it carries the expensive part (parser line numbers) while delivering none of the ergonomics. No indentation help, no completions.
3. **Upload + CodeMirror 6 with schema completions** — CHOSEN.

# Rationale

The initial recommendation was option 2, on the grounds that a client-side schema is a second source of truth that will drift. That reasoning was WRONG here, for two measured reasons:

- **The shape is tiny.** 11 field names across 3 structs: `Ontology`{id,name,description,topics,validations}, `OntologyNode`{description,children,validations}, `Validation`{name,message,rule}. "A schema that will drift" is a fair worry about 200 fields; it is a weak one about 11.
- **The expensive work is shared.** Multi-error collection and `yaml.Node.Line` positions inside the parser are needed by options 2 AND 3 alike. Charging that to option 2 and then charging option 3 for the editor on top made the last rung look like the tall one when it is the shortest.

# The choice, and its CONDITION

CodeMirror 6 (yaml mode, lint, autocomplete). Diagnostics come from the real Go parser over HTTP, so the wizard can never bless YAML that `Manager.Create` then rejects.

**The completion field list MUST be served from Go (`GET /api/v1/ontologies/schema`) with a reflection test asserting every `yaml:` tag on those three structs appears in the served list.** This is a condition of the decision, not a nicety: a drifted completion list teaches confidently wrong field names, which is worse than no completions. The test is what turns drift into a build failure instead of a silent lie. A hand-maintained TypeScript copy is the version of this feature that was rejected.

# Known risk

CM6 edits in a `contenteditable`. `CreateRepoForm` already carries a comment recording the desktop WKWebView autocapitalizing typed input ("test" → "Test"), and ontology keys are lowercase-only — the same class of bug would corrupt YAML keys. Verify CM6 in the desktop build BEFORE building step 3 on it.

# Non-scope

Does not authorise a general-purpose code editor elsewhere in the app, and does not make ontologies editable post-creation.
