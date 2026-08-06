---
type: observation
domain: [web, ui, fact]
confidence: 0.95
sources: 1
entities: [FactMetaLine, FactBody, RightPanel.renderFact, LensMeta, StatBox, RepoRows, repoHueBg, displayLensPath, web/src/FactMetaLine.tsx, web/src/FactBody.tsx, web/src/RightPanel.tsx]
refs: ['src://7b4887ce51d9/web/src/FactMetaLine.tsx@aa4432b46e76bdc7fcfdc9c0d55b87bc03eefec2:b717cd91474a8928f41286858ad69b03dab69f5f', 'kb://3ec012f5b4d2/kb/decisions/ui/summary-panel/repos-ranked-rows-with-activity/fed2c140.md', 'https://github.com/knomit/knomit/pull/70']
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

**Structural consequence**: the chips and the StatBoxes moved OUT of FactBody into a new FactMetaLine rendered by RightPanel.renderFact. They were never body — they describe the fact, so they belong with the title and the controls. FactBody is now content only: prose, tag clouds, refs. `LensMeta` and `StatBox` were deleted with them.
