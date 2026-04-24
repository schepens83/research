# Research Notes: Serverless Collaborative HTML Tool

## Goal

Build Phase 0 + Phase 1 of the serverless collaborative tool described in `spec.md`.
Validate the File System Access API + isomorphic-git stack is actually buildable
before committing to the full architecture.

## Research: FSA ↔ isomorphic-git adapter

The spec warns this adapter is the tricky piece. Searched for existing implementations.

### What exists

**LightningFS** (`@isomorphic-git/lightning-fs`) — the official browser FS for isomorphic-git.
Uses IndexedDB as its backing store. This means git state lives in IndexedDB, *not* on the
actual shared drive. Doesn't help here — we need git to operate directly on `data.json`
sitting on the network share.

**isomorphic-git FS interface** — documented at https://isomorphic-git.org/docs/en/fs
The required `promises` API is just 8 methods:
`readFile`, `writeFile`, `unlink`, `readdir`, `mkdir`, `rmdir`, `stat`, `lstat`
Plus optional `readlink`, `symlink`, `chmod`.

**chigichan24/duff** — a browser-only Git diff viewer built on FSA + isomorphic-git.
Contains a complete, working FSA adapter at `client/src/lib/fsaAdapter.ts`.
Fetched the source via GitHub API. Key findings:
- ~80 lines of TypeScript, trivially convertible to plain JS
- Uses a `WeakMap` to cache adapter instances per directory handle
- Uses a `Map` for per-path handle caching (avoids redundant `getDirectoryHandle` traversals)
- Implements all 8 required methods plus `readlink`/`symlink` as no-ops
- Sets `fs.promises = fs` (isomorphic-git's preferred interface)
- Path handling: splits on `/`, traverses from root handle, caches intermediate handles

This adapter is the answer to spec section 4's "search for a community implementation
or implement the small subset yourself" — it's small enough to inline directly.

### No published npm package for FSA↔isomorphic-git

No dedicated package found on npm. The duff adapter is the cleanest reference
implementation available. For Phase 2 we'll inline it (converted to JS) directly
in the HTML or as a module.

## Phase 0 implementation notes

Built into `index.html`. Key decisions:

- `showDirectoryPicker({ mode: 'readwrite' })` — opens native folder picker
- `idb-keyval` from jsDelivr CDN (ESM) — persists the `FileSystemDirectoryHandle` in IndexedDB
- On reload: `handle.requestPermission({ mode: 'readwrite' })` — re-grants without re-prompting
  (Edge/Chrome require this gesture; the stored handle alone isn't enough after a session ends)
- Writes a test sentinel file `data.json` to confirm the folder is writable

## Phase 1 implementation notes

Single HTML file. No build step. Libraries from CDN:
- `idb-keyval@6` (ESM) — handle + username persistence
- Alpine.js 3 — reactive UI, no build step, `x-data`/`x-bind` style

Intentional omissions for Phase 1 minimal proof:
- **No SQLite WASM** — SQLite WASM from `file://` or a network share URL requires
  `Cross-Origin-Embedder-Policy: require-corp` and `Cross-Origin-Opener-Policy: same-origin`
  headers for SharedArrayBuffer. These headers can't be set on a static file served
  from a network share without a server. The non-SharedArrayBuffer build exists but
  adds complexity. Phase 1 uses plain JS filtering instead — adequate for hundreds of records.
  Revisit in Phase 1.5 if search performance becomes a problem.
- **No isomorphic-git** — Phase 1 overwrites `data.json` directly. Phase 2 adds the save loop.
- **No CSV import** — Phase 1 focuses on the FSA API + data model validation.
- **No OneNote import** — same.

### JSON serialization

Implemented deterministic serializer: sorted keys at every level, 2-space indent.
This is forward-compatible with git line-based merging (Phase 2 requirement).
JSON.stringify with a replacer doesn't recurse, so implemented manually.

### Record model

Generic key-value records. Each record:
```json
{
  "REC-1745000000000-A4B2": {
    "_id": "REC-1745000000000-A4B2",
    "_updated_at": "2026-04-24T10:00:00Z",
    "_updated_by": "sander",
    "title": "Record title",
    "anykey": "anyvalue"
  }
}
```

Internal/metadata keys prefixed with `_`. User-defined fields are anything else.
The edit form lets users add/remove arbitrary key-value pairs dynamically.

## Phase 2 implementation notes

Wired isomorphic-git into the Phase 1 tool.

### FSA adapter (inlined)
Converted the duff TypeScript adapter to plain JS (~80 lines). One fix: the original
returns `Buffer.from(u8)` for binary reads (Node.js only). For browser-only use, returning
`Uint8Array` directly works — isomorphic-git's browser build accepts Uint8Array natively.
Also added `handleCache.delete(path)` after `writeFile`/`unlink` to avoid stale handles.

### CDN loading
isomorphic-git loaded from `esm.sh/isomorphic-git@1` — esm.sh bundles all internal
dependencies into a single ES module, which is more reliable than jsDelivr `+esm` for
packages with complex internal imports.

### dir: '/' convention
isomorphic-git's `dir` parameter is the working directory root. With `dir: '/'` and the
FSA adapter rooted at the picked folder, all paths resolve correctly:
- `data.json` → rootHandle.getFileHandle('data.json')
- `.git/HEAD` → rootHandle.getDirectoryHandle('.git') → getFileHandle('HEAD')
- isomorphic-git's internal path.join('/', 'data.json') → '/data.json' → ['data.json'] ✓

### First-run git init flow
On `pickFolder()` or `init()` from IndexedDB:
1. `git.resolveRef({ ref: 'HEAD' })` — if throws, no git exists
2. `git.init({ defaultBranch: 'main' })` — creates .git/ in the folder
3. Attempt initial commit if data.json already exists

### The save loop
`trySave()` is called on every `saveRecord()` / `deleteRecord()` and by a 30s auto-save timer.
Flow:
1. Get `currentHead = resolveRef('HEAD')`
2. If `currentHead === baseCommit`: write data.json + git add + git commit
3. If different: read current data.json from disk, call `threeWayMerge(baseData, ourRecords, diskData)`
4. If no conflicts: apply merged, write, commit
5. If conflicts: surface conflict UI, let user resolve field by field, then commit

### Three-way merge logic
Handles all cases from the spec:
- Only ours changed → take ours
- Only theirs changed → take theirs
- Both changed identically → take either
- Both changed differently → CONFLICT, default to ours, user can override
- Record deleted by one side vs edited by other → CONFLICT (shown as special case)
- New records added by one side → auto-merged (no conflict)

### Git status badge
Header shows three states:
- `✓ committed` (green) — all changes are committed
- `● unsaved` (orange) — local changes not yet committed
- `⚠ conflict` (red) — merge conflict waiting to be resolved

### Testing concurrent edits
To test Phase 2: open `index.html` in two separate Chrome/Edge windows pointing at the
same folder. Edit different fields in the same record in both windows. Save window 1,
then save window 2 — window 2 should detect the concurrent commit and auto-merge.
To trigger a conflict: edit the same field in both windows, save window 1 first, then
save window 2 — the conflict UI should appear.

## What still needs validating on the actual corporate setup

- Does Edge on a managed laptop allow `showDirectoryPicker()` at all? (Group policy may block it)
- Does FSA API work against a UNC path (`\\server\share\folder`) or only local drives?
- Does the directory handle survive a browser restart, or does it always require re-picking?
  (Spec says `requestPermission()` is needed; whether the handle itself persists is policy-dependent)
- Does concurrent write from two Edge tabs on the same machine work? (Phase 2 question)
