# Firefox Build Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build, lint, zip and sign the extension for Firefox (Manifest V3) from the same sources as the Chrome build, and run the playground's browser tests in Firefox too.

**Architecture:** WXT already builds per browser; the manifest becomes a function of the target browser and adds Firefox's `browser_specific_settings` there. Firefox builds carry a fourth version number and a name of their own. Signing is a manual script run by the owner.

**Tech Stack:** WXT 0.21, `web-ext`, Playwright, pnpm 12.9.1, Node 24.

**Spec:** `docs/superpowers/specs/2026-11-16-row-level-review-design.md` (§4, §6.6, §7, §8 step 1 and step 5)

## Global Constraints

- Work on a branch off `fork`; the pull request targets `fork`. Never commit to `main`, never force-push.
- Conventional Commits, English. Biome (`pnpm lint`) and `pnpm typecheck` stay clean.
- Manifest V3 is the only Firefox target; no Manifest V2 build is kept.
- Gecko settings, Firefox only: `id` `mihiraki@matheo-lucak.github.io`, `strict_min_version` `140.0`, `data_collection_permissions: { required: ["none"] }`.
- A Firefox version number is dotted integers only; fork builds are `0.4.0.1`, `0.4.0.2`, …
- No request to a host other than github.com, no telemetry.
- Nothing automated runs against github.com; live checks are the owner's.
- A new dependency whose install runs a script fails the install (`strictDepBuilds`); it is only added to `allowBuilds` in `pnpm-workspace.yaml` after reading what the script does.

## Review Focus

1. A Chrome build that changes because of the Firefox work: the Chrome manifest must stay byte-for-byte what it is today (Task 1 pins it).
2. A Firefox version that is not dotted integers, or that repeats a version already signed (Mozilla refuses the upload): Task 2 tests the version function.
3. Upstream's release workflows running in the fork when `fork` is merged: out of this plan, left to the owner (§6.8); Task 4 does not touch them.
4. `web-ext lint` errors hidden behind warnings: Task 3 fails the build on errors only, and lists the warnings it accepts.
5. A Playwright test that passes in Chromium and silently skips in Firefox: Task 4 asserts both projects ran.

---

### Task 1: Firefox manifest from the same config

**Files:**
- Modify: `apps/extension/wxt.config.ts`
- Create: `apps/extension/manifest-options.ts`
- Test: `apps/extension/manifest-options.test.ts`
- Modify: `apps/extension/vitest.config.ts` (include the root-level test)
- Modify: `apps/extension/public/_locales/en/messages.json`, `.../ja/messages.json` (key `extNameFork`)

**Interfaces:**
- Produces: `manifestFor(browser: string, forkBuild: number): Record<string, unknown>` — the manifest fields WXT receives; `GECKO_SETTINGS` (const).

- [ ] **Step 1: Write the failing tests** in `manifest-options.test.ts`:
  - `chrome manifest is unchanged`: `manifestFor("chrome", 1)` deep-equals the object `wxt.config.ts` passes today (`name: "__MSG_extName__"`, `description`, `default_locale: "en"`, `homepage_url`, `permissions: ["storage"]`) and has no `browser_specific_settings` and no `version`.
  - `firefox manifest carries gecko settings`: `manifestFor("firefox", 1).browser_specific_settings` equals `{ gecko: { id: "mihiraki@matheo-lucak.github.io", strict_min_version: "140.0", data_collection_permissions: { required: ["none"] } } }`.
  - `firefox build is named apart`: `manifestFor("firefox", 1).name === "__MSG_extNameFork__"`.
- [ ] **Step 2:** `pnpm --filter @mihiraki/extension exec vitest run manifest-options.test.ts` → FAIL (module not found).
- [ ] **Step 3:** Implement `manifestFor` in `manifest-options.ts`; `wxt.config.ts` uses `manifest: ({ browser }) => manifestFor(browser, forkBuild)`. Add `extNameFork` to both locale files: en `"Mihiraki (fork build)"`, ja `"Mihiraki（フォーク版）"`.
- [ ] **Step 4:** Tests pass; `pnpm --filter @mihiraki/extension build` then `diff` the new `.output/chrome-mv3/manifest.json` against one built from `fork` before the change → identical.
- [ ] **Step 5: Commit** `feat(extension): build a Firefox manifest with gecko settings`

### Task 2: Fork version number

**Files:**
- Create: `apps/extension/fork-build.json` (`{ "build": 1 }`)
- Modify: `apps/extension/manifest-options.ts`, `manifest-options.test.ts`

**Interfaces:**
- Consumes: `manifestFor` (Task 1).
- Produces: `forkVersion(upstreamVersion: string, build: number): string`; `manifestFor("firefox", build).version === forkVersion(pkg.version, build)`.

- [ ] **Step 1: Write the failing tests:** `forkVersion("0.4.0", 1) === "0.4.0.1"`; `forkVersion("0.4.0", 0)` throws; `forkVersion("0.4.0-beta", 1)` throws (not dotted integers); `forkVersion("0.4.0.2", 1)` throws (upstream already has four parts).
- [ ] **Step 2:** Run → FAIL.
- [ ] **Step 3:** Implement; `wxt.config.ts` reads `fork-build.json` and `package.json`'s version. First check that WXT honours a `version` set in the manifest config over `package.json` (WXT docs, "Manifest → version"); if it does not, set `version_name` is not enough for Firefox, so the fallback is a `build:hooks` `manifest:generated` hook that overwrites `manifest.version`.
- [ ] **Step 4:** Tests pass; `wxt build -b firefox --mv3` writes `"version": "0.4.0.1"` in `.output/firefox-mv3/manifest.json`; the Chrome manifest still has `0.4.0`.
- [ ] **Step 5: Commit** `feat(extension): number fork Firefox builds with a fourth part`

### Task 3: Firefox scripts and lint

**Files:**
- Modify: `apps/extension/package.json` (scripts, devDependency `web-ext`), root `package.json` (scripts)
- Create: `apps/extension/web-ext-config.mjs`

**Interfaces:**
- Produces scripts: extension `build:firefox` (`wxt build -b firefox --mv3`), `zip:firefox` (`wxt zip -b firefox --mv3`), `lint:firefox` (`web-ext lint --source-dir .output/firefox-mv3 --warnings-as-errors=false`), `sign:firefox` (`web-ext sign --channel unlisted --source-dir .output/firefox-mv3 --artifacts-dir .output/signed`); root `build:firefox`, `zip:firefox`, `lint:firefox`.

- [ ] **Step 1:** Add `web-ext` as a devDependency; `pnpm install`. If the install stops on a build script, read it and stop for the owner's decision before listing it in `allowBuilds`.
- [ ] **Step 2:** `pnpm build:firefox && pnpm lint:firefox` → exit 0, 0 errors. The accepted warnings are the four `UNSAFE_VAR_ASSIGNMENT` (`innerHTML` in bundled code); write them, by rule id and file, in a comment of `web-ext-config.mjs`.
- [ ] **Step 3:** `pnpm zip:firefox` → `.output/mihiraki-0.4.0.1-firefox.zip` and the sources zip exist.
- [ ] **Step 4:** `sign:firefox` reads `WEB_EXT_API_KEY` / `WEB_EXT_API_SECRET` from the environment; it is not run by the plan. Document in `docs/fork/firefox.md`: build, load as temporary add-on (`about:debugging#/runtime/this-firefox`), sign (bump `fork-build.json` first), install the signed `.xpi`.
- [ ] **Step 5: Commit** `build(extension): add Firefox build, lint, zip and sign scripts`

### Task 4: CI and Playwright in Firefox

**Files:**
- Modify: `.github/workflows/verify.yml` (build job, e2e job)
- Modify: `apps/playground/playwright.config.ts`

**Interfaces:**
- Produces: Playwright projects `chromium` (`devices["Desktop Chrome"]`) and `firefox` (`devices["Desktop Firefox"]`), both with today's `viewport` `{ width: 1400, height: 1000 }` and `locale: "ja-JP"`.

- [ ] **Step 1:** Replace the single `use` device by the two projects; keep `baseURL` and `webServer` as they are.
- [ ] **Step 2:** `pnpm --filter @mihiraki/playground exec playwright install firefox chromium-headless-shell`, then `pnpm test:e2e` → the existing `comment-pointer.spec.ts` passes in both projects; the list reporter shows each test twice, `[chromium]` and `[firefox]`.
- [ ] **Step 3:** `verify.yml`: the build job also runs `pnpm build:firefox` and `pnpm lint:firefox` and uploads `extension-firefox-mv3`; the e2e job installs `firefox` next to `chromium-headless-shell` (both in the cache key's install and `install-deps` lines).
- [ ] **Step 4:** Push the branch, open the pull request against `fork`; the `verify` checks pass on it.
- [ ] **Step 5: Commit** `ci: build, lint and run browser tests for Firefox`

### Task 5: Parity fixes from the owner's trial

Scope set by the owner's report on the trial build (spec §6.6, §12). Known candidate, only if the report shows comments failing to post or route data failing to load: send the requests through the page-world script. Then:

**Files:** `apps/extension/src/github/github-client.ts` (the `FetchFn` passed to `createGitHubBackend`), `src/host-sync/protocol.ts`, `bridge.ts`, `client.ts`.

**Interfaces:**
- Produces: `HOST_SYNC_EVENTS.fetchRequest` / `fetchResponse`; `createPageWorldFetch(client: HostSyncClient): FetchFn` — same signature as `FetchFn`, so `createGitHubBackend(pr, createPageWorldFetch(client))` is the only call-site change.

- [ ] **Step 1: Write the failing test** in `host-sync/host-sync.test.ts`: a request sent through `createPageWorldFetch` reaches the bridge's injected fetch with the same URL, method, headers and body, and its `Response` status and JSON come back unchanged; a URL not starting with `https://github.com/` is refused by the bridge.
- [ ] **Step 2–4:** Run → FAIL; implement; run → PASS, and `github-backend.test.ts` unchanged and passing.
- [ ] **Step 5: Commit** `fix(extension): send GitHub requests from the page in Firefox`

The owner then re-runs the trial; this task ends when the view appears, a comment posts and GitHub's counters follow.
