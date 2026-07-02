# literature-search-arxiv

Claude Code Skill for searching and retrieving scientific papers from arXiv.

## Features

- Search arXiv with advanced query syntax (author, title, abstract, category, date range) and get clean JSON metadata
- Download full-text papers as PDF or HTML
- Download LaTeX source archives (tar.gz)
- Built-in rate limiting (1 request / 3 seconds) to respect arXiv's Terms of Use

## Contents

```
literature-search-arxiv/
├── SKILL.md                        # skill prompt file
├── references/
│   └── query_syntax.md             # advanced arXiv query syntax reference
└── scripts/
    ├── search_arxiv.py             # search and extract metadata (JSON)
    ├── download_paper.py           # download PDF/HTML full text
    └── download_paper_source.py    # download LaTeX source tar.gz
```

## Requirements

- [`uv`](https://docs.astral.sh/uv/) on PATH — scripts run via `uv run`

## Usage

Copy the whole directory to your personal or project skills directory:

```bash
cp -r . ~/.claude/skills/literature-search-arxiv
```

Then invoke in Claude Code:

```
/literature-search-arxiv find recent papers on diffusion policy for robot manipulation
```

Note: on first use the skill notifies you about arXiv's API terms
(https://info.arxiv.org/help/api/index.html) and records it in
`LICENSE_NOTIFICATION.txt` inside the skill directory. That file is
runtime-generated and intentionally not tracked here. Always check each
paper's license for restrictions.
