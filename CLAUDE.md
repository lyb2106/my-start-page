# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal browser start page (new tab replacement). UI is Korean (`lang="ko"`). Desktop-only — no mobile breakpoints, drag-and-drop assumes mouse.

## Architecture

**Single-file app + 3 static JSON.** Entirety is `index.html` (~60KB inline HTML/CSS/JS) plus `data/shortcuts.json`, `data/bookmarks.json`, and `data/templates.json` fetched at load. No build, no bundler, no framework.

**Deploy target: GitHub Pages.** Page must be served over HTTP/HTTPS for `fetch()` to work — `file://` double-click does NOT work (CORS blocks fetching the JSON files; the page silently renders as empty).

**To run locally:** `python -m http.server 8000` from the repo root, then `http://localhost:8000`. The OAuth Client's authorized origins must include the local origin (currently `http://localhost:8000`) and `https://lyb2106.github.io`.

## Two storage layers

| Layer | Storage | Lifetime | Edited by |
|---|---|---|---|
| Static | `data/*.json` in git | Permanent in repo | Manual edit + commit |
| Dynamic | Google Drive folder `.my-start-page` (regular, user-visible at Drive root) + localStorage cache | Per-user | Page UI |

**Static**: bookmark pills, shortcut grid, doc templates. To change, edit the JSON, commit, push. No UI for editing — the right-click edit modal was removed. `data/templates.json` defines the 5 docs templates (`product_dev`, `rnd_grant`, `biz_proposal`, `academic_paper`, `blank`); each has `{id, label, sections:[{q, hint}]}`. Hints are the gray pre-fill text shown in the editor.

**Dynamic**: Daily Job, Brain Dump, Time Plan, Project Chart, Project Documents. Read/written via Drive REST API after Google login. All files live in a single folder named `.my-start-page` at the root of My Drive. The leading `.` is cosmetic — Drive does not treat it as hidden; it just makes the folder look unfamiliar enough that the user is less likely to delete it while browsing. Because this is a regular Drive folder (not the old app-private `appDataFolder`), other apps and tools (e.g., Claude.ai's Google Drive MCP connector) can also read these files.

Drive files (all inside `.my-start-page`):
- `dailyJob.json` — `{items:[{id,text,done}], lastResetDate}`. Resets at 06:00 KST (full clear, not just unchecking).
- `brainDump.json` — `[{id, category, name, createdAt}]`. `createdAt` is immutable after creation.
- `timeplan-YYYY-MM-DD.json` — `[{id, start, end, text, category, sourceType}]`. One file per day. `sourceType ∈ {'brainDump','dailyJob','manual'}` indicates origin of the block.
- `projectChart.json` — `[{id,name,purpose,tasks:[{id,text,done}],startDate,dueDate,createdAt}]`. Project progress chart data.
- `doc-<id>.json` — `{id, templateId, title, createdAt, updatedAt, sections:[{q,a}]}`. One file per project document. `id` is `uid()`. The list is discovered via `name contains 'doc-'` filtered to the `.my-start-page` folder ID. `.md`/`.txt` export is generated client-side from this JSON — nothing is persisted in those formats.

**Folder lookup (`driveGetFolderId`)**: on first authenticated use of the session, the app searches Drive by name+mimeType for a folder named `.my-start-page` that this OAuth client owns; if missing, it creates one. The folder ID is cached in memory (not localStorage) for the rest of the session, so a user-side deletion or rename is recoverable by reload. With `drive.file` scope the app can only see folders/files it created itself, which is why the cached approach is safe across page loads.

**One-time migration (`migrateFromAppData`)**: on the first authenticated load after the upgrade, every file in the legacy `appDataFolder` is copied into `.my-start-page` (existing destination files win — the copy is skip-if-present). After success, `localStorage['hp:migrated_v2']='1'` blocks further runs. The originals in `appDataFolder` are left in place as a safety net — they can be cleaned up by revoking app access in the Google Account settings.

## State → persistence → render flow

Every mutation calls its save function (`saveDailyJob`, `saveBrainDump`, `saveTimeplan(date)`) then re-renders. Save = synchronous localStorage write (`cacheSet`) + 300ms-debounced Drive PUT (`scheduleSave` → `driveSave`). When logged out, Drive PUT is skipped; localStorage still updates.

Load on init:
1. `cacheGet` for `dailyJob`/`brainDump`/`tp:<currentDate>` → paint immediately (cache-first).
2. If session token valid, `loadAllFromDrive()` fetches latest from Drive → may overwrite cache → re-render.

When date changes via nav (prev/next/today/picker), `onDateChange()` paints from cache for that date, then `loadTimeplan(date)` syncs from Drive.

## Docs (templates + project documents)

Two buttons sit between `shortcuts-grid` and `planner`: **새 글…** (template picker) and **나의 프로젝트 문서** (list browser). All state lives in `editorState` (active editor) and `docList` (last fetched list).

Editor flow: `docsPick(id)` → `docsEditorOpen({templateId, docId:null|<id>, title, sections:[{q,hint,a}]})`. Each section starts with `a === hint` and the textarea gets the `.has-hint` class (gray, italic). On first `input` event the class is removed permanently for that textarea — the hint stays editable; users delete what they don't need. Saved answer is whatever's in the textarea, including any hint text the user kept.

Save: synchronous to Drive only (no localStorage cache). `docsSaveToDrive(name, doc)` wraps `driveFind`/`driveCreate`/`driveUpdate` but returns a boolean instead of swallowing errors — the editor stays open and shows an alert on failure so the user doesn't lose work. Empty title is auto-filled with `<templateLabel>_<YYYY-MM-DD_HHmm>` at save time.

Draft recovery: every 300ms-debounced input writes `editorState` to `localStorage['hp:docs_draft']`. On opening the editor for a **new** doc whose `templateId` matches the draft, the user is prompted to restore. Saving or accepting "close without save" clears the draft. Drafts are per-template, not per-doc — there's only one slot at a time.

List: `docsFetchAll()` does a Drive search (`name contains 'doc-'`) then loads each file individually (N+1 fetch). Each row is double-clickable to open in the editor; row-level buttons offer `.md` download, `.txt` download, and delete (delete calls `driveDelete` which is a `DELETE` to `files/<id>` with the same 401-refresh retry as the rest of the Drive helpers). `.md` is generated by `docToMarkdown(doc)` (template label as blockquote header + sections as `## q\n\n a`); `.txt` is generated by `docToText(doc)` with `=` / `-` underlines.

## Auth

Google Identity Services token client with scopes `openid email profile https://www.googleapis.com/auth/drive.appdata https://www.googleapis.com/auth/drive.file`. `drive.file` is the primary scope (the app sees only files/folders it created in regular Drive); `drive.appdata` is kept solely for the one-time migration to read the legacy `appDataFolder` contents and can be dropped in a future cleanup.

Access token kept in `localStorage` (`drive_token`, `drive_token_expiry`, `drive_user`) so a single login persists across tabs and browser restarts. On 401 from Drive, `refreshToken()` does one silent retry via `requestAccessToken({prompt:''})`. On page load, if a stored `drive_user` exists but the token is missing/expired, `onGsiLoad` runs the same silent refresh before showing the login button — so a returning user auto-reconnects without UI.

**Scope version gate.** `localStorage['drive_scope_version']` is compared against the `SCOPE_VERSION` constant in code. When the constant changes (e.g., a new scope is added), `onGsiLoad` discards the stored token and forces a fresh consent click — silent refresh cannot grant scopes the user has never approved. Bump `SCOPE_VERSION` whenever you touch `SCOPES`.

`CLIENT_ID` constant is the OAuth Web Client public ID — safe to commit because origin restriction (`https://lyb2106.github.io` + `http://localhost:8000`) enforces security. To rotate, regenerate in GCP project `my-start-page-a48c0` and update the constant.

Logged-out state: page works read-only against localStorage cache. Edits still mutate cache but do not sync. On next login, Drive content **overwrites** local — local-only edits are lost.

## Conventions worth knowing

- **"Today" rolls over at 06:00 KST**, not midnight. `getPlannerDate()` returns the active planner day; Daily Job reset and `tpToday()` both use this. Don't use raw `new Date()` or `getKSTDateString` (removed) for planner-day comparisons.
- **Time grid:** 06:00 – 24:00, 30-min slots → 36 rows × 30px. Block layout: `top = (startMin - 360) / 30 × 30px`, `height = (endMin - startMin) / 30 × 30px`. Constants in code: `SLOT_MIN=30`, `SLOT_PX=30`, `HOUR_START=6`, `HOUR_END=24`, `SLOT_COUNT=36`.
- **`__CAL__` sentinel:** if a shortcut's `icon` field equals the string `'__CAL__'`, `calIcon()` generates a live canvas favicon showing today's calendar date number. Preserve this when touching `renderShortcuts()`.
- **Favicons** in `shortcuts.json` use `https://www.google.com/s2/favicons?domain=<host>&sz=64` — set manually when adding a shortcut.
- **XSS:** any user-entered string rendered via `innerHTML` (Daily Job text, Brain Dump name, block text, doc title, doc section question label) must go through `escapeHtml()`. Static template strings and OAuth-provided fields (user name, picture URL) are trusted. `textarea`'s inner content is also escaped (so `<` round-trips correctly via `el.value`).
- **Drag-and-drop moves the source.** Dragging Brain Dump or Daily Job item into Time Plan creates a new block AND deletes the source from its panel in a single action (`saveBrainDump`/`saveDailyJob` fire alongside `saveTimeplan`). `sourceType` field records origin but is informational only — the source no longer exists in the panel. No reverse drag (Time Plan → panel).
- **Block resize and drop snap to 30-min grid.** `Math.round(deltaY / SLOT_PX)`; never sub-slot precision.
- **Brain Dump's `createdAt` is immutable.** `bdSave` only writes `name` and `category` on edit — never overwrites `createdAt`.
- **Block overlap is allowed** but renders stacked (later block on top by z-index, hover raises). No side-by-side cluster layout.
- **`id` generation:** `uid()` = `Date.now().toString(36) + 6 random chars`. Avoids the collision risk of plain `Date.now()` in fast successive adds.
- **Shortcuts/bookmarks length is not enforced.** `shortcuts.json` may contain any number; UI just renders what's there. The original "10 fixed slots" rule was dropped along with the edit modal.
