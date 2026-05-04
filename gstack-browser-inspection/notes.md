# Investigation Notes — gstack Browser Inspection

## Repo

- **Source**: https://github.com/garrytan/gstack
- **Stars**: ~89k (as of 2026-05-04)
- **Created**: March 2026
- **Cloned to**: `/tmp/gstack`

## Goal

Understand how gstack achieves "quick web browser inspection" — the headline
feature described in BROWSER.md as "~100-200ms per call, zero context-token
overhead."

---

## Step 1 — Find the right file

`find /tmp/gstack -type f` showed a `BROWSER.md` at the repo root — the
main reference doc. Also found `browse/src/snapshot.ts`,
`browse/src/cdp-inspector.ts`, and `browse/src/cdp-commands.ts`.

---

## Step 2 — Read BROWSER.md

The document is ~1,300 lines covering:
- Architecture (CLI → HTTP daemon → Playwright → Chromium)
- ~70 commands across read/write/inspect/navigate/extract
- Snapshot system (the `@ref` mechanism)
- Browser-skills runtime (codified Playwright scripts)
- Security stack (6 layers of prompt-injection defense)
- Real-browser mode (`$B connect`)
- Side Panel Chrome extension with CSS Inspector

### Performance table from the doc

| Tool              | First call | Subsequent | Ctx overhead |
|-------------------|-----------|------------|--------------|
| Chrome MCP        | ~5s       | ~2-5s      | ~2000 tokens |
| Playwright MCP    | ~3s       | ~1-3s      | ~1500 tokens |
| gstack browse     | ~3s       | ~100-200ms | 0 tokens     |
| gstack + skill    | ~3s       | ~200ms     | 0 tokens     |

The speed advantage comes from the daemon being persistent — first call pays
the 3s Chromium startup, every call after that is HTTP POST → response.

---

## Step 3 — Read snapshot.ts

Key discovery: the snapshot system is built entirely on Playwright's native
`ariaSnapshot()` API. No DOM mutation, no injected scripts for the base
snapshot.

Flow:
1. `page.locator('body').ariaSnapshot()` → YAML-like accessibility tree text
2. Parse each line with a regex into role/name/props/children
3. Assign `@e1`, `@e2`, ... refs to each node
4. Build a Playwright `Locator` (via `getByRole` + `nth()`) per ref
5. Store `Map<string, Locator>` on the tab session
6. Return compact text to stdout

Flags:
- `-i` (interactive only): filters to roles in INTERACTIVE_ROLES set
- `-C` (cursor-interactive): `page.evaluate` scan for `cursor:pointer`,
  `onclick`, `tabindex>=0` on non-standard elements — these are things the
  ARIA tree misses. Assigns `@c1`, `@c2` refs with deterministic nth-child CSS paths.
- `-D` (diff): stores baseline, returns unified diff on next call
- `-a` (annotate): injects overlay `<div>`s at each ref's bounding box,
  screenshots, removes overlays
- `-H` (heatmap): same idea with color-coded overlays

Ref staleness: `resolveRef()` runs a 5ms `count()` check before using any
ref. If 0, throws immediately with a re-snapshot message.

---

## Step 4 — Read cdp-inspector.ts

The `$B inspect <selector>` command opens a persistent CDP session per page
and runs:
- `DOM.getDocument` → find the element node
- `DOM.getBoxModel` → content/padding/border/margin quads
- `CSS.getMatchedStylesForNode` → full rule cascade
- `CSS.getComputedStyleForNode` → ~55 key computed properties
- `CSS.getInlineStylesForNode` → inline style overrides

Post-processing:
- Sorts rules by specificity (highest first) computed by regex-parsing selectors
- Marks each property as overridden/active
- Handles `!important` correctly (can override higher-specificity rule)
- Handles pseudo-elements (::before, ::after)

The session is a WeakMap of `Page → CDPSession`. Auto-detaches on navigation,
re-creates transparently on next inspect call.

---

## Step 5 — Architecture summary

```
Claude Code Bash tool
    │
    ▼ shell exec (~1ms startup)
browse/dist/browse CLI    ← compiled Bun binary (~58MB)
    │ HTTP POST
    │ 127.0.0.1:<random-port>
    │ Bearer token
    ▼
Bun HTTP daemon           ← spawned on first call, persists 30 min idle
    │ Playwright API
    ▼
Chromium (headless or headed)
```

---

## Step 6 — Key insight on "quick"

The trick isn't any single clever API — it's the architecture: a persistent
local daemon means Chromium only starts once per session. After that, every
`$B` command is ~100ms because it's a local HTTP round-trip to an already-warm
browser.

The secondary insight is the **browser-skills runtime**: `/scrape` + `/skillify`
codifies a multi-step browsing flow into a deterministic Playwright script.
Second and subsequent calls run the script directly (~200ms) rather than having
the LLM re-explore the page (~30s).

The third insight is **plain text stdout**: MCP tools pay ~1500-2000 tokens
per call just for JSON-schema framing. gstack's CLI prints plain text; zero
protocol overhead.

---

## Files investigated

- `/tmp/gstack/BROWSER.md` — main reference doc
- `/tmp/gstack/browse/src/snapshot.ts` — snapshot + @ref system
- `/tmp/gstack/browse/src/cdp-inspector.ts` — CDP-based CSS inspection
- `/tmp/gstack/scrape/SKILL.md` — /scrape skill (match-or-prototype flow)
- `/tmp/gstack/browse/src/browser-manager.ts` — (noted, not read; handles Chromium lifecycle)
