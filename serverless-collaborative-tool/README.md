# Serverless Collaborative HTML Tool — Phase 0+1

A minimal proof-of-concept for the architecture described in `spec.md`:
a single HTML file that reads and writes structured data on a shared folder using
the browser's File System Access API, with no server required.

## What was built

`index.html` — a single-file tool covering Phase 0 and Phase 1 of the spec:

**Phase 0 (environment validation):**
- Calls `showDirectoryPicker()` to let the user select a local or network drive folder
- Writes and reads `data.json` in that folder
- Persists the directory handle in IndexedDB via `idb-keyval` so the folder is remembered across sessions
- On reload, calls `handle.requestPermission({ mode: 'readwrite' })` to re-grant access without re-prompting
- Displays a confirmation banner when FSA read/write succeeds

**Phase 1 (single-user tool):**
- Generic key-value records stored as JSON (title + arbitrary user-defined fields)
- Add, edit, delete records
- Full-text search across all field values
- Deterministic JSON serializer (sorted keys, 2-space indent) — forward-compatible with git line-based merging in Phase 2
- Username stored in localStorage, stamped onto `_updated_by` and `_updated_at` metadata fields on every save

## What was NOT built (by design)

- **No SQLite WASM** — requires `COOP/COEP` HTTP headers for `SharedArrayBuffer`, which can't be set on a static file served from a network share. Plain JS filtering is used instead; adequate for hundreds of records. Revisit if search performance becomes an issue.
- **No isomorphic-git** — Phase 1 overwrites `data.json` directly. Phase 2 adds the three-way merge loop.
- **No CSV import, no OneNote import** — Phase 2+ features.

## Key research finding: FSA ↔ isomorphic-git adapter

The spec flagged this as the hardest unknown. Conclusion: **it's solved, no new library needed.**

The isomorphic-git `fs` interface requires just 8 async methods:
`readFile`, `writeFile`, `unlink`, `readdir`, `mkdir`, `rmdir`, `stat`, `lstat`

A complete, working adapter was found in [chigichan24/duff](https://github.com/chigichan24/duff)
(`client/src/lib/fsaAdapter.ts`). It's ~80 lines of TypeScript, trivially converted to plain JS,
and can be inlined directly into the HTML. It:
- Takes a `FileSystemDirectoryHandle` (the result of `showDirectoryPicker()`)
- Returns an `fs.promises`-compatible object
- Caches directory traversals for performance
- Is already in use in a real browser git tool

For Phase 2, inline this adapter and wire isomorphic-git's `git.commit()` into the save loop.

## Stack used

| Library | Purpose | Source |
|---|---|---|
| File System Access API | Read/write folder on local or network drive | Browser native |
| `idb-keyval@6` | Persist directory handle and username | `cdn.jsdelivr.net/npm/idb-keyval@6/+esm` |
| Alpine.js 3 | Reactive UI, no build step | `cdn.jsdelivr.net/npm/alpinejs@3` |

## What still needs validating on the actual corporate setup

Before building Phase 2, test `index.html` against the real environment:

1. **Does `showDirectoryPicker()` work on a managed Edge install?**
   Corporate group policies sometimes restrict the File System Access API.
   Open `index.html` in Edge and click "Pick folder" — if a folder picker appears, you're clear.

2. **Does it work against a UNC path (`\\server\share\folder`)?**
   The FSA API operates on handles, not paths, so it should be path-agnostic —
   but Windows + Edge + corporate network drive combinations can be unpredictable.
   Test by picking a folder that lives on the network share, not a local drive.

3. **Does the directory handle survive a browser restart?**
   Close Edge completely, reopen `index.html`. If it loads your records without re-prompting
   for a folder (just shows a permission banner that auto-resolves), handle persistence works.

4. **Does it work from a SharePoint URL?**
   If the folder is accessed via `https://company.sharepoint.com/...` rather than a mapped drive,
   test that scenario separately — SharePoint synced folders may behave differently.

## Phase 2 plan

With Phase 1 validated, Phase 2 adds the three-way merge loop:

1. Inline the FSA adapter (converted from duff's TypeScript)
2. On first use: `git.init()` the selected folder
3. Replace the direct `writeFile` call with the save loop from spec section 3:
   - Read current `data.json`
   - Check git HEAD hash against the version we loaded from
   - If changed: three-way JSON merge at field level
   - Surface conflicts in the UI
   - `git.add()` + `git.commit()` with user + timestamp
4. Test with two browser tabs on the same folder making concurrent edits

## Running it

Open `index.html` directly in Chrome or Edge (86+). No server, no build step.
Pick any local folder. The tool creates `data.json` there on first save.

For testing the network share scenario: map the drive, pick the mapped folder in the picker.
