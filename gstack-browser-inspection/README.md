# How gstack Achieves Quick Web Browser Inspection

**Repo investigated**: https://github.com/garrytan/gstack  
**Key document**: `BROWSER.md` (1,300 lines, repo root)  
**Date**: 2026-05-04

---

## Summary

gstack achieves ~100-200ms browser inspection through three interlocking ideas:

1. **Persistent Chromium daemon** — pay the startup cost once (3s), then every
   subsequent command is a local HTTP round-trip to an already-warm browser.
2. **Plain-text stdout CLI** — skips all MCP protocol overhead (~1500–2000
   tokens saved per call vs Chrome MCP or Playwright MCP).
3. **Codified browser-skills** — a `/scrape` + `/skillify` loop that compiles
   multi-step browsing flows into deterministic Playwright scripts that run in
   ~200ms without LLM re-exploration.

---

## Architecture

```
Claude Code (Bash tool)
    │
    ▼ shell exec — ~1ms startup
browse/dist/browse CLI         ← compiled Bun binary (~58MB)
    │ HTTP POST
    │ 127.0.0.1:<random-port 10000-60000>
    │ Authorization: Bearer <root-token>
    ▼
Bun HTTP daemon                ← auto-spawned on first call
    │                            auto-stops after 30 min idle
    │ Playwright API             each project root gets its own
    ▼                            daemon, port, and state file
Chromium (headless or headed)
```

The state file lives at `<project>/.gstack/browse.json` (chmod 600). The CLI
reads it, posts one command, prints the response, and exits. About 1ms of its
own overhead.

---

## The Snapshot System — How `@ref` Works

**Source**: `browse/src/snapshot.ts`

The key browser inspection primitive is `$B snapshot`:

```bash
$B snapshot -i        # interactive elements only
$B snapshot -C        # + cursor-interactive (divs with pointer, onclick, tabindex)
$B snapshot -a        # + annotated screenshot with red overlay boxes
$B snapshot -D        # unified diff vs previous snapshot
```

Under the hood:

1. `page.locator('body').ariaSnapshot()` — Playwright's native accessibility
   tree API returns YAML-like text. **No DOM mutation. No injected scripts.**

2. Each line is parsed by regex into `{ role, name, props, children }`.

3. Each parsed node gets a ref (`@e1`, `@e2`, ...):
   - A Playwright `Locator` is built via `getByRole(role, { name }) .nth(n)`
   - `nth()` disambiguates when multiple elements share a role+name

4. The `Map<ref, Locator>` is stored on the tab session. Later commands
   (`click @e3`, `fill @e7 "text"`) look up the Locator and call the action.

**Staleness detection**: before using any ref, `resolveRef()` runs a 5ms
`count()` check. If the element count is 0 (SPA navigation mutated the DOM),
it throws immediately with a message to re-snapshot — rather than waiting
for Playwright's 30s action timeout.

### The `-C` cursor-interactive flag

The ARIA tree misses elements that are interactive but not semantically marked:
`div` menus, popover children, custom dropdowns. `-C` runs a
`page.evaluate()` scan that looks for:

- `cursor: pointer` in computed style
- `onclick` attribute
- `tabindex >= 0`
- Elements inside floating containers (Radix, Floating UI portals)

Each found element gets a deterministic `nth-child` CSS path and an `@c` ref.

---

## The CDP Inspector — `$B inspect`

**Source**: `browse/src/cdp-inspector.ts`

For deep CSS inspection:

```bash
$B inspect ".hero-section"           # full cascade + box model
$B inspect ".hero-section" --all     # include user-agent rules
$B inspect ".hero-section" --history # show CSS modifications since page load
```

Mechanism:
- Opens a persistent CDP session per page (`WeakMap<Page, CDPSession>`)
- Enables `DOM` and `CSS` domains on first use, auto-detaches on navigation
- Sends: `DOM.querySelector` → `DOM.getBoxModel` → `CSS.getMatchedStylesForNode`
  → `CSS.getComputedStyleForNode` → `CSS.getInlineStylesForNode`
- Post-processes: sorts matched rules by specificity (regex-parsed), marks
  overridden properties, handles `!important`, extracts pseudo-elements

Output: selector, tag/id/classes, box model (margin/border/padding/content),
matched rules sorted by specificity with overridden flags, inline styles,
~55 computed key properties, pseudo-elements.

The Chrome extension's "CSS Inspector" pane is a UI wrapper around this:
clicking any element on the page fires `$B inspect` via the sidebar and
shows the cascade interactively. A "Send to Code" button injects the result
into a live Claude PTY inside the Side Panel.

---

## `$B ux-audit` — Cheap Structural Map

```bash
$B ux-audit
```

Returns JSON with site identity, navigation, headings (capped 50), text
blocks, interactive elements (capped 200). Used by `/qa` and `/design-review`
for coverage mapping without dumping full HTML. No CDP session required —
pure Playwright `page.evaluate()`.

---

## The Productivity Loop — `/scrape` + `/skillify`

**Sources**: `scrape/SKILL.md`, `skillify/SKILL.md`

```
First call:  /scrape latest hacker news stories
             → agent drives page with $B goto, $B text, $B html, ...
             → returns JSON in ~30s
             → "Say /skillify to make this permanent (200ms next time)"

/skillify    → codifies the $B calls into a deterministic Playwright script
             → writes to ~/.gstack/browser-skills/hn-front/
             → atomic: stage → test → rename (or discard on failure)

Second call: /scrape hacker news front page
             → matches hackernews-frontpage skill
             → runs $B skill run hackernews-frontpage
             → returns same JSON in ~200ms
```

Browser-skills are self-contained directories:
```
browser-skills/<name>/
├── SKILL.md         # frontmatter: triggers, args, host, description
├── script.ts        # deterministic Playwright script
├── _lib/browse-client.ts  # vendored SDK copy (byte-identical, version-frozen)
├── fixtures/        # captured HTML for offline tests
└── script.test.ts   # parser tests against fixture (no daemon required)
```

Three-tier storage: project → global (`~/.gstack/`) → bundled (gstack install).
First hit wins — project-tier skills shadow global, global shadow bundled.

---

## Why Not MCP?

The BROWSER.md performance table explains the design choice:

| Tool              | First call | Subsequent | Context overhead/call |
|-------------------|-----------|------------|----------------------|
| Chrome MCP        | ~5s       | ~2-5s      | ~2000 tokens         |
| Playwright MCP    | ~3s       | ~1-3s      | ~1500 tokens         |
| gstack browse     | ~3s       | **100-200ms** | **0 tokens**      |
| gstack + skill    | ~3s       | **~200ms**    | **0 tokens**      |

MCP tools include full JSON schemas in every call. In a 20-command browser
session, that's 30,000–40,000 tokens of pure protocol framing. gstack's CLI
prints plain text; the LLM's Bash tool already exists, so they use it.

---

## Security Stack (noted)

6-layer prompt-injection defense sits between page content and Claude:

| Layer | What it does |
|-------|-------------|
| L1 | Datamarking — tag content origin |
| L2 | Strip hidden elements |
| L3 | ARIA + URL blocklist + envelope wrapping |
| L4 | TestSavantAI ONNX ML classifier (22MB, distilbert) |
| L4b | Claude Haiku transcript cross-check |
| L5 | Canary token (session-exfil detection — always blocks) |
| L6 | Ensemble combiner — BLOCK only when L4 AND L4b both ≥ 0.75 |

---

## Key Source Files

| File | What it does |
|------|-------------|
| `BROWSER.md` | Complete reference — this is the doc to read |
| `browse/src/snapshot.ts` | AX tree → @ref system (the core of quick inspection) |
| `browse/src/cdp-inspector.ts` | CDP-based deep CSS inspector |
| `browse/src/cli.ts` | Thin client — reads state, HTTP POST, prints response |
| `browse/src/server.ts` | Bun HTTP daemon — routes commands, dual-listener for ngrok |
| `browse/src/browser-manager.ts` | Chromium lifecycle, tab management, ref map |
| `browse/src/browser-skill-commands.ts` | `$B skill list/show/run/test/rm` |
| `browse/src/cdp-allowlist.ts` | Deny-default allowlist for raw CDP dispatch |
| `extension/inspector.js` | Chrome extension CSS Inspector panel |
| `scrape/SKILL.md` | `/scrape` skill: match-or-prototype entry point |
| `skillify/SKILL.md` | `/skillify` skill: codify last /scrape into permanent skill |
