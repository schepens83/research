# Setting Up Your Own AI Research Repository

A complete guide to replicating Simon Willison's [simonw/research](https://github.com/simonw/research) methodology — a system for running autonomous LLM coding agents on research questions and collecting their findings in a structured, reviewable format.

## Table of Contents

- [The Core Idea](#the-core-idea)
- [Repository Structure](#repository-structure)
- [Setting Up Your Repo](#setting-up-your-repo)
- [The AGENTS.md File](#the-agentsmd-file)
- [The Workflow](#the-workflow)
- [Writing Good Prompts](#writing-good-prompts)
- [Auto-Generated README with Cogapp](#auto-generated-readme-with-cogapp)
- [GitHub Actions for Automation](#github-actions-for-automation)
- [Tools of the Trade](#tools-of-the-trade)
- [Safety and Sandboxing](#safety-and-sandboxing)
- [Running Parallel Agents](#running-parallel-agents)
- [Lessons from the Original Repo](#lessons-from-the-original-repo)
- [Other People Who've Done This](#other-people-whove-done-this)
- [Open Points to Investigate](#open-points-to-investigate)
- [Sources](#sources)

## The Core Idea

Pick a research question. Spin up an asynchronous coding agent. Let it run experiments, write code, take notes, and report back when it's done. Review the results at your convenience.

Simon Willison's research repo contains 82+ projects — each one a self-contained investigation carried out entirely by an LLM (primarily Claude Code). Every line of text and code was written by the agent. Simon's role is to write prompts, review results, and merge PRs.

The key insight: a **dedicated research repo** (separate from production code) lets you give agents maximum freedom — full network access, fewer restrictions, no risk to real systems. The agent works asynchronously while you do other things.

## Repository Structure

```
research/
├── AGENTS.md                    # Instructions for coding agents (key file!)
├── CLAUDE.md                    # Points to AGENTS.md via @AGENTS.md
├── README.md                    # Auto-generated index of all projects
├── requirements.txt             # cogapp, llm, llm-github-models
├── .gitignore                   # .DS_Store, __pycache__, node_modules
├── .github/
│   └── workflows/
│       └── update-readme.yml    # Auto-updates README on push
│
├── project-alpha/               # Each project is a folder
│   ├── notes.md                 # Working notes (written during investigation)
│   ├── README.md                # Final report (written at end)
│   ├── _summary.md              # Auto-generated summary (by GitHub Action)
│   ├── code.py                  # Any code the agent wrote
│   └── results.json             # Any data/artifacts
│
├── project-beta/
│   ├── notes.md
│   ├── README.md
│   ├── _summary.md
│   └── ...
```

### What Goes in Each Project Folder

| File | Purpose | Created By |
|------|---------|------------|
| `notes.md` | Running log of what the agent tried and learned | Agent, during work |
| `README.md` | Polished final report with findings | Agent, at the end |
| `_summary.md` | 1-paragraph summary for the index | GitHub Action + LLM |
| Code files | Scripts, benchmarks, demos | Agent |
| Binary artifacts | Images, compiled outputs (<2MB) | Agent |
| `.diff` files | Git diffs of modified external repos | Agent |

### What Does NOT Go in the Folder

- Full copies of cloned repositories
- Large binary files (>2MB)
- `_summary.md` (created automatically — agents should not create this)

## Setting Up Your Repo

### Step 1: Create the Repository

```bash
mkdir my-research
cd my-research
git init
```

### Step 2: Create AGENTS.md

This is the most important file. It tells every coding agent how to behave in this repo.

```markdown
Start by creating a new folder for your work with an appropriate name.

Create a notes.md file in that folder and append notes to it as you work,
tracking what you tried and anything you learned along the way.

Build a README.md report at the end of the investigation.

Your final commit should include just that folder and selected items from
its contents:

- The notes.md and README.md files
- Any code you wrote along the way
- If you checked out and modified an existing repo, the output of "git diff"
  against that modified repo saved as a file - but not a copy of the full repo
- If appropriate, any binary files you created along the way provided they
  are less than 2MB in size

Do NOT include full copies of code that you fetched as part of your
investigation. Your final commit should include only new files you created
or diffs showing changes you made to existing code.

Don't create a _summary.md file - these are added automatically after you
commit your changes.
```

Note how **minimal** this is — just 15 lines. The simplicity is intentional. It gives agents freedom to explore while ensuring consistent, reviewable output.

### Step 3: Create CLAUDE.md

```markdown
@AGENTS.md
```

The `@` syntax tells Claude Code to include the contents of AGENTS.md. This way your agent instructions live in one place (AGENTS.md) and work with multiple tools.

### Step 4: Create .gitignore

```
.DS_Store
__pycache__

# npm dependencies
node_modules/
package.json
package-lock.json

# Pyodide cross-build environment
.pyodide-xbuildenv/
```

### Step 5: Create requirements.txt

```
cogapp
llm
llm-github-models
```

These power the automatic README generation:
- **[cogapp](https://nedbatchelder.com/code/cog/)** — Ned Batchelder's code generation tool. It runs Python code embedded in specially-marked comments and replaces them with output.
- **[llm](https://llm.datasette.io/)** — Simon Willison's CLI for interacting with LLMs.
- **[llm-github-models](https://github.com/simonw/llm-github-models)** — Plugin that lets `llm` use GitHub Models (free tier) for generating summaries.

### Step 6: Push to GitHub

```bash
git add -A
git commit -m "Initial commit"
git remote add origin https://github.com/YOUR_USERNAME/research.git
git push -u origin main
```

## The AGENTS.md File

AGENTS.md is the beating heart of this setup. It's recognized by multiple AI coding tools:

- **Claude Code**: Reads AGENTS.md from the repo root automatically
- **OpenAI Codex**: Reads AGENTS.md as agent instructions
- **GitHub Copilot Coding Agent**: Also reads AGENTS.md
- **Google Jules**: Reads AGENTS.md

The file acts as shared instructions across all agent platforms. Keep it short and focused on output format, not methodology — let agents pick their own approach.

### Customization Ideas

You could extend AGENTS.md with:
- Preferred programming languages
- Specific testing requirements
- Output format preferences (tables, charts, etc.)
- Links to tools the agent should use (e.g., `uvx rodney` for browser testing)
- References to other projects in the repo to build on

But resist the urge to over-specify. The original's power comes from its brevity.

## The Workflow

### Basic Flow (Using Claude Code for Web)

1. **Write a prompt** describing your research question
2. **Send it to Claude Code** (web interface at claude.ai/code, connected to your repo)
3. **Agent works autonomously**: creates folder, takes notes, experiments, writes report
4. **Agent files a PR** against your repo
5. **Review the PR** — the body contains your original prompt
6. **Merge** if the results are good, or provide follow-up prompts
7. **GitHub Actions** auto-regenerates the README index

### Basic Flow (Using Local Claude Code)

1. Open a terminal in your research repo
2. Run Claude Code with your prompt
3. Agent creates a folder, works, commits
4. Push to GitHub
5. GitHub Actions auto-regenerates the README

### Basic Flow (Using Other Agents)

Works with Codex, Jules, Copilot Coding Agent — all read AGENTS.md and follow the same pattern.

## Writing Good Prompts

Studying the 100+ prompts in simonw/research PRs reveals clear patterns:

### Direct Investigation
> Investigate the pdfium-render Rust crate - get it working and use it to turn a PDF into a folder full of rendered pages as JPGs

### Benchmark/Comparison
> Use Python to explore different ways of implementing tags in SQLite - JSON arrays, M2M tables, FTS, like queries. Design and run a benchmark to compare them

### Clone and Build
> Clone https://github.com/X/Y to /tmp - research what it takes to get a WebAssembly build of that working that can then be imported into and used from Pyodide in a browser

### Build on Prior Work
> Look at similar examples in this repo to see how that can work

### Specific Deliverables
> Check in the images in a folder but not the original PDF

### With Spec References
> Fetch https://gist.githubusercontent.com/... to /tmp with curl. React to option 2 in that document.

### Tips for Good Prompts

- **Be specific about goals** but **flexible about methods** — tell the agent what you want, not how to get it
- **Include URLs** to repos, docs, or specs the agent should reference
- **Specify output format** when it matters (JSON, images, benchmarks)
- **Reference other projects** in the repo when building on prior work
- **Keep follow-up prompts minimal** — a nudge is often enough ("Any other options that might help?")
- **Include practical constraints** ("check in the images but not the original PDF", "binary files under 2MB")

## Auto-Generated README with Cogapp

The README.md uses [cogapp](https://nedbatchelder.com/code/cog/) to embed Python that auto-generates the project index. Here's how it works:

### How Cogapp Works

Cogapp finds specially-marked sections in files:

```
<!--[[[cog
# Python code here
print("This output replaces the section")
]]]-->
(generated content appears here)
<!--[[[end]]]-->
```

Running `cog -r README.md` executes the Python and replaces the content between the markers.

### What the Script Does

The embedded Python in simonw/research README.md:

1. **Lists all subdirectories** and gets their first commit date via `git log`
2. **Sorts by date** (most recent first)
3. **Extracts the H1 title** from each project's README.md
4. **Checks for cached `_summary.md`** — if found, uses it
5. **If no summary exists**, calls `llm -m github/gpt-4.1` to generate a 1-paragraph summary from the README
6. **Saves the summary** to `_summary.md` for future caching
7. **Injects an AI-GENERATED-NOTE** banner into each project's README.md (skippable with `<!-- not-ai-generated -->`)

### The LLM Summary Prompt

```
Summarize this research project concisely. Write just 1 paragraph (3-5 sentences)
followed by an optional short bullet list if there are key findings. Vary your
opening - don't start with "This report" or "This research". Include 1-2 links
to key tools/projects. Be specific but brief. No emoji.
```

### Key Design Decisions

- Uses `github/gpt-4.1` via GitHub Models (free, no API key needed beyond GITHUB_TOKEN)
- Summaries cached in `_summary.md` so they're only generated once
- The `<!-- not-ai-generated -->` marker lets you include human-written projects
- Dates are normalized to UTC

## GitHub Actions for Automation

### The update-readme.yml Workflow

```yaml
name: Update README with cogapp

on:
  push:
    branches:
      - main

permissions:
  contents: write
  models: read

jobs:
  update-readme:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5
        with:
          fetch-depth: 0  # Need full history for git log dates

      - name: Set up Python
        uses: actions/setup-python@v6
        with:
          python-version: '3.11'
          cache: 'pip'
          cache-dependency-path: 'requirements.txt'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run cogapp to update README
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: cog -r -P README.md

      - name: Commit and push if changed
        run: |
          git config --local user.email "github-actions[bot]@users.noreply.github.com"
          git config --local user.name "github-actions[bot]"
          git add README.md
          git add */README.md 2>/dev/null || true
          git add */_summary.md 2>/dev/null || true
          if git diff --staged --quiet; then
            echo "No changes to commit"
          else
            git commit -m "Auto-update README with cogapp [skip ci]"
            git push
          fi
```

Key details:
- `fetch-depth: 0` — needs full git history to calculate project dates
- `permissions: models: read` — required for GitHub Models API access
- `[skip ci]` — prevents infinite loop (auto-commit doesn't retrigger the action)
- `GITHUB_TOKEN` — automatically provided, used by `llm-github-models` plugin

## Tools of the Trade

### Primary Agent Tools

| Tool | Type | Best For |
|------|------|----------|
| [Claude Code for web](https://claude.ai/code) | Async, cloud | Fire-and-forget research; files PRs automatically |
| [Claude Code CLI](https://www.claude.com/product/claude-code) | Local, interactive | Hands-on research where you want to watch/guide |
| [OpenAI Codex](https://openai.com/codex) | Async, cloud | Alternative async agent |
| [GitHub Copilot Coding Agent](https://github.com/features/copilot) | Async, cloud | GitHub-native workflow |
| [Google Jules](https://jules.google.com/) | Async, cloud | Google's coding agent |

### Supporting Tools

| Tool | Purpose |
|------|---------|
| [cogapp](https://nedbatchelder.com/code/cog/) | Code generation for README auto-updates |
| [llm](https://llm.datasette.io/) | CLI for calling LLMs (summary generation) |
| [llm-github-models](https://github.com/simonw/llm-github-models) | Free LLM access via GitHub Models |
| [Showboat](https://github.com/simonw/showboat) | CLI for agents to build visual demo documents |
| [Rodney](https://github.com/simonw/rodney) | CLI browser automation for agent testing |
| [uv](https://docs.astral.sh/uv/) | Fast Python package manager (used via `uvx`) |

### Showboat + Rodney (Visual Verification)

These are tools Simon built specifically for agents to demo their work:

- **Showboat**: Agents use it to construct Markdown documents with screenshots, command output, and explanatory text. Commands: `init`, `note`, `exec`, `image`, `pop`, `verify`.
- **Rodney**: CLI wrapper around Chrome DevTools Protocol. Agents use it to start Chrome, navigate URLs, click elements, run JavaScript, and take screenshots.

Together, they let agents visually verify that their code actually works — not just that tests pass. Install via `uvx showboat` and `uvx rodney`.

## Safety and Sandboxing

### Why a Separate Repo?

Research repos should be **isolated from production code**. This lets you:
- Grant full network access without worrying about data exfiltration
- Run agents with fewer permission restrictions
- Let agents install arbitrary dependencies
- Avoid accidental changes to production systems

### YOLO Mode (Maximum Autonomy)

For research tasks, Simon runs Claude Code with:
```bash
claude --dangerously-skip-permissions
```

This skips all permission prompts, letting the agent work fully autonomously. Only use this in isolated/sandboxed environments.

### Docker Sandboxing (Recommended for Risky Research)

```bash
# SSH into remote machine, start Docker container
docker run -it --rm ubuntu:latest bash
# Install Claude Code inside, run with skip-permissions
IS_SANDBOX=1 claude --dangerously-skip-permissions
```

The `IS_SANDBOX=1` flag tells Claude Code it's in a sandboxed environment.

### Key Safety Principle

> "Go forth and live dangerously! (But do it in a sandbox.)" — Simon Willison

The main risk is **prompt injection** — if an agent reads untrusted content (web pages, repos) that contains instructions designed to hijack the agent. Sandboxing limits the blast radius.

## Running Parallel Agents

Simon runs 2-3 research projects per day with minimal personal time:

- **Multiple terminal windows** with different agents in different directories
- **Fresh checkouts in `/tmp`** for isolation (not git worktrees)
- **Multiple agent tools** simultaneously: Claude Code, Codex, Jules
- **Bottleneck is review capacity**, not agent speed
- For async agents (Claude Code for web, Codex Cloud): just submit and walk away

### Tips for Parallel Research

1. Pick questions that are genuinely independent
2. Use async/cloud agents for maximum parallelism
3. Use `/tmp` checkouts to avoid interference
4. Batch your reviews — check all results at once
5. Let compound knowledge build — later projects reference earlier ones

## Lessons from the Original Repo

After analyzing 82+ projects and 100+ commits, here are the patterns that make this work:

### What Makes It Successful

1. **Minimal agent instructions** — AGENTS.md is 15 lines. Over-specifying constrains creativity.
2. **Consistent output format** — notes.md + README.md means every project is reviewable the same way.
3. **Automation handles the boring parts** — cogapp + GitHub Actions generate the index and summaries.
4. **Transparency** — Prompts are preserved in PR bodies. Transcripts are linked.
5. **Compound knowledge** — Projects reference each other. The repo gets more valuable over time.
6. **Low time investment** — Write a prompt, review later. 5-10 minutes of human time per project.
7. **Quality varies and that's OK** — Simon explicitly calls some output "total slop." The point is exploration, not production quality.

### Common Project Patterns

- **Benchmark/comparison**: Test multiple approaches, produce tables and recommendations
- **Feasibility study**: "Can X work with Y?" — agent proves or disproves it
- **Tool evaluation**: Clone a repo, build it, test it, document findings
- **Port/adaptation**: Take something from one language/platform to another
- **Security research**: Test attack vectors, demonstrate defenses
- **Integration**: Get two technologies working together

### What Agents Can't Do Well

- Prove something is **impossible** (they can only show feasibility)
- Make **subjective design decisions** (they need clear criteria)
- Work with **private/authenticated services** without setup help
- Handle **hardware-specific issues** without human intervention

## Other People Who've Done This

### Alex Ledger ([aled1027/research](https://aled1027.github.io/research/))

- 28 projects covering three.js, games, productivity tools, collaborative systems
- Uses Claude Code web interface
- Same AGENTS.md structure as Simon's
- Deploys interactive projects to GitHub Pages
- Reports "30 minutes with very little effort" for working results
- Blog post: [Remarkable Success Applying Simon Willison's Async Code Research Tasks](https://alexledger.substack.com/p/remarkable-success-applying-simon)

## Open Points to Investigate

These are areas worth exploring further when setting up your own version:

### Tooling Choices
- [ ] **Which LLM for summaries?** Simon uses `github/gpt-4.1` (free via GitHub Models). Alternatives: OpenAI API, Anthropic API, local models via Ollama.
- [ ] **Cogapp vs alternatives?** Cogapp is mature but niche. Alternatives: custom Python script, GitHub Actions that directly edit README, or a simpler approach entirely.
- [ ] **Do you need `llm` CLI?** It's powerful but adds a dependency. A direct API call in the GitHub Action might be simpler.

### Workflow Questions
- [ ] **GitHub Pages for interactive projects?** Alex Ledger does this effectively. Worth setting up if you build web demos.
- [ ] **PR-based vs direct-push?** Async agents (Claude Code for web) file PRs. Local agents push directly. PRs give you a review step but add friction.
- [ ] **AI-GENERATED-NOTE banners?** These are good practice for transparency. Consider whether you want them.
- [ ] **Topic focus vs broad exploration?** Simon covers everything from SQLite to Rust to browser security. A focused repo might be more useful.

### Infrastructure
- [ ] **Docker sandbox setup** for risky research tasks
- [ ] **Showboat + Rodney integration** for visual verification
- [ ] **Multiple agent tool accounts** for parallel work
- [ ] **Cost tracking** — Claude Code for web and other async agents aren't free

### Advanced Patterns
- [ ] **Cross-project references** — How to best structure prompts that build on prior work
- [ ] **Template projects** — Pre-configured folders for common research types
- [ ] **Quality gates** — Automated checks before merging (linting, test execution)
- [ ] **Archival** — How to handle projects that become outdated

## Sources

- [Code research projects with async coding agents like Claude Code and Codex](https://simonwillison.net/2025/Nov/6/async-code-research/) — Simon's primary blog post explaining the methodology
- [Getting DeepSeek-OCR working on an NVIDIA Spark via brute force using Claude Code](https://simonwillison.net/2025/Oct/20/deepseek-ocr-claude-code/) — Detailed walkthrough of one research project
- [Living dangerously with Claude](https://simonwillison.net/2025/Oct/22/living-dangerously-with-claude/) — Sandbox and safety approach
- [Embracing the parallel coding agent lifestyle](https://simonwillison.net/2025/Oct/5/parallel-coding-agents/) — Running multiple agents simultaneously
- [Introducing Showboat and Rodney, so agents can demo what they've built](https://simonwillison.net/2026/Feb/10/showboat-and-rodney/) — Visual verification tools for agents
- [Remarkable Success Applying Simon Willison's Async Code Research Tasks](https://alexledger.substack.com/p/remarkable-success-applying-simon) — Alex Ledger's experience replicating the methodology
- [simonw/research on GitHub](https://github.com/simonw/research) — The original repository
- [Simon Willison's async-coding-agents tag](https://simonwillison.net/tags/async-coding-agents/) — All related blog posts
