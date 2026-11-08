---
type: observation
domain: [web, ui, fact]
confidence: 0.95
sources: 1
entities: [FactMetaLine, FactBody, FactBand, FactBandPath, FactPage, FactView, renderFact, ChatFactChip, LensMeta, StatBox, RepoRows, repoHue, web/src/FactMetaLine.tsx, web/src/FactBand.tsx, web/src/FactBody.tsx, web/src/FactView.tsx]
refs: ['src://7b4887ce51d9/web/src/FactMetaLine.tsx@70826ce63e0332b8e01918eb049772babeeb25db:5144a214d7f3530164b4ea68defb7fa7067f400d', 'src://7b4887ce51d9/web/src/FactBand.tsx@70826ce63e0332b8e01918eb049772babeeb25db:e5837ce5709b8bc9d5f803a1d808d6be82da4079#L65-L87', 'src://7b4887ce51d9/web/src/FactChrome.tsx@70826ce63e0332b8e01918eb049772babeeb25db:64ef9fa62e0faa19d791f03de9ee9e5ce0c3e0c0#L295-L335', 'src://7b4887ce51d9/web/src/FactView.tsx@70826ce63e0332b8e01918eb049772babeeb25db:5c7171ed9d5d17f9e5d41bf914f4a9d0fe3c4ba5#L93-L122', 'src://7b4887ce51d9/web/src/FactBody.tsx@70826ce63e0332b8e01918eb049772babeeb25db:7935c0de5417be8f1a12090f6d0c44ba399bd73d#L326-L334', 'kb://3ec012f5b4d2/kb/decisions/ui/summary-panel/repos-ranked-rows-with-activity/fed2c140.md', 'kb://3ec012f5b4d2/kb/decisions/ui/factbody-shared-component/4cd05433.md', 'https://github.com/knomit/knomit/pull/70']
---
# The fact header is ONE dot-separated meta line, and the lens mount is drawn the dashboard's way (hue dot + plain mono name) — lightening the badge is what let the mount stay on the line instead of moving elsewhere

The fact header stood 233px between the title and the first line of prose, stacking four rows in three visual languages: a mono path line, a chip row (kind/type/origin), and two 70px StatBoxes holding one number each. Now 98px.

**Options considered (grammar)**
- **A — everything is a figure**: the dashboard's value-over-uppercase-label strip, with type and origin as figures too. ~112px. Rejected by the user in favour of B.
- **B — one line, no labels (CHOSEN)**: type · origin · conf X · N sources · [mount · branch] · path. ~74px in the mock, 98px live with a two-line title.
- **C — meta rail beside the body**: rejected — spends horizontal space the splitter can take below 600px, at the cost of the prose's line length.
- **D — type glyph leads the title, path as a breadcrumb above**: rejected, though it has the strongest hierarchy; it answers "what kind of fact" in two places.

**The mount placement question, and how it dissolved.** With B chosen, the lens mount looked like it had to move off the line (the pill plus a long branch name pushed the path off the end). Four placements were drawn: back on the line, an address line above the title, beside the controls, and a hue spine. The user then pointed at the dashboard's Repo rows and asked why that format was not used — which reframed the question from WHERE to WHICH TREATMENT.

The header drew a mount as a bordered, `repoHueBg`-filled pill. The dashboard's Repo rows draw the same thing as a 5px hue dot plus the plain mono name at #b9c1cd. That was a THIRD treatment for one concept (badge pill, dashboard row, and the Library's source badge), and it was the heavy one. Adopting the dashboard's dropped the mount's cost to roughly a third — and the placement problem went with it. The line now differs between contexts by exactly two extra tokens, which is what made B worth picking in the first place.

**The branch STAYS.** It was dropped from the early mocks by accident, not by decision, and it must not be: in a lens the top bar names the lens, its mount count and its write target — never a READ mount's branch. This line is the only place it is visible, and a read mount can sit on `main` while the write repo is on an agent branch.

**Two wording details that are load-bearing**
- "1 source" / "4 sources", never a bare number: the StatBox had a SOURCES label above the digit; on one line the word has to travel with it or the number means nothing.
- Parts are built as a LIST with separators interleaved, so a missing part never leaves a leading or trailing `·`. A stray dot is the tell that a header is printing a gap where a value should be.

**Structural consequence**: the chips and the StatBoxes moved OUT of FactBody into a new FactMetaLine (then rendered by RightPanel.renderFact). They were never body — they describe the fact, so they belong with the title and the controls. FactBody is now content only: prose, tags, refs. `LensMeta` was deleted.

**State at 70826ce6 (re-read).**
- FactMetaLine is row 1 of FactBand. FactBand is drawn by FactPage (FactChrome.tsx), which `renderFact` in FactView.tsx calls. RightPanel.renderFact no longer exists: Browse (RightPanel) and the chat's side panel both mount FactView. ChatFactChip also renders FactMetaLine, with `filterable={false}`.
- The PATH is no longer on the meta line. It is row 2 of the band (FactBandPath), which the title replaces once the title scrolls away. The meta line is type · origin · conf · N sources · [mount · branch], built as a list with `·` interleaved.
- The load-bearing details still hold: "1 source" / "N sources", the mount drawn as a hue dot plus the plain mono name at #b9c1cd, and the branch kept beside the mount.
- `StatBox` was NOT deleted. It is still exported from FactBody.tsx, but nothing imports it, and the header uses none. LensMeta is gone.
