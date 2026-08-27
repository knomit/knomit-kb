---
type: observation
domain: [ui, web, desktop, accessibility]
confidence: 0.9
sources: 2
entities: [basicSetup, defaultKeymap, indentWithTab, toggleTabFocusMode, EditorView.contentDOM, OntologyEditor, web/src/OntologyEditor.tsx, '@codemirror/commands']
motifs: [name-implies-absent-capability]
refs: ['src://7b4887ce51d9/web/src/wizardState.ts@cc5752af8d0cfe94c788ed7a099090e2bd9a354d:fdebe067487bd12a64e62c8df20d6c7a80b9befc', 'src://7b4887ce51d9/web/src/OntologyEditor.tsx@24853d4622e68c9b1035f2b298e870e0dd258d60:82e9201b07787641a93f55580076b9cbdb83bb81', 'kb://3ec012f5b4d2/kb/gotchas/web/editors/codemirror/b64862a6.md']
---
# CodeMirror's basicSetup does NOT bind Tab to indent — ship it as-is and Tab still moves focus, which is the exact complaint that made a plain textarea unacceptable

# The fact

`basicSetup` installs `defaultKeymap`, and `defaultKeymap` has **no `Tab` binding**. Verified against `@codemirror/commands` — neither `defaultKeymap` nor `standardKeymap` binds it. So a CodeMirror editor with only `basicSetup` behaves like a `<textarea>` for Tab: focus moves to the next control instead of indenting.

For an indentation-sensitive format like the ontology YAML, that defeats the entire reason for choosing a code editor over a textarea. Add `keymap.of([indentWithTab])` from `@codemirror/commands`, placed **ahead of `basicSetup`** in the extensions array so it wins under CodeMirror's facet ordering.

# The accessibility half — do not omit it

Binding Tab inside an editor is a known keyboard trap. What makes it acceptable is that `defaultKeymap` also ships `{key:"Ctrl-m", mac:"Shift-Alt-m", run: toggleTabFocusMode}` — the escape hatch that lets a keyboard user restore Tab-as-focus. **`indentWithTab` is only acceptable while that escape hatch is present**; if `basicSetup` is ever replaced with a hand-rolled extension set, the toggle must be carried over deliberately.

# The WKWebView guard, which is separate and also load-bearing

CodeMirror edits in a `contenteditable`. knomit's desktop shell is a WKWebView, which autocapitalizes and autocorrects typed text — this already broke the repo-name input once ("test" became "Test" and failed the lowercase-only `isValidRepoName` with a confusing 400). Ontology topic keys are lowercase-kebab-only, so the same behaviour here is SILENT DATA CORRUPTION. Set `autocapitalize="off"`, `autocorrect="off"` and `spellcheck="false"` on `view.contentDOM`.

**No automated check available in CI can verify that guard.** A spike proved it: removing the three attributes produced byte-identical results under a real WebKit engine, because Playwright's synthetic `keyboard.type()` never routes through the macOS `NSSpellChecker` substitution service that causes the bug. The attributes are unit-tested for PRESENCE; their EFFECT can only be confirmed by a human typing in the packaged app. Treat a green CI run as saying nothing about this.
