# Space Stepping Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On a pull request's Files changed page, Space marks the file being read as viewed and scrolls to the next unviewed file; Shift+Space takes the last step back; one popup setting turns both off.

**Architecture:** A pure planner decides what a key press does from a snapshot of the files (order, viewed state, position); a page adapter, kept with the other GitHub page assumptions, takes that snapshot and performs the actions through GitHub's own controls; a keyboard listener in the capture phase wires them together and keeps a history of steps.

**Tech Stack:** TypeScript, WXT storage, Vitest + jsdom, Preact (popup).

**Spec:** `docs/superpowers/specs/2026-11-16-row-level-review-design.md` (§6.7, §7, §8 step 4)

## Global Constraints

- Work on a branch off `fork`; the pull request targets `fork`. Conventional Commits, English.
- No code, text or markup is taken from the "Spacebar Review for GitHub" add-on (GPL-3.0); behaviour comes from the spec only.
- Fixtures are hand-written from a public pull request; nothing from a private one.
- A file is never marked viewed while off screen; marking goes through GitHub's Viewed control only.
- Neither key acts while focus is in a text field, on a button or a link, in a comment form, or with Ctrl, Alt or Meta held.
- On by default; strings in English and Japanese.

## Review Focus

1. Space held down (key repeat) marking a run of files in a second: repeats are ignored (Task 3 test `ignores key repeat`).
2. Space pressed while GitHub's own "Submit review" dialog or a GitHub menu has focus on a non-field element: the dialog keeps the key (Task 3 test `leaves the key alone inside a dialog`).
3. Shift+Space after the reader unmarked the file by hand: the file stays unmarked, only the window moves back (Task 4 test).
4. The next unviewed file not yet in the page (GitHub adds file boxes as one scrolls): scroll target comes from the full file list (Task 2 and Task 1 evidence).
5. The page left by SPA navigation to another pull request: history and listener are dropped (Task 5 test `stops with the session`).

---

### Task 1: Evidence from a public pull request (owner)

Nothing automated runs against github.com logged in, so the owner captures, on any public pull request with three or more files, logged in, in Firefox devtools:

- [ ] **Step 1:** `outerHTML` of one file header with the Viewed control unchecked, then checked.
- [ ] **Step 2:** `outerHTML` of the "Submit review" button.
- [ ] **Step 3:** In the console on `/pull/N/changes`, the key names only of one entry of the route's file list, and the name of any viewed field:
  `fetch(location.pathname, {headers:{Accept:"application/json","X-Requested-With":"XMLHttpRequest","GitHub-Verified-Fetch":"true"}}).then(r=>r.json()).then(j=>console.log(Object.keys(j.payload.pullRequestsChangesRoute.diffSummaries[0])))`
- [ ] **Step 4:** How the file tree links to a file not yet in the page (the `href` of one tree entry).
- [ ] **Step 5:** Write `apps/extension/src/stepping/fixture.ts` from it: `appendSteppableFile(document, path, { viewed: boolean, top: number, height: number }): Promise<HTMLElement>` built on `inline/fixture.ts`'s `appendFileBlock`, plus `appendSubmitReviewButton(document)`. Record the selectors in the file's header comment with the capture date.
- [ ] **Step 6: Commit** `test(extension): fixture of the Viewed control and the review button`

### Task 2: Viewed state from the route data

**Files:** Modify `apps/extension/src/github/route-data.ts`, `route-data.test.ts`.

**Interfaces:**
- Produces: `RouteData.fileStates: readonly { readonly path: string; readonly isViewed: boolean }[]`, every changed file in route order, Markdown or not.

- [ ] **Step 1: Write the failing test:** a route payload whose `diffSummaries` hold three files, the second viewed (field name from Task 1 Step 3), gives `fileStates` `[a false, b true, c false]` in that order; a payload without the field gives all `false`.
- [ ] **Step 2–4:** Run → FAIL; implement in the existing parser; run → PASS.
- [ ] **Step 5: Commit** `feat(extension): read every file's viewed state from the route`

### Task 3: Step planner (pure)

**Files:** Create `apps/extension/src/stepping/planner.ts`, `planner.test.ts`.

**Interfaces:**
- Produces:
  ```ts
  export interface FileSnapshot { readonly path: string; readonly isViewed: boolean; readonly box: { readonly top: number; readonly bottom: number } | null }
  export type StepAction =
    | { readonly kind: "markAndGo"; readonly mark: string; readonly goTo: string | null }
    | { readonly kind: "goTo"; readonly goTo: string }
    | { readonly kind: "submitReview" }
    | { readonly kind: "none" };
  export function planStep(files: readonly FileSnapshot[]): StepAction;
  export function isSteppingKey(event: KeyboardEvent): "step" | "back" | null;
  ```
  `box` is relative to the window's top edge, `null` when the file is not in the page.

- [ ] **Step 1: Write the failing tests** (`planStep`):
  - `marks the file holding the top edge and goes to the next unviewed below`: boxes a[-500,-10] b[-10,800] c[800,1600], all unviewed → `markAndGo` mark `b` goTo `c`.
  - `uses the first file below the edge when none holds it`: a[-500,-100], gap, b[50,400] → mark `b`.
  - `skips viewed files when going on`: b current, c viewed, d unviewed → goTo `d`; d with `box: null` still chosen.
  - `marks and stays when nothing unviewed is below but one is above`: → `markAndGo` with `goTo` the first unviewed above.
  - `goes to the first unviewed above without marking when the current file is viewed and none is below`: → `goTo`.
  - `opens submit review when every file is viewed` → `submitReview`.
  - `never marks a file that is off screen`: current file's box bottom ≤ 0 is never chosen as `mark`.
  Tests for `isSteppingKey`: `" "` → `step`; Shift+`" "` → `back`; with Ctrl, Alt or Meta → `null`; `repeat: true` → `null` (`ignores key repeat`); target (via `composedPath()[0]`) an `input`, `textarea`, `select`, `[contenteditable]`, `button`, `a[href]`, `summary`, or inside `[role="dialog"]` / `[role="menu"]` → `null` (`leaves the key alone inside a dialog`); target a `div` inside a shadow root → `step`.
- [ ] **Step 2–4:** Run → FAIL; implement; run → PASS.
- [ ] **Step 5: Commit** `feat(extension): plan a Space step from the files on the page`

### Task 4: Page adapter and step history

**Files:** Create `apps/extension/src/stepping/page.ts`, `page.test.ts`, `history.ts`, `history.test.ts`.

**Interfaces:**
- Consumes: `FileSnapshot`, `StepAction` (Task 3); `RouteData.fileStates` (Task 2); `fileContainerId` (`inline/file-anchor.ts`); fixture (Task 1).
- Produces:
  ```ts
  export interface SteppingPage {
    snapshot(): Promise<readonly FileSnapshot[]>;   // route order; DOM viewed state overrides the route's for files in the page
    setViewed(path: string, isViewed: boolean): Promise<boolean>; // clicks GitHub's control; false if the file is not in the page
    isViewed(path: string): Promise<boolean | null>;
    goTo(path: string): Promise<void>;              // scrolls the box into view, or follows the file tree link when absent
    openSubmitReview(): boolean;
    scrollY(): number; scrollTo(y: number): void;
  }
  export function createSteppingPage(document: Document, fileStates: () => readonly { path: string; isViewed: boolean }[]): SteppingPage;
  export interface StepRecord { readonly scrollY: number; readonly marked: string | null }
  export function createStepHistory(): { push(r: StepRecord): void; pop(): StepRecord | null; clear(): void };
  ```
- [ ] **Step 1: Write the failing tests:** `setViewed` toggles the fixture's control and `isViewed` reads it back; `snapshot` reports a file absent from the page with `box: null` and its route state; history pops in reverse order and returns `null` when empty.
- [ ] **Step 2–4:** Run → FAIL; implement; run → PASS.
- [ ] **Step 5: Commit** `feat(extension): act on GitHub's Viewed control and remember steps`

### Task 5: Keys, setting, popup

**Files:**
- Create `apps/extension/src/stepping/space-stepping.ts`, `space-stepping.test.ts`, `src/settings/stepping-setting.ts`
- Modify `entrypoints/github.content/index.tsx`, `src/settings/SettingsPopup.tsx` (+ test), `src/settings/messages.ts`, `entrypoints/popup/main.tsx`

**Interfaces:**
- Consumes: Tasks 3 and 4.
- Produces: `startSpaceStepping(document: Document, page: SteppingPage, isEnabled: () => boolean): () => void`; storage item `sync:spaceStepping` (boolean, default `true`) exposed as `steppingSlot` with `getValue/setValue/watch` like `languageSlot`; `PopupMessages.spaceStepping: string` (en "Space steps through files, Shift+Space steps back", ja "Space でファイルを順に確認、Shift+Space で一つ戻る").

- [ ] **Step 1: Write the failing tests** (`space-stepping.test.ts`, on the fixture):
  - Space marks the current file and scrolls to the next; `preventDefault` was called.
  - Shift+Space after it: the window is back at the recorded `scrollY` and the file is unmarked.
  - Shift+Space after the reader unmarked the file by hand: window back, file still unmarked, no click on the control.
  - Shift+Space with an empty history: `preventDefault` not called (the browser scrolls up).
  - Space with focus on a `div` inside a shadow host built by `createShadowHost` still steps (the listener is on `document` in the capture phase, before the host stops the event).
  - Disabled setting: neither key is handled.
  - `stops with the session`: after the returned stop function, keys are not handled and the history is cleared.
  Popup test: the checkbox reflects the stored value and writes `false` when unticked.
- [ ] **Step 2–4:** Run → FAIL; implement; wire `startSpaceStepping` in `index.tsx` next to `startReview`, started and stopped by `followPullRequestPages` for every pull request (Markdown or not); run → PASS; `pnpm test`, `pnpm typecheck`, `pnpm lint` clean.
- [ ] **Step 5:** Manual check by the owner on a public pull request in Firefox (spec §7 checklist lines for Space and Shift+Space).
- [ ] **Step 6: Commit** `feat(extension): step through files with Space and back with Shift+Space`
