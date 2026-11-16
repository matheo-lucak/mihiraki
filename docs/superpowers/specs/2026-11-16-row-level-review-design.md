# Row-level table review, Firefox and file stepping — design

Date: 2026-11-16. Status: approved by the owner on 2026-11-16, with two changes
folded in below: GitHub's theme is kept (§6.5) and Shift+Space takes a Space
step back (§6.7). Base: upstream `KinjiKawaguchi/mihiraki` 0.4.0 (`beac356`).

## 1. Intent

The owner reviews pull requests whose changes are Markdown specifications: long
documents made of prose and of tables, some of them wide, with cells that hold
whole paragraphs. GitHub's rich diff shows a changed table as the whole old
table struck through followed by the whole new one, and offers no way to
comment from the rendered view. The source diff shows a changed table row as
one very long line.

Mihiraki already replaces the rich diff with a rendered before / after view and
posts review comments from it through the logged-in GitHub session. This fork
adds what the owner's reviews still lack:

1. tables compared row by row, with removed and added rows that stay easy to read;
2. a comment thread shown at the row it is about;
3. a Firefox build, since Firefox is the owner's browser;
4. the Space key to step through the files of a pull request, and Shift+Space
   to step back.

The view keeps GitHub's theme: every colour is GitHub's own, as in upstream.

Success is the owner reviewing a real specification pull request in Firefox
from the rendered view alone, tables included, without going back to the source
diff to understand what changed in a row.

## 2. What upstream does today

Read in the source at `beac356`; names are given so a builder can find them.

- `packages/core`: `parseBlocks` (`markdown/blocks.ts`) cuts a document into
  top-level blocks. A table is one block, a list is one block. The renderer
  puts `data-line-start` / `data-line-end` on every nested block element, table
  rows included. `alignBlocks` (`diff/align.ts`) pairs blocks: equal blocks
  anchor, changed regions are paired by similarity (`PAIRING_THRESHOLD = 0.5`,
  `blockSimilarity`), blocks of different kinds never pair. A modified pair is
  word-diffed as HTML by `diffBlockHtml`, or merged into one HTML by
  `mergeBlockHtml` for the single-column layout.
- `packages/ui`: `SplitReview` draws one grid row per aligned block, base left,
  head right; `UnifiedReview` draws one column (`unifiedCell`). The layout
  follows GitHub's own split / unified setting (`DiffLayout`). `RenderedBlock`
  shows a "+" on the innermost hovered element that has source lines, and press
  and drag extends the selection. The comment form is inserted under that
  element, except inside a table, where it goes after the whole table
  (`insertSlotBelow`, `split-view/rendered-dom.ts`). Threads are placed per
  aligned block (`placeThreads` in `core/document/split-document.ts`).
- `apps/extension` (WXT, Manifest V3, Chrome only): a content script mounts the
  view inside each Markdown file box of `/pull/N/changes` (or `/files`), a
  second content script in the page's own world (`world: "MAIN"`) keeps
  GitHub's page state in step, `github-client.ts` calls GitHub's internal page
  endpoints with the session cookies. Each mount is a shadow root; keyboard
  events are stopped at the shadow host so GitHub's shortcuts do not fire while
  typing (`shadow-host.ts`). Colours are `--mhr-*` tokens mapped to GitHub's
  Primer variables (`packages/ui/src/styles.css`). The popup holds one setting,
  the display language (`sync:language`).

Consequences for tables today: a single row can already be commented, since
each row carries its lines. But a changed table is drawn as two independent
tables whose rows do not line up, an inserted row has no gap opposite, nothing
tints a row or a cell, and both the form and the thread of a row appear under
the whole table.

A second consequence, from how blocks are paired: two versions of a table pair
only when the word overlap of their whole source reaches 0.5. A table that
grew a lot or was heavily reworded is not paired at all and shows as one table
removed and one table added.

## 3. Goals, non-goals, anti-goals

Goals

- G1. A changed table is compared row by row and cell by cell, in both layouts
  described in §6.2.
- G2. A removed row and an added row are tinted and never struck through; a
  changed cell is tinted, with the changed words marked inside it.
- G3. A comment form and a thread on a table row sit directly under that row.
- G4. Every colour comes from GitHub's theme, as in upstream, so the view
  follows GitHub's light, dark, colour-blind and high-contrast themes (§6.5).
- G5. The extension builds for Firefox, passes `web-ext lint` without errors,
  and can be signed for the owner's own use.
- G6. Space marks the file being read as viewed and moves to the next one;
  Shift+Space takes that step back (§6.7).
- G7. Each feature is a self-contained set of commits with tests, so it can be
  offered upstream later without rework.

Non-goals

- Detecting moved rows or moved columns: a moved row shows as removed and added.
- Pairing list items the way table rows are paired. Lists stay one block.
- Folding unchanged blocks of the document. Upstream shows the whole document
  and that stays. (Unchanged table rows are folded, §6.2.)
- Changing which GitHub endpoints are called or the review session logic. One
  exception: if Firefox refuses a request the way the extension sends it today,
  the way it is sent may change (§6.6); what is sent may not.
- A look of the fork's own. The owner decided to keep GitHub's theme (§6.5).
- A Chrome Web Store or public Firefox Add-ons listing.
- Review of files that are not Markdown.

Anti-goals — these would make the work a failure even if it runs

- Any content of a private pull request, any company or repository name, in
  this public fork: fixtures, screenshots, tests, documents, commit messages.
- Any request to a host other than github.com, any telemetry.
- Struck-through rows or blocks.
- A fork that can no longer take upstream's changes: edits to upstream files
  stay small and local, new behaviour lives in new files where it can.

## 4. Constraints

- github.com's Content Security Policy refuses bundled web fonts: system font
  stacks only. Styles go in a `<style>` element inside the shadow root, as
  upstream does.
- Comments go through GitHub's internal page endpoints with the session. They
  accept any line of a changed file, where the public API refuses lines outside
  a diff hunk; they can also change without notice. Nothing in this design adds
  a new endpoint.
- Release Firefox only installs signed extensions. Signing for own use is
  `web-ext sign --channel unlisted`, which uploads the build to Mozilla and
  needs the owner's AMO API credentials. A Firefox version number is dotted
  integers only.
- The page hooks upstream depends on (`div#diff-<sha256(path)>`,
  `[data-diff-header-wrapper]`, `[data-component="SegmentedControl"]`,
  `data-color-mode`, the names of GitHub's page stores) are GitHub's and move
  from time to time. Upstream keeps them in a few files of `apps/extension/src`
  (`inline/github-file-dom.ts`, `inline/file-anchor.ts`,
  `host-sync/github-stores.ts`, `github/viewscreen.ts`) and tests them on a
  small hand-written file block (`inline/fixture.ts`), not on a saved page. A
  new page assumption goes next to those, with a unit test on a hand-written
  fixture of the same kind.
- Nothing automated runs against github.com. Checks on the live site, logged
  in, are the owner's, from the checklist of §7.
- Upstream conventions apply: Conventional Commits, biome, vitest, Playwright
  in `apps/playground`. Commit messages and documents of the fork are in
  English.
- `main` of the fork mirrors upstream and receives no commit of its own.

## 5. Approaches

For the table comparison, three ways were weighed.

A. Row pairs in the core, drawn by a table component (chosen). The core turns a
   pair of tables into a table diff: matched columns, paired rows, a state per
   cell. The UI draws that model in one of two layouts. The model is pure and
   testable without a browser, both layouts share it, and thread placement by
   row falls out of it.

B. Keep two rendered tables and level them in the browser: measure row heights,
   insert spacer rows, re-measure on resize. No change to the core, but the
   result depends on measuring a live layout, breaks when a thread opens or a
   font loads, and gives no cell states to tint.

C. One merged table only. Simple, and the best reading of a wide table, but it
   drops the side-by-side comparison for the many small tables where it reads
   best.

A is chosen; it contains C as one of its two layouts.

For Firefox: build the same code with WXT's Firefox target and fix what differs
(chosen), against maintaining a separate Firefox port (no reason found for one:
the unchanged code builds and lints clean once the manifest carries an id).

For Space stepping: build it into this extension from a description of the
behaviour (chosen), against installing a separate small add-on that does it.
One extension holds the logged-in session instead of two, and the feature is
small. No code is taken from that add-on, whose licence (GPL-3.0) differs from
this project's (MIT).

## 6. Design

### 6.1 Table diff model (core)

A pure function takes the base table and the head table and returns a table
diff. Either side may be absent (a table added or removed as a whole).

- Structure. A table is a header row and body rows; a row has its source line
  range and its cells; a cell has its rendered HTML and its text. GFM tables
  always have a header row.
- Columns are matched by header text, compared without case, in order. A
  header renamed in place (same position between matched neighbours) stays one
  column. A column present on one side only is a removed or an added column.
- Rows are paired on the text of their matched columns: identical rows anchor,
  rows between anchors pair by similarity as blocks do, the rest are removed or
  added. One more rule for specification tables, which usually carry an
  identifier in their first column: two rows whose first-column text is equal,
  not empty, and unique in both tables are the same row, whatever else changed.
- Cell states, per matched column of a paired row: same; changed (word diff,
  using upstream's HTML-aware `diffBlockHtml` / `mergeBlockHtml` on the cell);
  replaced. Replaced is needed because cells are paired by column and not by
  similarity: below a similarity threshold the word diff is noise, so both
  texts are shown whole, without word marks.
- Row states: unchanged, modified (any cell not same, or a non-empty cell in a
  removed or added column), removed, added.

Which tables form a pair. Upstream's block rule alone would leave a much-edited
table unpaired (§2). Inside a changed region, the similarity of two tables is
therefore the larger of upstream's block similarity and a table measure: when
at least half of their header cells are the same, 0.5 plus half the share of
first-column values the two tables have in common. Two versions of a table then
pair however much their rows changed, and where a document repeats one header
over several tables, the tables that share row identifiers pair first; with no
identifier in common they pair in document order. This is one kind-specific
hook in upstream's alignment (`diff/align.ts`); nothing changes for other
blocks.

The aligned-row model of a document gains a table case, so that a modified
table pair carries its table diff instead of a single word-diffed HTML. A table
that did not change, and a table on one side only inside an otherwise paired
document, go through the same model.

### 6.2 Two layouts for a table pair (ui)

Side by side, rows level:

```
BEFORE                                 | AFTER
Option | Effect           | Default    | Option | Effect               | Default
--warn | warn above limit | 250        | --warn | warn above the limit | 250       changed cell tinted, words marked
--old  | legacy switch    | off        | ///////////////////////////////////        removed row tinted, gap opposite
/////////////////////////////////      | --crit | critical above limit | 500       added row tinted, gap opposite
```

- The old row and the new row of a pair are on the same line and have the same
  height, whatever their content. A removed row has a gap of its height on the
  other side, and the reverse.
- Each side keeps its own columns, sized by its own content. A column's floor
  is its longest word, capped at 16 characters; at least 16 characters when its
  longest cell exceeds 30 characters, at least 26 when it exceeds 60. Above the
  floors, width is shared in proportion to the longest cell of each column. A
  side that still overflows scrolls sideways on its own, and the other side
  follows the scroll.
- Recommended technique: one CSS grid per table pair whose row tracks are shared
  by both sides through `subgrid`, each side owning its column tracks. It needs
  no measuring. The mock-up (§9) is built this way.

Merged, full width:

```
Option | Effect                                  | Default
--warn | warn above [limit → the limit]          | 250       one row; removed and added words in place
--old  | legacy switch                           | off       removed row, tinted
--crit | critical above limit                    | 500       added row, tinted
```

- One table across the whole view, each row once. A changed cell shows removed
  and added words in place; a replaced cell shows the old text then the new
  text as two tinted blocks; removed and added columns are tinted.

Which layout a table pair opens in:

- GitHub's layout is unified: merged.
- GitHub's layout is split: side by side when, on both sides, the column
  floors above fit in the width of the side; merged otherwise. The choice is
  made again when the view's width changes.
- A switch at the top right of each table pair, a text button naming the other
  layout, lets the reader change it; a choice made by hand holds for that table
  until the page is left.

On the pull request used for the mock-up, at a common laptop width, 22 of 49
changed tables opened side by side and 27 merged: wide tables with paragraph
cells are the common case in specifications, which is why the merged layout is
not optional.

Unchanged rows of a modified table: in both layouts, after keeping one row of
context on each side of a change, a run of three or more unchanged rows is
folded to one line ("12 unchanged rows") that opens on a click. A row that
carries a thread is never folded. A table that did not change at all is shown
whole, as every unchanged block is.

Table semantics stay available to assistive technology (real table elements or
the matching ARIA roles).

### 6.3 Comments and threads on rows

- What can be commented does not change: any rendered element that has source
  lines, a table row among them; press and drag still selects a range.
- The form and the threads of a row appear directly under that row. Both are
  new work: today the form goes after the table and the thread under the block
  (§2). Side by side, they sit on the side they belong to and the other side
  gets a spacer of the same height, so the rows below stay level. Merged, they
  span the table.
- Sides in the merged layout follow upstream's single-column rule: a removed
  row is commented on the base side, every other row on the head side.
- Thread placement moves from "the block that contains the line" to "the row
  that contains the line" for tables. Threads that cannot be placed keep
  upstream's handling.

### 6.4 Marks

All in GitHub's diff colours (§6.5).

- Removed row or block: the deletion tint and a deletion edge. Added: the
  addition tint and an addition edge. Text stays upright, full contrast, not
  struck.
- Changed cell: the attention tint; removed words marked in the deletion word
  colour on the base side, added words in the addition word colour on the head
  side, both in place in the merged layout.
- Replaced cell or block: no word marks.
- The gap opposite a removed or added row or block: upstream's hatch for an
  empty cell.
- In the merged layout colour alone tells a removed word from an added one. If
  that proves hard to read, underline the added words; do not strike the
  removed ones.

### 6.5 Colours: GitHub's theme

Decided by the owner: the view keeps GitHub's theme. The fork has no stylesheet
of its own, no palette and no setting for the look.

- Upstream already takes every colour from GitHub's Primer variables through
  its `--mhr-*` tokens (`packages/ui/src/styles.css`), each with a literal
  fallback for light and for dark. The variables are inherited from the page,
  so the view follows whichever GitHub theme is active: light, dark, dimmed,
  colour-blind, high contrast. That stays.
- The table marks reuse the tokens upstream uses for blocks:

| Mark | Token | GitHub variable |
| --- | --- | --- |
| Removed row, tint | `--mhr-removed-line` | `--diffBlob-deletionLine-bgColor` |
| Removed words, removed edge | `--mhr-removed-word` | `--diffBlob-deletionWord-bgColor` |
| Added row, tint | `--mhr-added-line` | `--diffBlob-additionLine-bgColor` |
| Added words, added edge | `--mhr-added-word` | `--diffBlob-additionWord-bgColor` |
| Gap opposite a row | the hatch of `.mhr-cell--empty` | `--bgColor-muted` |
| Table grid | `--mhr-border` | `--borderColor-default` |
| Table header background | `--mhr-bg-muted` | `--bgColor-muted` |
| Changed cell, tint | a new token | `--bgColor-attention-muted` |

- One token is added, for the changed cell. Upstream maps the same GitHub
  variable to `--mhr-selection`, the highlight of the range being commented: a
  changed cell inside such a range must still read as selected.
- An added token follows upstream's pattern: a GitHub variable with a literal
  fallback, in the light block and in the dark block.
- Typography, spacing and table proportions stay those of upstream and of the
  GitHub Markdown style it bundles. System font stacks only.
- One deliberate difference from upstream's marks: upstream strikes removed
  words through (`del.mhr-del`); here they keep the deletion word colour and
  lose the strike (§6.4).

### 6.6 Firefox

- WXT builds the Firefox Manifest V3 target from the same sources. The manifest
  gains, for Firefox only, `browser_specific_settings.gecko` with an `id` of
  the fork's own, `strict_min_version` 140.0 and
  `data_collection_permissions: { required: ["none"] }`.
- A trial of the unchanged extension with only that manifest change built, and
  `web-ext lint` went from one error (missing id) to none; four warnings remain
  about `innerHTML` in bundled code, which do not block unlisted signing.
- Manifest V3 is the only Firefox target; no Manifest V2 build is kept.
- Not yet known, and the first thing to settle: whether the session requests
  and the page-world script behave in Firefox as in Chrome. The owner loads the
  trial build on a real pull request and reports what happens. Whatever it
  shows is fixed first, before any new feature, so that Firefox reaches parity
  with upstream's Chrome behaviour. If Firefox refuses the session requests as
  sent from the content script, they are sent through the page-world script
  instead; the endpoints and payloads stay the same.
- Scripts: build, zip and sign for Firefox next to the existing Chrome ones.
  The signed build is installed by the owner; publishing it anywhere is out of
  scope.
- The fork's builds carry upstream's version with a fourth number (0.4.0.1,
  0.4.0.2, …), since each signed build needs its own version, and a name that
  cannot be mistaken for upstream's build.
- Playwright in `apps/playground` gains a Firefox project so that the layout
  tests of §7 run in both engines.

### 6.7 Stepping through files with Space

- On the Files changed page, Space acts on the file being read: the file whose
  box holds the top edge of the window, or the first one below that edge. If
  that file is not yet viewed, Space marks it viewed, through GitHub's own
  Viewed control so that GitHub records it, and scrolls to the next file not
  yet viewed below it. A file is never marked while it is off screen.
- When no unviewed file is left below, Space scrolls to the first unviewed file
  above without marking it. When every file is viewed, Space opens GitHub's
  "Submit review".
- Shift+Space takes the last Space step back: the window returns to where it
  was before that step, and if the step marked a file viewed, the file is
  un-marked through GitHub's Viewed control. Steps are remembered in order for
  as long as the page stays open, so pressing Shift+Space again takes back the
  step before.
- A file whose viewed state was changed by hand after the step is left as it
  is; Shift+Space then only moves the window back.
- With no step to take back, Shift+Space is left to the browser, which scrolls
  up one screen. A Space that opened "Submit review" is not a step.
- GitHub adds file boxes to the page as the reader scrolls, so the next
  unviewed file may not be in the page yet; the list of files and their viewed
  state has to come from what GitHub holds for the whole pull request, not only
  from the boxes present.
- It applies to every file of the pull request, Markdown or not.
- Neither key does anything of the kind while the focus is in a text field,
  on a button or a link, or in one of the extension's comment forms, or when
  Ctrl, Alt or the Meta key is held.
- The extension stops keyboard events whose target is inside one of its shadow
  roots. A key event goes to the focused element, so the case to cover is the
  focus resting inside the rendered view on something that is not a field, a
  button or a link: Space must still step, and Shift+Space step back.
- One setting in the popup turns both keys off; they are on by default in the
  fork.
- The two page controls it relies on (the Viewed control, the review button)
  are not in upstream's fixture. Their markup is read once on live GitHub, on a
  public pull request, and a hand-written fixture is made from it.

### 6.8 How the fork is run

- Branches: `main` mirrors upstream; `fork` is the integration branch and
  holds everything of the fork, these documents included. Work is done on
  branches off `fork` and merged into it by pull request inside the fork.
- Each feature keeps its core change and its UI change in separate commits,
  so that a feature can be cherry-picked onto `main` and offered upstream. Offering anything upstream is the owner's decision, each
  time.
- Taking upstream's changes: fast-forward `main`, then merge `main` into
  `fork`.
- Upstream's release workflows (release pull request on push to `main`,
  publish on tag) are not meant to run in the fork.

## 7. Testing

- Core, unit: column matching (same, renamed in place, added, removed), row
  pairing (anchors, similarity, the first-column rule and its uniqueness
  condition), cell states and the replaced threshold, a table on one side only,
  thread placement by row.
- UI, component: both layouts from one model, the layout choice and the switch,
  folded rows and the never-fold-a-thread rule, form and thread slots with
  their spacers.
- Browser, Playwright in the playground, Chromium and Firefox: paired rows have
  the same top and height; they still do after a thread opens under a row and
  after the width changes; a wide table opens merged.
- Extension: a written manual checklist run on a real pull request in Firefox
  and in Chrome before each signed build: the view appears, a comment on a row
  posts and shows under the row after reload, reply and resolve work, GitHub's
  counters follow, Space steps and records Viewed, Shift+Space takes the step
  back and GitHub shows the file as not viewed again, the marks read well in
  GitHub's light and dark themes.
- Extension, unit, on a hand-written fixture: which file is being read, what a
  Space step marks and where it goes, the order steps are taken back in, the
  cases where both keys are left alone.
- One synthetic specification, written for the purpose, is the shared fixture
  and the playground sample. It covers: a changed cell, a replaced cell, a
  removed row, an added row, an added column, a renamed header, a table too
  wide for one side, a cell holding a paragraph, a table added, a table
  removed.

## 8. Order of delivery

1. Firefox parity: manifest, scripts, whatever the trial shows. The owner can
   use upstream's features in Firefox from here.
2. Table diff model in the core, thread placement by row.
3. The two table layouts, the switch, folded rows, row and cell marks in
   GitHub's colours.
4. Space stepping, Shift+Space and their setting.
5. Signing, version scheme, manual checklist.

Each step ends with the tests of §7 that concern it passing, and with upstream's
own tests still passing.

## 9. Mock-up

A private mock-up, generated by a throwaway script from a real pull request and
shown to the owner on 2026-11-16, shows the behaviour §6.2 to §6.4 and §6.7
describe: both table layouts with their switch, the tints and marks, the folded
rows, a thread under a row, a working comment form that sends nothing, Space
stepping. Two things in it are not the target: its colours were a palette of
the author's, which the owner declined in favour of GitHub's theme (§6.5), and
its Space stepping has no step back. Its script is not part of the fork and its
content is not copied here; the rules a builder needs from it are written in
§6.2 and §6.4. The synthetic fixture of §7 is what reproduces these cases in
the repository.

## 10. Left to the builder

- How the table structure reaches the core: from the Markdown tokens at parse
  time, or from the block's rendered HTML.
- The exact similarity measure and thresholds for rows and for the replaced
  state. Upstream's measure (`blockSimilarity`) takes whole blocks and its
  threshold (`PAIRING_THRESHOLD`) is private to `align.ts`: a text-level
  function has to be drawn out of them, or written beside them.
- How "fits side by side" is decided: by measuring or by estimating from the
  content, as long as §6.2's rule holds and no layout flicker results.
- Real table elements with grid display, or a grid of `div`s with ARIA roles.
- Names of the new tokens, files and settings; the wording of new interface
  strings, in English and Japanese as upstream has both.
- Whether the word diff of a cell needs the short-island folding the mock-up
  uses (one mark per changed run, a one-word unchanged island between two
  changes absorbed), judged on the fixture.

## 11. Decisions

The owner's, on 2026-11-16:

- The design is approved.
- GitHub's theme is kept; the palette and the "Look" setting of the draft are
  dropped (§6.5).
- Shift+Space takes a Space step back (§6.7).

Calls made by the author, in the draft the owner approved:

1. Two table layouts with an automatic choice and a switch per table (§6.2),
   instead of side by side only.
2. Folding runs of three or more unchanged rows in a modified table (§6.2).
3. No strike-through anywhere, removed words included (§6.4): the one place
   where the marks differ from upstream's.
4. The first-column rule for pairing rows, and the header rule for pairing
   tables (§6.1).
5. Space acts on the file being read and never marks a file off screen; on by
   default; applies to every file (§6.7).
6. Space stepping is built in rather than installed as a separate add-on (§5).
7. Unlisted signing for own use, fork builds numbered with a fourth number and
   named apart from upstream's (§6.6).
8. One integration branch `fork`; features cherry-picked to `main` only if the
   owner decides to offer them upstream (§6.8).
9. Unchanged blocks of the document are not folded (§3).

Calls made by the author after the approval, for the owner to check:

10. What Shift+Space does exactly (§6.7): it moves the window back and
    un-marks the file; steps are remembered while the page stays open; with
    nothing to take back the key is left to the browser; one setting turns
    both keys off.
11. The changed cell uses GitHub's attention tint, the only colour added to
    upstream's set (§6.5).

## 12. Open with the owner

- The result of the Firefox trial (§6.6): does the view appear, does a comment
  post, do GitHub's counters follow.
- Agreement to sign through Mozilla with the owner's AMO credentials (§4), or
  another way to install.
