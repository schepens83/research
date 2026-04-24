# How to Build a tools.simonwillison.net-Style Site

This is a step-by-step guide to recreate the architecture of [tools.simonwillison.net](https://tools.simonwillison.net/) — a collection of single-file HTML tools hosted on GitHub Pages with AI-generated docs, a search index, and full build automation.

---

## What You're Building

A GitHub-hosted static site where:
- Each tool is one self-contained `{slug}.html` file
- Git commit history drives the metadata (created date, updated date, commit messages)
- A Python build pipeline generates `index.html`, `colophon.html`, `by-month.html`, `dates.json`, `tools.json`
- An LLM (Claude Haiku via Anthropic API) auto-generates 2–3 sentence `.docs.md` descriptions for each tool
- A shared `footer.js` is injected into every tool at build time
- Everything deploys to GitHub Pages on push to `main`

---

## Step 1: Repo Initialization

```bash
git init tools
cd tools
git checkout -b main
```

Create `.gitignore`:
```gitignore
# Generated files (rebuilt at deploy time)
tools.json
gathered_links.json
colophon.html
index.html
by-month.html

# Python
__pycache__/
*.pyc
.venv/
*.egg-info/

# Node
node_modules/

# Playwright
test-results/
playwright-report/

# Cloudflare dev
.wrangler/
```

---

## Step 2: Python Project Config

Create `pyproject.toml`:
```toml
[project]
name = "tools"
version = "0.1"
requires-python = ">=3.10"
dependencies = [
    "pytest-playwright",
    "pytest-unused-port",
]
```

This lets CI do `pip install -e .` to get test dependencies.

---

## Step 3: The Build Scripts

### `gather_links.py`

This is the foundation — it reads git history and builds metadata for all tools.

```python
#!/usr/bin/env python3
"""
Scans all *.html files in the repo root, reads their git log, extracts
commit metadata, and writes:
  - gathered_links.json  (full commit history per file)
  - tools.json           (summary metadata array for search/index)
"""
import json
import subprocess
import re
from pathlib import Path

def get_git_log(filepath):
    result = subprocess.run(
        ["git", "log", "--format=%H|%aI|%s", "--follow", str(filepath)],
        capture_output=True, text=True
    )
    commits = []
    for line in result.stdout.strip().splitlines():
        if not line:
            continue
        parts = line.split("|", 2)
        if len(parts) == 3:
            hash_, date, message = parts
            # Extract URLs from commit message
            urls = re.findall(r'https?://\S+', message)
            commits.append({
                "hash": hash_,
                "date": date[:10],  # YYYY-MM-DD
                "message": message,
                "urls": urls,
            })
    return commits

def get_docs_description(slug):
    docs_path = Path(f"{slug}.docs.md")
    if not docs_path.exists():
        return ""
    content = docs_path.read_text()
    # Strip the commit comment line, return first real paragraph
    lines = [l for l in content.splitlines() if not l.startswith("<!--")]
    for line in lines:
        line = line.strip()
        if line:
            return line
    return ""

def get_html_title(filepath):
    content = Path(filepath).read_text(errors="replace")
    m = re.search(r"<title>(.*?)</title>", content, re.IGNORECASE | re.DOTALL)
    if m:
        return m.group(1).strip()
    return Path(filepath).stem

def main():
    html_files = sorted(Path(".").glob("*.html"))
    # Exclude generated files
    exclude = {"index.html", "colophon.html", "by-month.html"}
    html_files = [f for f in html_files if f.name not in exclude]

    gathered = {}
    tools = []

    for f in html_files:
        slug = f.stem
        commits = get_git_log(f)
        if not commits:
            continue

        gathered[f.name] = commits

        description = get_docs_description(slug)
        title = get_html_title(f)
        created = commits[-1]["date"]   # oldest commit
        updated = commits[0]["date"]    # newest commit

        tools.append({
            "slug": slug,
            "title": title,
            "description": description,
            "created": created,
            "updated": updated,
            "url": f"/{slug}",
        })

    Path("gathered_links.json").write_text(json.dumps(gathered, indent=2))

    # Sort tools by most recently updated
    tools.sort(key=lambda t: t["updated"], reverse=True)
    Path("tools.json").write_text(json.dumps(tools, indent=2))

    print(f"Wrote {len(tools)} tools to tools.json")

if __name__ == "__main__":
    main()
```

### `build_dates.py`

```python
#!/usr/bin/env python3
"""Generates dates.json: {filename: last-commit-date} for every tool."""
import json
from pathlib import Path

gathered = json.loads(Path("gathered_links.json").read_text())
dates = {fname: commits[0]["date"] for fname, commits in gathered.items() if commits}
Path("dates.json").write_text(json.dumps(dates, indent=2))
print(f"Wrote {len(dates)} entries to dates.json")
```

### `build_index.py`

```python
#!/usr/bin/env python3
"""
Converts README.md to index.html, injecting recently-added and
recently-updated tool lists from tools.json.
"""
import json
import markdown
from pathlib import Path

tools = json.loads(Path("tools.json").read_text())
readme = Path("README.md").read_text()

# Convert markdown to HTML
body_html = markdown.markdown(readme, extensions=["tables", "fenced_code"])

# Build recently-added and recently-updated sections
by_created = sorted(tools, key=lambda t: t["created"], reverse=True)[:5]
by_updated = sorted(tools, key=lambda t: t["updated"], reverse=True)[:5]

def tool_li(t):
    return f'<li><a href="{t["url"]}">{t["title"]}</a></li>'

recently_html = f"""
<h2>Recently added</h2>
<ul>{"".join(tool_li(t) for t in by_created)}</ul>
<h2>Recently updated</h2>
<ul>{"".join(tool_li(t) for t in by_updated)}</ul>
"""

# Inject between markers in the converted README
body_html = body_html.replace(
    "<!-- recently starts --><!-- recently stops -->",
    f"<!-- recently starts -->{recently_html}<!-- recently stops -->"
)

page = f"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>YOUR NAME's Tools</title>
<style>
  body {{ font-family: sans-serif; max-width: 800px; margin: 2rem auto; padding: 0 1rem; }}
  a {{ color: #6200ea; }}
</style>
</head>
<body>
{body_html}
<script type="module" src="/homepage-search.js" data-tool-search></script>
</body>
</html>"""

Path("index.html").write_text(page)
print("Wrote index.html")
```

### `build_colophon.py`

```python
#!/usr/bin/env python3
"""
Generates colophon.html: a transparency page listing every tool with its
full commit history (linkified), AI description, and GitHub links.
"""
import json
import re
from pathlib import Path

gathered = json.loads(Path("gathered_links.json").read_text())

GITHUB_BASE = "https://github.com/YOUR_USERNAME/YOUR_REPO/blob/main"

def linkify(text):
    return re.sub(r'(https?://\S+)', r'<a href="\1">\1</a>', text)

def get_description(slug):
    p = Path(f"{slug}.docs.md")
    if not p.exists():
        return ""
    lines = [l for l in p.read_text().splitlines() if not l.startswith("<!--")]
    for line in lines:
        if line.strip():
            return line.strip()
    return ""

rows = []
# Sort by most recently updated
items = sorted(gathered.items(), key=lambda kv: kv[1][0]["date"] if kv[1] else "", reverse=True)

for fname, commits in items:
    if not commits:
        continue
    slug = fname.replace(".html", "")
    desc = get_description(slug)
    commit_items = "".join(
        f'<li>{c["date"]} — {linkify(c["message"])} '
        f'(<a href="https://github.com/YOUR_USERNAME/YOUR_REPO/commit/{c["hash"]}">commit</a>)</li>'
        for c in commits
    )
    rows.append(f"""
<section>
  <h2><a href="/{slug}">{slug}</a>
    <small>(<a href="{GITHUB_BASE}/{fname}">source</a>)</small>
  </h2>
  {f"<p>{desc}</p>" if desc else ""}
  <details>
    <summary>{len(commits)} commit{"s" if len(commits) != 1 else ""}</summary>
    <ul>{commit_items}</ul>
  </details>
</section>
""")

html = f"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Colophon — YOUR NAME's Tools</title>
<style>
  body {{ font-family: sans-serif; max-width: 800px; margin: 2rem auto; padding: 0 1rem; }}
  section {{ border-bottom: 1px solid #eee; padding: 1rem 0; }}
  details {{ margin-top: 0.5rem; }}
  a {{ color: #6200ea; }}
</style>
</head>
<body>
<h1>Colophon</h1>
<p>All {len(rows)} tools and their full commit histories.</p>
{"".join(rows)}
</body>
</html>"""

Path("colophon.html").write_text(html)
print(f"Wrote colophon.html with {len(rows)} tools")
```

### `build_by_month.py`

```python
#!/usr/bin/env python3
"""Generates by-month.html: tools grouped by creation month."""
import json
from collections import defaultdict
from pathlib import Path

gathered = json.loads(Path("gathered_links.json").read_text())

def get_description(slug):
    p = Path(f"{slug}.docs.md")
    if not p.exists():
        return ""
    lines = [l for l in p.read_text().splitlines() if not l.startswith("<!--")]
    for line in lines:
        if line.strip():
            words = line.strip().split()
            return " ".join(words[:30]) + ("…" if len(words) > 30 else "")
    return ""

by_month = defaultdict(list)
for fname, commits in gathered.items():
    if not commits:
        continue
    slug = fname.replace(".html", "")
    created = commits[-1]["date"][:7]   # YYYY-MM
    by_month[created].append({
        "slug": slug,
        "description": get_description(slug),
        "date": commits[-1]["date"],
    })

sections = []
for month in sorted(by_month.keys(), reverse=True):
    items = sorted(by_month[month], key=lambda t: t["date"])
    lis = "".join(
        f'<li><a href="/{t["slug"]}">{t["slug"]}</a>'
        + (f' — {t["description"]}' if t["description"] else "")
        + "</li>"
        for t in items
    )
    sections.append(f"<h2>{month}</h2><ul>{lis}</ul>")

html = f"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>By month — YOUR NAME's Tools</title>
<style>
  body {{ font-family: sans-serif; max-width: 800px; margin: 2rem auto; padding: 0 1rem; }}
  a {{ color: #6200ea; }}
</style>
</head>
<body>
<h1>Tools by month</h1>
{"".join(sections)}
</body>
</html>"""

Path("by-month.html").write_text(html)
print("Wrote by-month.html")
```

### `write_docs.py`

Requires the `llm` CLI (`pip install llm llm-anthropic`) with `ANTHROPIC_API_KEY` set.

```python
#!/usr/bin/env python3
"""
For each *.html tool that has changed since its .docs.md was last generated,
run Claude Haiku to write a 2–3 sentence description and save it as {slug}.docs.md.
"""
import subprocess
import sys
from pathlib import Path

DRY_RUN = "--dry-run" in sys.argv
VERBOSE = "--verbose" in sys.argv
SKIP = {"index", "colophon", "by-month"}
MODEL = "claude-haiku-4-5"

SYSTEM_PROMPT = (
    "Write a paragraph of documentation for this HTML tool page as markdown. "
    "Do not include any headings. Do not use the words 'just' or 'simply'. "
    "Keep it to 2-3 sentences."
)

def get_last_commit_hash(filepath):
    result = subprocess.run(
        ["git", "log", "-1", "--format=%H", str(filepath)],
        capture_output=True, text=True
    )
    return result.stdout.strip()

def get_stored_hash(docs_path):
    if not docs_path.exists():
        return None
    for line in docs_path.read_text().splitlines():
        if line.startswith("<!-- Generated from commit:"):
            return line.split(":")[1].strip().rstrip(" -->")
    return None

def main():
    html_files = sorted(Path(".").glob("*.html"))
    updated = []

    for f in html_files:
        slug = f.stem
        if slug in SKIP:
            continue

        docs_path = Path(f"{slug}.docs.md")
        current_hash = get_last_commit_hash(f)
        stored_hash = get_stored_hash(docs_path)

        if current_hash == stored_hash:
            if VERBOSE:
                print(f"  skip {slug} (up to date)")
            continue

        print(f"Generating docs for {slug}...")
        if DRY_RUN:
            continue

        html_content = f.read_text(errors="replace")
        result = subprocess.run(
            ["llm", "-m", MODEL, "--system", SYSTEM_PROMPT],
            input=html_content,
            capture_output=True, text=True
        )
        if result.returncode != 0:
            print(f"  ERROR: {result.stderr}", file=sys.stderr)
            continue

        docs_content = result.stdout.strip() + f"\n\n<!-- Generated from commit: {current_hash} -->\n"
        docs_path.write_text(docs_content)
        updated.append(slug)
        print(f"  Wrote {docs_path}")

    print(f"\nUpdated {len(updated)} docs files")

if __name__ == "__main__":
    main()
```

### `build_redirects.py`

```python
#!/usr/bin/env python3
"""Generates meta-refresh redirect pages from _redirects.json."""
import json
from pathlib import Path

redirects = json.loads(Path("_redirects.json").read_text())

for slug, target in redirects.items():
    html = f"""<!DOCTYPE html>
<html>
<head>
<meta http-equiv="refresh" content="0; url={target}">
<title>Redirecting…</title>
</head>
<body>
<p>Redirecting to <a href="{target}">{target}</a>…</p>
</body>
</html>"""
    Path(f"{slug}.html").write_text(html)
    print(f"  {slug}.html → {target}")
```

Create `_redirects.json` (start empty):
```json
{}
```

---

## Step 4: `footer.js`

This file is injected at build time into every tool. It:
1. Adds a navigation footer (Home, About, View source, Changes)
2. Shows the last-updated date fetched from `dates.json`
3. Optionally shows a page weight badge

```javascript
// footer.js — injected by build.sh into every tool page
(function() {
  const slug = location.pathname.replace(/^\//, '').replace(/\.html$/, '');
  const GITHUB_REPO = 'https://github.com/YOUR_USERNAME/YOUR_REPO';
  const SITE_ROOT = 'https://YOUR_DOMAIN/';

  // Track visit in localStorage analytics
  const analytics = JSON.parse(localStorage.getItem('tools_analytics') || '[]');
  analytics.push({ slug, timestamp: Date.now() });
  // Keep last 500 visits
  if (analytics.length > 500) analytics.splice(0, analytics.length - 500);
  localStorage.setItem('tools_analytics', JSON.stringify(analytics));

  // Build footer
  const footer = document.createElement('footer');
  footer.style.cssText = 'padding: 1rem; margin-top: 2rem; border-top: 1px solid #eee; font-size: 0.85rem; color: #666;';
  footer.innerHTML = `
    <a href="/">← Home</a> ·
    <a href="/colophon#${slug}">About ${slug}</a> ·
    <a href="${GITHUB_REPO}/blob/main/${slug}.html">View source</a> ·
    <a href="${GITHUB_REPO}/commits/main/${slug}.html" id="footer-changes">Changes</a>
  `;
  document.body.appendChild(footer);

  // Fetch dates.json to show last-updated date
  fetch('/dates.json')
    .then(r => r.json())
    .then(dates => {
      const date = dates[slug + '.html'];
      if (date) {
        const a = document.getElementById('footer-changes');
        if (a) a.textContent = `Changes (updated ${date})`;
      }
    })
    .catch(() => {});
})();
```

---

## Step 5: `homepage-search.js`

```javascript
// homepage-search.js — search widget for index.html
(function() {
  const script = document.querySelector('[data-tool-search]');
  if (!script) return;

  let tools = [];
  let analytics = [];

  try {
    analytics = JSON.parse(localStorage.getItem('tools_analytics') || '[]');
  } catch(e) {}

  function visitCount(slug) {
    return analytics.filter(v => v.slug === slug).length;
  }

  function score(tool, query) {
    const q = query.toLowerCase();
    let s = visitCount(tool.slug) * 0.5;
    if (tool.slug.startsWith(q)) s += 10;
    if (tool.title.toLowerCase().includes(q)) s += 5;
    if (tool.description.toLowerCase().includes(q)) s += 2;
    return s;
  }

  function render(query) {
    const box = document.getElementById('tool-search-results');
    if (!query.trim()) { box.innerHTML = ''; return; }
    const results = tools
      .map(t => ({ ...t, _score: score(t, query) }))
      .filter(t => t._score > 0 || t.slug.includes(query.toLowerCase()) || t.title.toLowerCase().includes(query.toLowerCase()))
      .sort((a, b) => b._score - a._score)
      .slice(0, 12);
    box.innerHTML = results.map(t =>
      `<li><a href="${t.url}">${t.title}</a>${t.description ? ' — ' + t.description : ''}</li>`
    ).join('');
  }

  fetch('/tools.json')
    .then(r => r.json())
    .then(data => {
      tools = data;
      // Build UI
      const h1 = document.querySelector('h1');
      const wrapper = document.createElement('div');
      wrapper.innerHTML = `
        <input id="tool-search" type="search" placeholder="Search tools…"
          style="font-size:1rem;padding:0.5rem;width:100%;max-width:400px;margin:1rem 0">
        <ul id="tool-search-results" style="list-style:none;padding:0"></ul>
      `;
      h1.after(wrapper);
      const input = document.getElementById('tool-search');
      input.addEventListener('input', e => render(e.target.value));
      document.addEventListener('keydown', e => {
        if (e.key === '/' && document.activeElement !== input) {
          e.preventDefault();
          input.focus();
        }
      });
    })
    .catch(console.error);
})();
```

---

## Step 6: `build.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Unshallow if we're in a shallow clone (GitHub Actions default)
if git rev-parse --is-shallow-repository | grep -q true; then
  echo "Unshallowing repository..."
  git fetch --unshallow
fi

# Gather git metadata into gathered_links.json + tools.json
python3 gather_links.py

# Generate LLM docs if requested (requires llm CLI + ANTHROPIC_API_KEY)
if [ "${GENERATE_LLM_DOCS:-0}" = "1" ]; then
  echo "Generating LLM docs..."
  python3 write_docs.py
fi

# Build generated pages
python3 build_colophon.py
python3 build_dates.py
python3 build_index.py
python3 build_by_month.py

# Inject footer.js into every root-level .html tool
# (skip index.html, colophon.html, by-month.html, redirect stubs)
FOOTER_HASH=$(git rev-parse HEAD:footer.js 2>/dev/null || echo "v1")
SKIP_PATTERN="^(index|colophon|by-month)\.html$"

for f in *.html; do
  [[ "$f" =~ $SKIP_PATTERN ]] && continue
  # Skip if footer.js already injected
  if grep -q "footer.js" "$f"; then
    continue
  fi
  # Inject <script src="/footer.js?v=HASH"> before </body>
  awk -v hash="$FOOTER_HASH" '
    /<\/body>/ && !found {
      print "<script src=\"/footer.js?v=" hash "\"></script>"
      found=1
    }
    { print }
  ' "$f" > "$f.tmp" && mv "$f.tmp" "$f"
  echo "  Injected footer into $f"
done

# Build redirects last (avoid them appearing in index)
python3 build_redirects.py

echo "Build complete."
```

---

## Step 7: Jekyll Config (`_config.yml`)

GitHub Pages runs Jekyll automatically. Add this to prevent it from mangling your pre-built HTML files:

```yaml
title: YOUR NAME's Tools
description: A collection of browser-based utilities
theme: jekyll-theme-primer
```

And `CNAME` file:
```
tools.yourdomain.com
```
(or delete CNAME and use the default `username.github.io/repo` URL)

---

## Step 8: README.md

The README is the source of truth for the tool listing AND it gets converted to `index.html`. Structure it with these comment markers so recently-added/updated items can be injected:

```markdown
# YOUR NAME's Tools

<!-- recently starts --><!-- recently stops -->

## Category Name

- [Tool Title](https://tools.yourdomain.com/tool-slug) — short description

## Another Category

- [Another Tool](https://tools.yourdomain.com/another-tool) — description
```

---

## Step 9: Your First Tool

Create `hello-world.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Hello World</title>
<style>
  body { font-family: sans-serif; max-width: 600px; margin: 2rem auto; padding: 0 1rem; }
</style>
</head>
<body>
<h1>Hello World</h1>
<p>Your first tool goes here.</p>
</body>
</html>
```

Commit it. This creates the git history that `gather_links.py` will read.

```bash
git add hello-world.html
git commit -m "Add hello-world tool"
```

---

## Step 10: GitHub Actions Workflows

### `.github/workflows/pages.yml` (main deploy)

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write    # needed to commit .docs.md files back
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # CRITICAL: full history needed for gather_links.py

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
          cache-dependency-path: pyproject.toml

      - name: Install dependencies
        run: pip install markdown llm llm-anthropic

      - name: Build site
        env:
          GENERATE_LLM_DOCS: "1"
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: bash build.sh

      - name: Commit generated docs
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          # Stage any new/changed .docs.md files
          CHANGED=$(git diff --name-only HEAD -- '*.docs.md' && git ls-files --others --exclude-standard '*.docs.md')
          if [ -n "$CHANGED" ]; then
            git add *.docs.md
            STEMS=$(echo "$CHANGED" | sed 's/\.docs\.md//' | tr '\n' ' ' | xargs)
            git commit -m "Generated docs: $STEMS"
            git push
          else
            echo "No docs changes to commit"
          fi

      - name: Setup Pages
        uses: actions/configure-pages@v5

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: "."

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### `.github/workflows/test.yml`

```yaml
name: Run tests

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
          cache-dependency-path: pyproject.toml

      - name: Install dependencies
        run: pip install -e .

      - name: Cache Playwright browsers
        uses: actions/cache@v4
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ hashFiles('package-lock.json') }}

      - name: Install Playwright
        run: playwright install --with-deps chromium

      - name: Run tests
        run: pytest
```

### `.github/workflows/claude.yml` (optional — @claude in issues/PRs)

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'issues' && contains(github.event.issue.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude'))
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      - uses: anthropics/claude-code-action@beta
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

### `.github/workflows/claude-code-review.yml` (optional — auto PR review)

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@beta
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          direct_prompt: |
            Review this pull request focusing on:
            - Code quality and correctness
            - Security issues
            - Performance considerations
            - Whether the changes match the PR description
            Provide concise, actionable feedback.
```

---

## Step 11: GitHub Secrets to Configure

In your GitHub repo → Settings → Secrets and variables → Actions, add:

| Secret | How to get it |
|--------|--------------|
| `ANTHROPIC_API_KEY` | [console.anthropic.com](https://console.anthropic.com) → API keys |
| `CLAUDE_CODE_OAUTH_TOKEN` | Optional — for `@claude` in issues. [docs.anthropic.com/claude-code](https://docs.anthropic.com/en/docs/claude-code/github-actions) |

---

## Step 12: GitHub Pages Setup

1. Go to your repo → Settings → Pages
2. Source: **GitHub Actions**
3. If using a custom domain: add your domain and set up DNS. The `CNAME` file handles the GitHub side.

---

## Step 13: Install Python Build Dependencies Locally

```bash
pip install markdown llm llm-anthropic
llm keys set anthropic   # paste your API key
```

Test the build locally:
```bash
git fetch --unshallow || true   # only needed if you cloned with --depth
bash build.sh
```

---

## Step 14: Tests Setup

Create `tests/conftest.py`:
```python
import subprocess
import time
import pytest

class StaticServer:
    def __init__(self, port):
        self.port = port
        self.proc = subprocess.Popen(
            ["python", "-m", "http.server", str(port)],
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
        )
        # Wait for server to be ready
        import urllib.request, urllib.error
        for _ in range(50):
            try:
                urllib.request.urlopen(f"http://127.0.0.1:{port}/")
                break
            except Exception:
                time.sleep(0.1)

    def url(self, path=""):
        return f"http://127.0.0.1:{self.port}/{path}"

    def stop(self):
        self.proc.terminate()
        self.proc.wait()

@pytest.fixture
def static_server(unused_tcp_port):
    server = StaticServer(unused_tcp_port)
    yield server
    server.stop()
```

---

## Step 15: Ongoing Workflow

**Adding a new tool:**
1. Create `{slug}.html` in repo root — fully self-contained HTML
2. Add it to `README.md` under the appropriate category
3. Commit with a descriptive message (include links to any LLM conversations used)
4. Push — CI will auto-generate the `.docs.md` and rebuild the site

**Build pipeline on push:**
1. `gather_links.py` reads git history → `gathered_links.json` + `tools.json`
2. `write_docs.py` calls Claude Haiku for any new/changed tools → `*.docs.md`
3. CI commits any new `.docs.md` files back
4. Build scripts generate `index.html`, `colophon.html`, `by-month.html`, `dates.json`
5. `build.sh` injects `footer.js` into every tool
6. GitHub Pages deploys

**The colophon** at `/colophon` shows every tool's full commit history with linkified messages — great for showing LLM transcript links.

---

## Optional: Cloudflare Workers (for GitHub OAuth in tools)

If tools need GitHub API access via OAuth:

1. Install Wrangler: `npm install -g wrangler`
2. Create `cloudflare-workers/github-auth/worker.js` (OAuth relay)
3. Create `cloudflare-workers/github-auth/wrangler.toml`:
   ```toml
   name = "your-github-auth-worker"
   main = "worker.js"
   compatibility_date = "2024-01-01"
   ```
4. Deploy: `wrangler deploy`
5. Set secrets: `wrangler secret put GITHUB_CLIENT_ID` etc.
6. Add `.github/workflows/deploy-cloudflare-workers.yml` for auto-deployment

---

## Optional: Vercel Anthropic Proxy (for tools that call Claude)

For tools that call the Anthropic API from the browser (bypasses CORS):

1. Create `vercel/anthropic-proxy/index.js` (Express CORS proxy)
2. Create `vercel/anthropic-proxy/vercel.json`
3. Deploy to Vercel
4. In tool HTML, use the proxy URL instead of `api.anthropic.com` directly

---

## Summary: What to Customize

Replace these placeholders throughout:
- `YOUR_USERNAME` → your GitHub username
- `YOUR_REPO` → your repo name  
- `YOUR_DOMAIN` → your custom domain (or `username.github.io/repo`)
- `YOUR NAME` → your display name in page titles
- The README categories and tool links

The git history IS the database. Every tool's metadata (creation date, update date, commit messages with LLM transcript links) comes from git log. Keep commits atomic and descriptive.
