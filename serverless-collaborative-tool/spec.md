# Serverless Collaborative HTML Tool — Architecture Specification

**Purpose of this document:** Hand-off specification for an LLM (or developer) to build a single-file HTML tool that supports multi-user collaboration on structured data, with optional local AI inference, in a corporate environment with no server access.

**Read this whole document before writing any code.** The architecture is unusual and the constraints drive most of the decisions.

---

## 1. Context and constraints

The user works in a corporate environment with strict limits:

- No ability to run a local server or deploy a backend
- Work laptop is managed; installing software requires IT approval
- No external cloud services that require corporate network exceptions (no Cloudflare, no public APIs for data)
- A shared network drive is available (mapped drive or UNC path), accessible to the whole team
- OneDrive sync exists but causes file conflicts when multiple users edit the same file
- Existing tools are built as single-file HTML (Simon Willison style), with SQLite-via-WASM for some
- The team currently uses OneNote heavily; OneNote is being deprecated, so this tool is part of the migration path
- Data is semi-structured: a template is loosely followed for incident records (problem / cause / solution / system / etc.)
- CSV imports from upstream systems (e.g., SAP extracts) need to feed the tool

**Hard requirements:**

- Runs as static HTML files served from a shared drive (no server)
- Multiple team members can edit concurrently without silent data loss
- Conflicts must be detected and surfaced, not silently overwritten
- Works in Microsoft Edge on a managed corporate laptop
- No external network dependencies for core functionality
- Should not be a dead end — must be extensible toward AI-assisted retrieval and generation

**Soft requirements:**

- Auto-save every 30-60 seconds
- Searchable across all records
- CSV import with column-to-field mapping
- OneNote HTML import for migration
- Eventually: semantic search over records, optionally local LLM inference

---

## 2. Architecture overview

The tool is a single HTML file (or small set of files) that lives on a shared network drive alongside its data. All users open it directly in their browser via a file path or SharePoint URL.

```
\\company-share\team-folder\pcss-tool\
├── index.html              # The application
├── data.json               # The data (git-tracked, granular structure)
├── .git/                   # Git repo metadata (managed by isomorphic-git)
├── model.gguf              # Optional: local LLM, pre-distributed
├── embeddings.bin          # Optional: precomputed embeddings cache
└── exports/                # Optional: snapshots, CSV exports
```

**Key architectural decisions and the reasoning:**

| Decision | Why |
|---|---|
| File System Access API for storage | Only browser API that can read/write a network share without a server |
| JSON as source of truth (not SQLite) | Git can merge JSON; binary `.db` files cause silent overwrites |
| Granular JSON structure (record → field per line) | Maximizes git's auto-merge success; matches OneNote's paragraph granularity |
| isomorphic-git for versioning | Pure-JS, no remote needed, operates directly on shared folder |
| SQLite WASM as in-memory query layer | Keeps SQL power for searching/filtering without paying the merge cost |
| Auto-save with merge loop | Sync granularity comparable to OneNote; conflict surface is small |
| Pre-distributed models on the share | Avoids painful download UX, enables fully air-gapped operation |

---

## 3. The merge model — read this carefully

This is the conceptual core. Get this wrong and the tool silently loses data.

**The data shape:**

```json
{
  "INC-2024-001": {
    "title": "PSU undervoltage on X3000",
    "system": "Power",
    "problem": "Voltage drops to 22V at startup",
    "cause": "Capacitor C47 degraded",
    "solution": "Replace C47 with high-temp variant",
    "tags": ["capacitor", "thermal"],
    "_updated_by": "sander",
    "_updated_at": "2026-04-16T10:30:00Z"
  },
  "INC-2024-002": { ... }
}
```

**Critical formatting rule:** the JSON file must be written with one field per line and stable key ordering. This makes git's line-based diff align with logical fields, which makes auto-merge work. Use a deterministic JSON serializer (sorted keys, 2-space indent, one field per line).

**The save loop:**

```
Every 30 seconds, if there are local changes:
  1. Read the current data.json from the shared drive (cheap)
  2. Get the git HEAD commit hash before our changes
  3. Compare to the commit hash we loaded from
     - If same: trivial commit, no merge needed
     - If different: someone else committed, run JSON merge
  4. Three-way merge:
     - base = the version we originally loaded
     - ours = base + our local edits
     - theirs = the current version on disk
     For each record:
       For each field:
         - If only ours changed: take ours
         - If only theirs changed: take theirs
         - If both changed identically: take either
         - If both changed differently: CONFLICT — surface to user
  5. Write merged data.json
  6. git add + git commit with message including user + timestamp
```

**Conflict UI:** when a real conflict happens, do not auto-resolve. Show both versions side by side with field name, ask the user which to keep, or let them edit a merged version.

**This is not a real-time sync system.** It's eventually-consistent with a sub-minute window. Acceptable for departmental tools where simultaneous edits to the same field are rare.

---

## 4. Library stack

All loaded from CDN in the HTML, no build step.

**File System Access API** — browser native, no library needed. `window.showDirectoryPicker()` returns a directory handle. Persist it in IndexedDB so users only pick the folder once. On each subsequent open, call `requestPermission({ mode: 'readwrite' })` to re-grant access.

**idb-keyval** — tiny IndexedDB wrapper for storing the directory handle and user preferences. Single function for get/set.

**isomorphic-git** — pure-JS git implementation. Operates against any filesystem you provide. Does init, add, commit, log, diff, branch, merge — all client-side, no remote required.

**A File System Access ↔ isomorphic-git filesystem adapter** — isomorphic-git expects a Node-fs-like API (`readFile`, `writeFile`, `readdir`, etc.). The File System Access API exposes a different shape. You need an adapter layer that translates between them. Search npm and GitHub for "isomorphic-git fsa adapter" — there are community implementations. If none fit, implement the small subset isomorphic-git needs (it's documented in their FS interface spec).

**SQLite WASM** (`@sqlite.org/sqlite-wasm`) — load this for in-memory querying. On data load, parse the JSON and INSERT records into an in-memory SQLite database. Use SQL for search, filter, aggregate. On write, update the JSON record and let the auto-save loop handle persistence. SQLite is read-only-ish from the user's perspective in this architecture — the durable store is JSON.

**Papa Parse** — CSV parsing. Streaming, handles quoted fields and edge cases. Used for CSV import.

**transformers.js** (Xenova) — for embedding-based semantic search. Loads ONNX models, runs them in WASM (or WebGPU if available). Use a small embedding model like `all-MiniLM-L6-v2` (~25MB) stored on the shared drive. Embed each record once, cache embeddings, recompute on edit.

**wllama** or **web-llm** — optional, for local LLM generation. wllama is closer to llama.cpp and uses WASM; web-llm uses WebGPU and is faster but has stricter browser requirements. Only add this in a later phase, and only if there's a clear use case the embedding-based search doesn't cover.

**Vanilla JS or a tiny templating layer** — `lit-html` or Alpine.js are good lightweight options. Avoid React/Vue and any build step.

---

## 5. Phased build order

Do not try to build everything at once. Each phase delivers standalone value and validates the riskiest assumption of that phase.

### Phase 0 — validate the environment (1-2 hours)

**Before writing any tool code**, write a 50-line HTML file that:

1. Calls `showDirectoryPicker()` and lets the user pick the shared drive folder
2. Writes a small JSON file to that folder
3. Reads it back
4. Persists the directory handle in IndexedDB
5. On reload, retrieves the handle and re-reads the file without re-prompting

**Test this on the actual work laptop with the actual shared drive.** This validates the single biggest unknown: whether Edge on a managed corporate laptop allows File System Access API against a mapped network drive or UNC path. If this works, the architecture is viable. If it doesn't, stop and reconsider — possibly fall back to download/upload UI patterns.

### Phase 1 — single-user structured tool with CSV import

Goal: replace the structured-data parts of OneNote usage for one person.

- Single HTML file, loads JSON from shared drive, displays records in a list/table
- Detail view per record with form fields matching the template structure
- Edit, save back to JSON (no git yet — just overwrite the file)
- CSV import with drag-drop, column mapping UI, append or merge mode
- OneNote HTML import: write a parser for the template, extract records, populate JSON
- SQLite WASM in memory for searching/filtering across all records
- Export back to CSV

This delivers immediate value. People can start using it. You learn the actual data shape from real usage.

### Phase 2 — add git versioning and the auto-save loop

Goal: enable safe multi-user editing.

- Wire in isomorphic-git via the FSA adapter
- On first run in a folder without a `.git`, run `git init`
- Auto-save every 30-60 seconds with the three-way merge described in section 3
- Build the conflict resolution UI
- Show edit history per record (`git log --follow` equivalent on the field)
- Display "last edited by X at Y" on each record from commit metadata

This is the riskiest engineering. Test with two browsers open against the same folder making conflicting edits. Verify nothing is silently lost.

### Phase 3 — semantic search

Goal: find similar past records by description, not just keyword.

- Add transformers.js with a small embedding model from the shared drive
- On record load, compute embedding if missing, cache in `embeddings.bin` (or as a field in JSON)
- Search bar accepts natural language; embed the query, find top-k by cosine similarity
- Show similarity scores in results

This is the main retrieval capability of a RAG system, and it's useful on its own without any generative LLM.

### Phase 4 — optional generation

Goal: synthesis, drafting, summarization.

- Add wllama or web-llm with a small generative model from the shared drive
- Use cases: summarize a record, suggest fields when importing unstructured text, draft a response
- Keep this clearly optional — embedding search delivers most of the RAG value and is much more reliable

---

## 6. File and module layout

For Phase 1, single file is fine. By Phase 2, split into modules using ES modules:

```
index.html              # Main HTML, imports the modules
modules/
  storage.js            # FSA wrappers, directory handle persistence
  git.js                # isomorphic-git setup, save loop, merge logic
  data.js               # JSON load/save, schema utilities
  query.js              # SQLite WASM setup, query helpers
  csv.js                # Papa Parse wrappers, import/export
  onenote-import.js     # HTML parser for OneNote exports
  embeddings.js         # transformers.js setup, embed/search
  ui/
    record-list.js      # List view
    record-detail.js    # Edit form
    conflict.js         # Merge conflict UI
    search.js           # Search bar with semantic results
```

Browsers support ES modules natively; no bundler needed if the files sit on the shared drive.

---

## 7. Things that will go wrong (and the answers)

**The shared drive path uses backslashes / has spaces / is a UNC path.** Test these explicitly in Phase 0. The File System Access API operates on directory handles, not paths, so it should be path-agnostic — but Windows + Edge + corporate group policies sometimes interfere.

**Two users hit save at the exact same moment.** The git commit will fail for one of them (the index will be locked or the commit will be a no-fast-forward). Catch the error, re-run the merge loop, retry. This is correct behavior.

**A user's browser has stale data and overwrites with old values.** The three-way merge prevents this — `base` is the version they loaded, so any field they didn't touch is taken from `theirs` (the current disk version), not from their stale local state.

**The JSON file becomes too large.** At ~10K records of moderate size, you're looking at maybe 20-50MB of JSON. Still workable but slow. Mitigation: split into multiple files (`data-2024.json`, `data-2025.json`) or partition by category. Don't optimize until it's actually a problem.

**Someone edits the JSON file directly in a text editor.** They might. Git will treat it as a normal commit. As long as they don't break the JSON syntax, this is fine and actually a useful escape hatch.

**The `.git` folder gets corrupted on the network share.** Possible if someone deletes files or if a write is interrupted mid-operation. Mitigation: periodically push to a second backup location, even just another folder on the same drive.

**File System Access API permissions reset between browser sessions.** This is browser policy. The directory handle persists in IndexedDB but the *permission* may need re-granting. Handle this gracefully — `requestPermission()` on load, prompt the user if denied.

**Models won't load because of memory limits.** A 7B-parameter model at 4-bit quantization is ~4GB in memory. Edge may cap browser memory below this on some configurations. Stick to 1-3B models for generation, or use API-based generation as fallback.

---

## 8. What this is NOT

To prevent scope creep:

- Not a real-time collaboration tool (Google Docs, Figma). Sub-minute eventual consistency is the design.
- Not a replacement for a proper database. SQLite WASM is a query layer, not the persistence layer.
- Not a replacement for proper version control on code. The git repo here is for data, not for the tool's source.
- Not a general AI assistant. The optional LLM is for narrow tasks (summarize a record, classify, draft).
- Not multi-tenant. One folder, one team. If multiple teams need separate data, they each get their own folder.

---

## 9. Validation checklist before declaring done

- [ ] Phase 0: Edge on managed laptop can read/write to the shared drive via FSA API
- [ ] Phase 1: One user can do a full session (load → edit → save → reload) with no data loss
- [ ] Phase 1: CSV import handles a real upstream file from production
- [ ] Phase 1: OneNote import successfully migrates at least 80% of existing records cleanly
- [ ] Phase 2: Two browsers editing different records simultaneously both succeed, no data lost
- [ ] Phase 2: Two browsers editing the same field simultaneously surface a conflict UI
- [ ] Phase 2: `git log` on the data file shows meaningful history with user attribution
- [ ] Phase 3: Semantic search returns relevant records for natural-language queries
- [ ] Documentation: A new team member can follow a setup guide to get the tool running

---

## 10. References for the implementing LLM

- File System Access API: https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API
- isomorphic-git: https://isomorphic-git.org/
- LightningFS (one option for FS layer): https://github.com/isomorphic-git/lightning-fs
- SQLite WASM official: https://sqlite.org/wasm/doc/trunk/index.md
- transformers.js: https://huggingface.co/docs/transformers.js
- wllama: https://github.com/ngxson/wllama
- web-llm: https://github.com/mlc-ai/web-llm
- Papa Parse: https://www.papaparse.com/
- Simon Willison's HTML tools patterns: https://simonwillison.net/2025/Dec/10/html-tools/

---

**End of specification.** Implementing LLM: when in doubt, start with Phase 0. Do not begin Phase 2 work until Phase 1 is in real use by at least one person and you have feedback on the actual data model.
