# Research

Research projects carried out by AI tools and documented investigations.

Each directory is a separate project — some involve coding and experiments,
others are purely written investigations capturing lessons learned or answers
to research questions.

*Times shown are in UTC.*

<!--[[[cog
import os
import re
import subprocess
import pathlib
from datetime import datetime, timezone

MODEL = "github/gpt-4.1"

research_dir = pathlib.Path.cwd()
subdirs_with_dates = []

for d in research_dir.iterdir():
    if not d.is_dir():
        continue
    if d.name.startswith('.') or d.name.startswith('_'):
        continue
    if d.name == '.github':
        continue
    readme = d / 'README.md'
    if not readme.exists():
        continue
    result = subprocess.run(
        ['git', 'log', '--follow', '--format=%aI', '--', str(readme)],
        capture_output=True, text=True
    )
    dates = result.stdout.strip().split('\n')
    first_date = dates[-1] if dates and dates[-1] else None
    if first_date:
        dt = datetime.fromisoformat(first_date).astimezone(timezone.utc)
        subdirs_with_dates.append((dt, d))

subdirs_with_dates.sort(key=lambda x: x[0], reverse=True)

for dt, d in subdirs_with_dates:
    readme = d / 'README.md'
    content = readme.read_text()

    # Extract H1 title
    title_match = re.search(r'^#\s+(.+)$', content, re.MULTILINE)
    title = title_match.group(1) if title_match else d.name

    # Get or generate summary
    summary_file = d / '_summary.md'
    if summary_file.exists():
        summary = summary_file.read_text().strip()
    else:
        try:
            result = subprocess.run(
                ['llm', '-m', MODEL,
                 'Summarize this research project concisely. Write just 1 paragraph '
                 '(3-5 sentences). Vary your opening - do not start with "This report" '
                 'or "This research". Include 1-2 links to key tools/projects if relevant. '
                 'Be specific but brief. No emoji.\n\n' + content],
                capture_output=True, text=True, timeout=60
            )
            summary = result.stdout.strip()
            if summary:
                summary_file.write_text(summary + '\n')
        except Exception:
            summary = ''

    date_str = dt.strftime('%Y-%m-%d')
    cog.outl(f'## [{title}]({d.name}/README.md)')
    cog.outl(f'*{date_str}*')
    cog.outl('')
    if summary:
        cog.outl(summary)
    cog.outl('')
]]]-->
<!--[[[end]]]-->
