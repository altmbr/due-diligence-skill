# /due-diligence — Due Diligence Orchestrator for Claude Code

A Claude Code skill that runs structured due diligence on a company or opportunity — a job offer, an investment, a partnership, or just a company you want to understand. It launches parallel research agents, builds a single master overview, tracks what's known vs. unknown, and ends with a recommendation and conversation prep.

**One command. A decision-grade master overview, not a pile of notes.**

## What It Does

Type `/due-diligence [company]` (optionally with a role, e.g. `/due-diligence Acme CFO`) and the skill will:

1. **Scope** — asks one question: job, investment, partnership, or general? Any files to read first? Detects public companies and resolves ticker + CIK
2. **Ingest existing data** — reads your transcripts, data-room files and prior research so nothing is re-researched
3. **Scan social signal** (optional) — last 30 days of Reddit, X, YouTube, HN and Polymarket on the company, CEO and industry
4. **Pre-fetch SEC data** (public companies) — EDGAR filings and XBRL financials, PDFs pre-parsed so agents can use them
5. **Build a master overview skeleton**, then **launch parallel agents** with workstreams tuned to the DD type
6. **Monitor** — kills and relaunches stuck agents
7. **Synthesize** into the master overview, with targeted follow-up social scans for gaps
8. **Stress-test** — Hamilton Helmer's 7 Powers, plus a Buffett vs. Thiel investor-lens analysis
9. **Prep the conversation** — questions to ask, questions they'll ask you, a one-page brief
10. **Export a PDF** with a cover page and clickable table of contents

## Output Structure

```
~/Documents/[Company]/
├── [COMPANY]-MASTER-OVERVIEW.md     # The deliverable
├── [COMPANY]-MASTER-OVERVIEW.pdf
└── research/
    ├── [agent-name].md               # One file per research workstream
    ├── sec-prefetch.md               # Public companies only
    ├── social_*.md                   # Social signal scans
    ├── 7-powers-analysis.md
    └── investor-lens.md
```

The master overview follows a fixed template (included at the end of `SKILL.md`): executive summary, team, financial analysis (business anatomy, margins, cash generation, balance sheet, outlook), competitive landscape, GTM, product, risk register, valuation or compensation analysis, **Go / No-Go**, and a ranked list of open questions.

## Install

```bash
git clone https://github.com/altmbr/due-diligence-skill.git
cp due-diligence-skill/SKILL.md ~/.claude/commands/due-diligence.md
```

Then type `/due-diligence` in Claude Code.

## Key Design Decisions

- **Ingest before researching** — existing transcripts and notes come first; agents only chase the gaps.
- **Claims vs. evidence** — company marketing and actual data are kept separate. Every data point is dated and sourced inline.
- **SEC filings are the gold standard** for public companies, pre-fetched so agents start with audited numbers.
- **Go / No-Go must recommend** — "it depends" isn't allowed. State the call, the assumptions, and what would change it.
- **Competitors carry their funding** — every competitor table includes cumulative funding raised, sorted descending.
- **Interview prep is one page** — specific enough that a vague answer is itself a red flag.

## Requirements

- [Claude Code](https://claude.com/claude-code) (CLI)
- [`/research`](https://github.com/altmbr/claude-research-skill) — this skill reuses its agent write protocol and monitoring rules
- Optional: [last30days](https://github.com/mvanhorn/last30days-skill) for social signal scans (skipped if absent)
- Optional: `PyMuPDF` for parsing SEC PDFs; `weasyprint` + `markdown` for the PDF export

## License

MIT — see [LICENSE](LICENSE).

## Author

**Bryan Altman** — [GitHub](https://github.com/altmbr)
