# Research Notes: Analyzing simonw/research for Replication

## Goal

Deep-dive into Simon Willison's `simonw/research` repository to understand its structure, methodology, tooling, and workflows, so we can create our own version following the same patterns.

## What I Investigated

### Repository Overview
- **Origin**: https://github.com/simonw/research
- **Author**: Simon Willison (creator of Datasette, django-sql-dashboard, llm CLI, etc.)
- **Purpose**: A dedicated repo where LLM coding agents (primarily Claude Code) autonomously run research experiments and report findings
- **Scale**: 82+ research projects, 249 commits, started late 2025
- **Contributors**: Simon Willison (107 commits), github-actions[bot] (87), Claude (52), Copilot (3)

### How I Traced the Methodology
1. Read AGENTS.md and CLAUDE.md (the agent instruction files)
2. Examined 10+ project directories for structural patterns
3. Read the README.md build script (cogapp-based auto-generation)
4. Read the GitHub Actions workflow
5. Analyzed 20 closed PR bodies to see actual prompts used
6. Fetched Simon's blog post: "Code research projects with async coding agents" (Nov 6, 2025)
7. Fetched his DeepSeek-OCR blog post for workflow details
8. Fetched his "Living dangerously with Claude" post for sandbox approach
9. Fetched his "Parallel coding agents" post for scaling tips
10. Fetched his "Showboat and Rodney" post for agent demo tooling
11. Read Alex Ledger's Substack post about replicating the methodology
12. Checked Alex Ledger's actual research repo at aled1027.github.io/research/

### Key Structural Findings

**Root-level files:**
- `AGENTS.md` - Instructions for coding agents (15 lines, very concise)
- `CLAUDE.md` - Just `@AGENTS.md` (pointer to AGENTS.md)
- `README.md` - Auto-generated index of all projects (huge, ~31K tokens)
- `requirements.txt` - cogapp, llm, llm-github-models
- `.gitignore` - .DS_Store, __pycache__, node_modules, .pyodide-xbuildenv
- `.github/workflows/update-readme.yml` - Auto-updates README on push

**Each project folder contains:**
- `notes.md` - Working notes, what was tried, observations (written by agent during work)
- `README.md` - Final polished report (written by agent at end)
- `_summary.md` - Auto-generated 1-paragraph summary (by GitHub Actions + LLM)
- Code files, scripts, data files as appropriate
- Binary artifacts if <2MB (e.g., wordcloud.png)

### AGENTS.md is Remarkably Minimal

The entire AGENTS.md is just 15 lines. It tells agents to:
1. Create a new folder with an appropriate name
2. Create notes.md and append to it as you work
3. Build a README.md report at the end
4. Final commit should include: notes.md, README.md, code, git diffs (not full repos), small binaries
5. Do NOT include full copies of fetched code
6. Don't create _summary.md (auto-generated)

This simplicity is deliberate - it gives agents freedom to explore while maintaining consistent output structure.

### The README Auto-Generation Pipeline

The README uses cogapp (a Python code generation tool by Ned Batchelder) with embedded Python that:
1. Iterates all subdirectories
2. Gets first commit date for each via `git log`
3. Extracts H1 title from each project's README.md
4. Checks for cached `_summary.md`; if missing, generates one via `llm` CLI using `github/gpt-4.1` model
5. Saves generated summaries to `_summary.md` for caching
6. Also injects an AI-GENERATED-NOTE banner into each project's README.md
7. Supports `<!-- not-ai-generated -->` marker to skip the banner

### GitHub Actions Workflow

On every push to main:
1. Checkout with full history (`fetch-depth: 0`)
2. Install Python 3.11 + requirements
3. Run `cog -r -P README.md`
4. Commit and push any changes to README.md, */README.md, */_summary.md
5. Uses `[skip ci]` in commit message to avoid infinite loops

### How Projects Get Created (The Actual Workflow)

From analyzing PR bodies, the workflow is:

1. **Simon writes a prompt** - Usually 2-5 sentences, sometimes longer with specific URLs or specs
2. **Sends to Claude Code for web** - The async agent that works on a server and files PRs
3. **Agent works autonomously** - Creates folder, takes notes, writes code, experiments
4. **Agent files a PR** - The PR body contains Simon's original prompt (in blockquote)
5. **Simon reviews** - Merges or provides follow-up prompts
6. **GitHub Actions auto-updates** - README regenerated with new project

### Prompt Patterns from PRs

Looking at actual prompts, they follow patterns:
- **Direct investigation**: "Investigate the pdfium-render Rust crate..."
- **Benchmark request**: "Use Python to explore different ways of implementing tags in SQLite..."
- **Clone and build**: "Clone https://github.com/X to /tmp - research what it takes to get..."
- **Tool reference**: "Run 'uvx rodney --help' to learn Rodney" (browser automation)
- **Building on prior work**: "look at similar examples in this repo to see how that can work"
- **Specific deliverables**: "check in the images in a folder but not the original PDF"

### Sandbox and Safety Approach

- Research repos get **full network access** (unlike production repos)
- For maximum autonomy: `--dangerously-skip-permissions` ("YOLO mode")
- Best practice: run inside Docker containers on remote hardware
- Monitor via VS Code Remote SSH watching notes.md in real-time
- The key insight: research repos are low-risk, so fewer restrictions are appropriate

### Parallel Agent Strategy

Simon runs 2-3 research projects per day with "minimal" personal time investment:
- Multiple terminal windows with different agents in different directories
- Fresh checkouts in `/tmp` for isolation (not git worktrees)
- Multiple agent tools: Claude Code, Codex CLI, Codex Cloud, Jules
- Bottleneck is code review capacity, not agent speed

### Showboat and Rodney (Agent Demo Tools)

- **Showboat**: CLI tool for agents to construct Markdown demo documents with screenshots
- **Rodney**: CLI browser automation (Chrome DevTools protocol) for agents to test web UIs
- Used together so agents can visually verify their work, not just pass tests
- Installed via `uvx showboat` and `uvx rodney`

### Alex Ledger's Adaptation

Alex Ledger replicated the methodology successfully:
- Created his own research repo at github.com/aled1027/research
- 28 projects (three.js, games, productivity tools, collaborative systems)
- Used Claude Code web interface
- Same AGENTS.md structure
- Key difference: his projects tend to be more interactive/visual (deployed to GitHub Pages)
- Reports "30 minutes with very little effort" for working results

### What Makes This Work

1. **Isolation**: Dedicated repo, not mixed with production code
2. **Minimal instructions**: AGENTS.md is short - agents have freedom to explore
3. **Consistent output**: notes.md + README.md pattern creates reviewable artifacts
4. **Automation**: cogapp + GitHub Actions handle index generation
5. **Transparency**: Prompts preserved in PR bodies, transcripts linked
6. **Asynchronous**: Fire-and-forget, review when ready
7. **Compound knowledge**: Projects in same repo can reference each other

### Open Questions for Our Own Setup

1. Which LLM provider/tool to use as primary? (Claude Code for web, local Claude Code, Codex?)
2. Should we use cogapp or a simpler README generation approach?
3. Do we want GitHub Pages deployment for interactive projects?
4. Should we integrate Showboat/Rodney for visual verification?
5. What topics/domains to focus research on?
6. Do we want the AI-GENERATED-NOTE banners?
7. Should we use a different LLM for summary generation (Simon uses github/gpt-4.1)?
8. How to handle the `llm` CLI setup and API keys for the GitHub Action?
