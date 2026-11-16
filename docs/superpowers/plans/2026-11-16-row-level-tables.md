# Row-Level Table Review Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Compare a changed Markdown table row by row and cell by cell, draw it side by side with level rows or as one merged table, and put a table row's comment form and threads directly under that row.

**Architecture:** The core keeps each table's structure from parse time, pairs two tables' columns and rows into a `TableDiff`, pairs tables across versions with a table-aware measure, and places threads by table row. The UI draws every table block through one new component, `TablePairView`, a CSS grid whose rows are shared by both sides through `subgrid`; it reuses the comment-selection callbacks `SplitReview` already hands to each cell. Colours are GitHub's, through upstream's `--mhr-*` tokens.

**Tech Stack:** TypeScript, markdown-it 15, `diff` (`diffArrays`), Preact, Vitest + jsdom + @testing-library/preact, Playwright.

**Spec:** `docs/superpowers/specs/2026-11-16-row-level-review-design.md` (§2, §3, §5 A, §6.1–§6.5, §7, §8 steps 2–3, §10)

## Global Constraints

- Work on branches off `fork`; pull requests target `fork`. Conventional Commits, English. Core change and UI change of a feature in separate commits (§6.8).
- Keep edits to upstream files small; new behaviour in new files (`packages/core/src/table/`, `packages/ui/src/table-view/`).
- No struck-through text anywhere: `del.mhr-del` loses `text-decoration: line-through`.
- Colours only from GitHub's variables through `--mhr-*` tokens, each with a light and a dark literal fallback; one token added: `--mhr-changed-cell` → `--bgColor-attention-muted` (light fallback `#fff8c5`, dark `rgba(187, 128, 9, 0.15)`).
- Values pinned by the spec: column floor = longest word capped at 16ch; ≥16ch when the longest cell exceeds 30 characters; ≥26ch when it exceeds 60; fold runs of ≥3 unchanged rows after keeping 1 row of context each side of a change; never fold a row with a thread; table pairing applies when ≥ half the header cells match; pairing threshold stays upstream's 0.5.
- The replaced-cell threshold starts at 0.35 (the mock-up's value) and is a named constant.
- Fixtures and the playground sample are synthetic; nothing from a private pull request.
- Interface strings in English and Japanese (`packages/ui/src/i18n/en.ts`, `ja.ts`).
- Upstream's tests keep passing: `pnpm test`, `pnpm typecheck`, `pnpm lint`, `pnpm test:e2e`.

## Review Focus

1. GFM cell edge cases: escaped pipes `\|`, inline code holding a pipe, empty cells, a body row with fewer or more cells than the header (GFM pads and truncates). Test in Task 1.
2. A first-column value that repeats inside one table (the identifier rule must not fire) and an empty first column. Test in Task 2.
3. A thread whose range starts above the table and ends inside it, or covers several rows: it goes under the row holding its last line. Test in Task 4.
4. Cells holding images, links or inline HTML: sanitised and images held exactly as in other blocks. Test in Task 7.
5. A table with a header and no body rows, on one side or both. Tests in Tasks 2 and 7.

---

### Task 1: Table structure at parse time (core)

**Files:**
- Create: `packages/core/src/table/table-source.ts`, `table-source.test.ts`
- Modify: `packages/core/src/markdown/types.ts` (optional field), `packages/core/src/markdown/blocks.ts` (fill it)

**Interfaces:**
- Produces:
  ```ts
  export interface TableCellSource { readonly html: string; readonly text: string }
  export interface TableRowSource { readonly lines: LineRange; readonly cells: readonly TableCellSource[] }
  export interface TableSource { readonly header: TableRowSource; readonly rows: readonly TableRowSource[] }
  export function readTable(tokens: readonly Token[], sourceLines: readonly string[], render: (inline: Token) => string): TableSource;
  // SourceBlock gains: readonly table?: TableSource   (set only when kind === "table")
  ```
  `text` is the cell's inline source, trimmed; `html` is the rendered inline content of the `th`/`td`.

- [ ] **Step 1: Write the failing tests** (`table-source.test.ts`, through `parseBlocks`):
  - three-column table: header cells' `text` `["Option","Effect","Default"]`; row 1 `lines` `{ start: 3, end: 3 }`; cell html of `` `a` `` contains `<code>a</code>`.
  - `\|` and `` `a|b` `` stay inside one cell; an empty cell has `text ""`.
  - a body row with fewer cells than the header gets empty cells up to the header's count; extra cells are dropped (GFM).
  - a header-only table has `rows: []`.
  - a front-matter block has no `table`.
- [ ] **Step 2:** `pnpm --filter @mihiraki/core exec vitest run src/table/table-source.test.ts` → FAIL.
- [ ] **Step 3:** Implement; `parseBlocks` passes the block's token slice and `(t) => md.renderer.renderInline(t.children ?? [], md.options, env)`.
- [ ] **Step 4:** Run → PASS; `pnpm --filter @mihiraki/core test` → all pass.
- [ ] **Step 5: Commit** `feat(core): keep a table's rows and cells from parse time`

### Task 2: Table diff model (core)

**Files:**
- Create: `packages/core/src/diff/pairing.ts` (moved out of `align.ts`), `packages/core/src/table/table-diff.ts`, `table-diff.test.ts`
- Modify: `packages/core/src/diff/align.ts` (use `pairBySimilarity`), `packages/core/src/diff/similarity.ts` (export `textSimilarity`)

**Interfaces:**
- Consumes: `TableSource` (Task 1); `diffBlockHtml`, `mergeBlockHtml`.
- Produces:
  ```ts
  // diff/pairing.ts — align.ts's weighted LCS, unchanged in behaviour, made generic
  export const PAIRING_THRESHOLD = 0.5;
  export function pairBySimilarity<T>(removed: readonly T[], added: readonly T[], similarity: (a: T, b: T) => number, threshold: number): { baseIndex: number; headIndex: number }[];
  // diff/similarity.ts — blockSimilarity becomes a kind check around this
  export function textSimilarity(a: string, b: string): number;
  // table/table-diff.ts
  export const REPLACED_THRESHOLD = 0.35;
  export type ColumnKind = "matched" | "removed" | "added";
  export interface TableColumn { readonly kind: ColumnKind; readonly base: number | null; readonly head: number | null }
  export type CellState = "same" | "changed" | "replaced" | "removed" | "added";
  export interface TableDiffCell { readonly state: CellState; readonly baseHtml: string | null; readonly headHtml: string | null; readonly mergedHtml: string | null }
  export interface TableDiffRow { readonly kind: RowKind; readonly base: TableRowSource | null; readonly head: TableRowSource | null; readonly cells: readonly TableDiffCell[] }
  export interface TableDiff { readonly columns: readonly TableColumn[]; readonly header: TableDiffRow; readonly rows: readonly TableDiffRow[] }
  export function diffTables(base: TableSource | null, head: TableSource | null): TableDiff;
  ```
  `cells` is index for index with `columns`. `changed`: `baseHtml`/`headHtml` from `diffBlockHtml`, `mergedHtml` from `mergeBlockHtml`. `replaced`: both plain, `mergedHtml: null`. `same`: both plain, `mergedHtml` the head html.

- [ ] **Step 1: Write the failing tests:**
  - `align.test.ts` unchanged and passing after the move (behaviour pin).
  - columns: same headers → all `matched`; a column added in the middle → `added` at its place; header compared without case; a header renamed in place between matched neighbours → one `matched` column.
  - rows: identical rows → `unchanged`; one cell edited → `modified`, that cell `changed` with `<del class="mhr-del">` in `baseHtml` and `<ins class="mhr-ins">` in `headHtml`; a row removed → `removed`, cells `removed`; a row added → `added`.
  - identifier rule: rows reordered with a unique first column `--warn`, every other cell rewritten → still paired as `modified`; a first column `x` occurring twice in one table → rule not applied (pairing by similarity); empty first column → rule not applied.
  - `replaced`: a cell whose `textSimilarity` < 0.35 → `replaced`, no marks, `mergedHtml: null`.
  - a row in a removed or added column with non-empty text makes the row `modified`.
  - `diffTables(t, null)` → every row `removed`; `diffTables(null, t)` → every row `added`; header-only on both sides → `rows: []`.
- [ ] **Step 2:** Run → FAIL.
- [ ] **Step 3:** Implement. Column matching: `diffArrays` on lower-cased trimmed header texts; inside each unmatched gap, equal counts on both sides pair by position (renamed in place), otherwise removed/added. Row anchoring: `diffArrays` on a key per row = `id:<first cell text>` when that text is non-empty and unique in both tables, else `row:<texts of matched columns joined by \u0000>`; changed regions paired by `pairBySimilarity(…, textSimilarity of the joined matched-column texts, PAIRING_THRESHOLD)`. An anchored row whose cells differ is `modified`.
- [ ] **Step 4:** Run → PASS; core suite passes.
- [ ] **Step 5: Commit** `feat(core): compare two tables by column and row`

### Task 3: Pair much-edited tables (core)

**Files:** Modify `packages/core/src/diff/align.ts`, `align.test.ts`; Create `packages/core/src/table/table-similarity.ts`, `table-similarity.test.ts`.

**Interfaces:**
- Consumes: `TableSource`, `pairBySimilarity`, `blockSimilarity`.
- Produces: `tableSimilarity(a: TableSource, b: TableSource): number` — 0 when fewer than half the header cells match (matched count / larger header count, compared without case); otherwise `0.5 + 0.5 * dice(first-column values)` where dice = `2·|common| / (|A| + |B|)` over non-empty first-cell texts. `align.ts` uses `max(blockSimilarity, tableSimilarity)` when both blocks are tables.

- [ ] **Step 1: Write the failing tests:**
  - `tableSimilarity`: same header, no common first-column value → `0.5`; same header, all first-column values common → `1`; one header cell of four in common → `0`.
  - `alignBlocks`: a table whose every body row was reworded (block similarity < 0.5) but whose header is kept → one `modified` row, not `removed` + `added`.
  - two removed and two added tables under one shared header, in the same order, the first pair sharing identifiers `a1…a5` and the second `b1…b5` → paired first-with-first, second-with-second (pairing stays order-preserving); the same four tables with no identifiers in common → paired in document order.
  - a paragraph next to a table never pairs with it (kinds differ) — unchanged behaviour.
- [ ] **Step 2–4:** Run → FAIL; implement; run → PASS.
- [ ] **Step 5: Commit** `feat(core): pair two versions of a table by header and identifiers`

### Task 4: Split rows carry the table diff; threads by table row (core)

**Files:** Modify `packages/core/src/document/split-document.ts` (+ test), `packages/core/src/index.ts`; Create `packages/core/src/table/table-threads.ts`, `table-threads.test.ts`.

**Interfaces:**
- Consumes: `diffTables` (Task 2).
- Produces:
  - every `SplitRow` variant gains `readonly table: TableDiff | null` — non-null when the row's block (either side) is a table; `buildSplitRows` fills it (`diffTables(base?.table ?? null, head?.table ?? null)`).
  - `placeTableThreads(table: TableDiff, threads: RowThreads): { readonly header: RowThreads; readonly byRow: readonly RowThreads[] }` — a thread goes to the row whose lines on `thread.side` contain `thread.lines.end`; any other thread of the block (lines on the header, the separator, or above the table) goes to `header`.
  - exports from `index.ts`: `diffTables`, `placeTableThreads`, types `TableDiff`, `TableDiffRow`, `TableDiffCell`, `TableColumn`, `CellState`, `TableSource`, `TableRowSource`, `TableCellSource`.

- [ ] **Step 1: Write the failing tests:**
  - `buildSplitRows` on a document with a paragraph and a changed table → the paragraph row has `table: null`, the table row a `TableDiff` with one `modified` row.
  - `placeTableThreads`: a head thread on line 5 (row 2's line) → `byRow[1].head`; a base thread on a removed row's line → that row's `base`; a thread on lines 2–5 (starting above) → `byRow[1]` (its last line); a thread on the header line → `header`.
- [ ] **Step 2–4:** Run → FAIL; implement; run → PASS; `pnpm typecheck` passes (UI compiles with the new field).
- [ ] **Step 5: Commit** `feat(core): attach the table diff to split rows and place threads by table row`

### Task 5: Layout choice and folding (ui, pure)

**Files:** Create `packages/ui/src/table-view/table-fit.ts`, `table-fit.test.ts`, `table-folding.ts`, `table-folding.test.ts`.

**Interfaces:**
- Consumes: `TableDiff`, `TableDiffRow` (Task 2).
- Produces:
  ```ts
  export function columnFloorCh(cellTexts: readonly string[]): number;
  export function fitsSideBySide(table: TableDiff, sideWidthPx: number, chPx: number, cellPaddingPx: number): boolean;
  export type FoldedItem = { readonly kind: "row"; readonly index: number } | { readonly kind: "fold"; readonly from: number; readonly count: number };
  export function foldRows(rows: readonly TableDiffRow[], keep: ReadonlySet<number>): readonly FoldedItem[];
  ```
  `keep` = indices of rows that carry a thread. `cellPaddingPx` is 26 (GitHub's `6px 13px` padding) + 1 border.

- [ ] **Step 1: Write the failing tests:**
  - `columnFloorCh(["a", "bb"])` → 2; a 20-letter word → 16 (cap); a cell of 31 characters of short words → 16; a cell of 61 characters → 26.
  - `fitsSideBySide`: two 3-column sides of short cells at 560 px → true; a side with three 61-character cells at 560 px (3·26ch·8px + padding > 560) → false; checks both sides.
  - `foldRows`: 10 unchanged rows, a modified row at 5 → rows 0–3 folded (count 4), 4–6 shown, 7–9 folded (count 3); a run of 2 unchanged rows beyond context is not folded; a kept row (thread) inside a run splits the fold; a table with no change at all → nothing folded (all rows).
- [ ] **Step 2–4:** Run → FAIL; implement; run → PASS.
- [ ] **Step 5: Commit** `feat(ui): choose a table's layout and fold its unchanged rows`

### Task 6: `TablePairView`, both layouts and the marks (ui)

**Files:** Create `packages/ui/src/table-view/TablePairView.tsx`, `TableCellHtml.tsx`, `TablePairView.test.tsx`; Modify `packages/ui/src/i18n/messages.ts`, `en.ts`, `ja.ts`, `packages/ui/src/styles.css`.

**Interfaces:**
- Consumes: `TableDiff` (Task 2), `fitsSideBySide`, `foldRows` (Task 5); `sanitizeHtml`, `holdImages`, `useImagePolicy`, `SafeHtml`.
- Produces:
  ```ts
  export interface TablePairViewProps {
    readonly table: TableDiff;
    readonly layout: "split" | "unified";       // GitHub's layout; unified forces merged
    readonly sideWidthPx?: number;              // injected in tests; measured with ResizeObserver otherwise
    readonly comment: Readonly<Record<Side, CommentCellProps>>;   // Task 7 uses it; rendered inert here
    readonly threads: { readonly header: RowThreads; readonly byRow: readonly RowThreads[] };
  }
  export function TablePairView(props: TablePairViewProps): JSX.Element;
  ```
  Messages added: `showMerged` (en "Show merged", ja "まとめて表示"), `showSideBySide` (en "Show side by side", ja "左右に並べて表示"), `unchangedRows(count)` (en "`${count} unchanged rows`", ja "`変更のない ${count} 行`").

  DOM (ARIA table roles on `div`s, one element per row per side so `data-line-start`/`data-line-end` sit on it):
  - side by side: `div.mhr-tg` (grid, columns `[base] minmax(0,1fr) [head] minmax(0,1fr)`, one row track per table row) > `div.mhr-tside[data-side][role=table]` (`grid-row: 1 / -1; grid-template-rows: subgrid`; own column tracks) > `div.mhr-trow.mhr-trow--{kind}[role=row]` (`display: contents` is not used: each row is a subgrid of its side's columns) > `div.mhr-tcell.mhr-tcell--{state}[role=cell|columnheader]`; a missing row on one side is `div.mhr-tgap`.
  - merged: `div.mhr-tmerged[role=table]` > rows once; `replaced` cell shows `div.mhr-was` then `div.mhr-now`.
  - switch: `button.mhr-tswitch` top right, text naming the other layout; a fold: `button.mhr-tfold` spanning the side, text `unchangedRows(n)`, expands on click.
  - sideways scroll of one side scrolls the other (`scrollLeft` mirrored on `scroll`).

- [ ] **Step 1: Write the failing tests** (jsdom, `I18nProvider locale="en"`):
  - a modified row renders level: same grid row index (`style.gridRow` or order) on both `.mhr-tside`; a removed row has `.mhr-tgap` opposite.
  - classes: removed row `mhr-trow--removed`, added `mhr-trow--added`, changed cell `mhr-tcell--changed` containing `del.mhr-del` on base, `ins.mhr-ins` on head; replaced cell `mhr-tcell--replaced` with no `ins`/`del`.
  - `sideWidthPx={560}` with three 61-character columns → merged; with short cells → side by side; `layout="unified"` → merged whatever the width.
  - the switch toggles the layout and its text; the choice survives a re-render with the same table.
  - fold: 10 unchanged + 1 modified → a button "4 unchanged rows" that, clicked, shows the 4 rows.
  - a cell with `<img src="https://example.com/a.png">` holds the image like other blocks (`data-mhr-src`).
  - header-only table renders its header and nothing else.
- [ ] **Step 2–4:** Run → FAIL; implement; styles: `.mhr-trow--removed` uses `--mhr-removed-line` + `inset 3px 0 0 var(--mhr-removed-word)`, `--added` the addition pair, `.mhr-tcell--changed` `--mhr-changed-cell` (new token, light and dark blocks), `.mhr-tgap` the hatch of `.mhr-cell--empty`, grid lines `--mhr-border`, header `--mhr-bg-muted`; `del.mhr-del` loses its strike; run → PASS.
- [ ] **Step 5: Commit** `feat(ui): draw a table pair side by side with level rows or merged`

### Task 7: Comments and threads under a table row (ui)

**Files:** Modify `packages/ui/src/table-view/TablePairView.tsx` (+ create `TablePairView.comments.test.tsx`).

**Interfaces:**
- Consumes: `CommentCellProps` (`split-view/SplitCellView.tsx`), `ThreadList`, `placeTableThreads` output (Task 4).
- Produces: in both layouts, under a row: a slot row `div.mhr-tslot` spanning its side's columns holding the form (when `formAfterLines` equals the row's lines) or the row's `ThreadList`; side by side, the other side gets an empty `div.mhr-tspacer` on the same grid row. Merged: the slot spans the table; a removed row's form and threads are on the base side, every other row's on the head side.

- [ ] **Step 1: Write the failing tests:**
  - hovering a head row shows `button.mhr-add` ("Add a comment"); `mouseDown` then `mouseUp` opens the form in a `.mhr-tslot` directly after that row on the head side, with a `.mhr-tspacer` on the same grid row on the base side.
  - dragging from row 2 to row 4 highlights rows 2–4 (`mhr-selected`) and opens the form under row 4.
  - a head thread on row 2's line renders under row 2, not after the table; a header thread renders above the first body row.
  - merged: a removed row's "+" calls `onSelectionStart` with `side: "base"`.
  - a row with a thread is never folded (render with a thread inside a would-be fold).
- [ ] **Step 2–4:** Run → FAIL; implement (row `onMouseOver` → `onPointerOverLines(row.lines)`; "+" behaviour as `AddCommentButton` in `RenderedBlock.tsx`: `mouseDown` → `onSelectionStart`, keyboard click → `onRequestComment`); run → PASS.
- [ ] **Step 5: Commit** `feat(ui): open the comment form and show threads under a table row`

### Task 8: Wire tables into both reviews (ui)

**Files:** Modify `packages/ui/src/split-view/SplitReview.tsx`, `UnifiedReview.tsx` (+ their tests).

**Interfaces:**
- Consumes: `SplitRow.table`, `placeTableThreads`, `TablePairView`.
- Produces: a row with `table !== null` renders `div.mhr-row.mhr-row--table` holding one `TablePairView` spanning both columns (split) or the single column (unified), with `comment={{ base: comment.cellProps("base", rowIndex), head: comment.cellProps("head", rowIndex) }}` and `threads={placeTableThreads(row.table, placement.byRow[rowIndex])}`. Every other row is rendered as today.

- [ ] **Step 1: Write the failing tests:** `SplitReview` with a changed table → one `.mhr-row--table` and no `.markdown-body table`; the paragraph rows unchanged (existing tests pass untouched); `UnifiedReview` with a changed table → merged layout.
- [ ] **Step 2–4:** Run → FAIL; implement; run → PASS; `pnpm test`, `pnpm typecheck`, `pnpm lint` clean.
- [ ] **Step 5: Commit** `feat(ui): review every table through the table pair view`

### Task 9: Synthetic sample and browser tests (playground)

**Files:** Modify `apps/playground/src/samples.ts`; Create `apps/playground/e2e/table-rows.spec.ts`.

**Interfaces:**
- Produces: sample file `docs/options-spec.md` (base and head) — synthetic, English — covering every case of spec §7: a changed cell, a replaced cell, a removed row, an added row, an added column, a renamed header, a table too wide for one side, a cell holding a paragraph, a table added, a table removed; and one sample thread on a table row.

- [ ] **Step 1: Write the failing spec** (`table-rows.spec.ts`, open `/` then the file `docs/options-spec.md`):
  - `paired rows share top and height`: for each `.mhr-trow` pair on the grid row of a modified row, base and head `boundingBox()` have equal `y` and `height` (±0.5px).
  - `rows stay level after a thread opens`: open the form under a head row; the next row's pair still has equal `y`.
  - `rows stay level after a width change`: `setViewportSize({ width: 1100, height: 1000 })`; re-check.
  - `a wide table opens merged`: the wide table has `.mhr-tmerged`.
  - `the sample thread shows under its row`.
- [ ] **Step 2:** `pnpm test:e2e` → FAIL (sample missing).
- [ ] **Step 3:** Write the sample.
- [ ] **Step 4:** `pnpm test:e2e` → PASS in every configured Playwright project (Chromium; Firefox too once the Firefox plan's Task 4 is merged).
- [ ] **Step 5: Commit** `test(playground): synthetic specification with tables and level-row checks`
