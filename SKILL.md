# /due-diligence — Due Diligence Orchestrator

Run a structured due diligence process on a company or opportunity. Launches parallel research agents, builds a master overview, tracks what's known vs. unknown, and generates interview preparation.

Builds on the `/research` skill's agent management and write protocols.

## Arguments
- `<company-name>` — The company or opportunity to research
- Optional: `<role>` — If evaluating a specific role (e.g., "CFO", "CBO", "advisor")

## Instructions

### Phase 1: Scope the DD

Ask the user (one prompt, not multiple):

```
DD: [Company Name]

What type of due diligence?
1. Job opportunity (evaluating a role)
2. Investment (evaluating equity/returns)
3. Partnership/consulting engagement
4. General (just want to understand the company)

Any data room files or transcripts I should read first?
[Paste paths or say "none"]
```

Based on the answer, adjust the agent workstreams:
- **Job opportunity:** Emphasize team/culture, compensation/equity, role scope, CEO profile, growth trajectory
- **Investment:** Emphasize financials, unit economics, competitive moat, TAM, capital structure
- **Partnership:** Emphasize strategic fit, revenue potential, team quality, market position
- **General:** Balanced across all dimensions

**Public company detection:**

After scoping, determine if the company is publicly traded:
1. Use WebSearch to check if the company has a ticker symbol / is listed on a stock exchange
2. If public (US-listed): resolve the ticker and CIK number using `https://www.sec.gov/files/company_tickers.json`
3. If public (foreign private issuer listed in US): note it files 20-F instead of 10-K
4. If private: skip all SEC-related phases — use Crunchbase, PitchBook, news, and LinkedIn instead

Store the result as a variable for downstream phases:
```
PUBLIC_COMPANY: true/false
TICKER: [ticker]
CIK: [10-digit CIK]
FILER_TYPE: "domestic" | "fpi"  (foreign private issuer)
FISCAL_YEAR_END: [month]
```

### Phase 2: Ingest Existing Data

If the user provides file paths:

1. Read all provided files (transcripts, PDFs, financial models, notes)
2. Extract key facts into a `known-facts.md` scratchpad
3. Identify gaps — what do we NOT know yet that matters?

If a project directory already exists (e.g., `~/Documents/[Company]/`):
1. Scan for existing research, transcripts, and overview documents
2. Read them to avoid re-researching known ground
3. Note what's already covered vs. what needs updating

### Phase 2.5: Social Signal Scan

Run the last30days script to capture real-time social sentiment about the company, its leadership, and its market. This surfaces what employees, customers, investors, and the tech community are **actually saying** — often more honest than press releases.

**Step 1: Resolve X handles**

Use WebSearch to find X/Twitter handles for:
- The company itself (e.g., @aeaborobotics)
- The CEO / key leadership (e.g., @ceo_handle)

**Step 2: Run social scans (up to 3 queries)**

```bash
# Find last30days skill root
for dir in \
  "${CLAUDE_PLUGIN_ROOT:-}" \
  "$HOME/.claude/skills/last30days" \
  "$HOME/.agents/skills/last30days" \
  "$HOME/.codex/skills/last30days"; do
  [ -n "$dir" ] && [ -f "$dir/scripts/last30days.py" ] && SKILL_ROOT="$dir" && break
done

if [ -n "${SKILL_ROOT:-}" ]; then
  export PATH="$HOME/Library/Python/3.9/bin:$PATH"

  # Scan 1: Company overview — what people are saying about the company
  python3 "${SKILL_ROOT}/scripts/last30days.py" "{COMPANY_NAME}" --emit=compact --include-web \
    > ~/Documents/{Company}/research/social_company.md 2>&1

  # Scan 2: CEO / key leader — reputation, public statements, discourse
  python3 "${SKILL_ROOT}/scripts/last30days.py" "{CEO_NAME} {COMPANY_NAME}" --emit=compact --include-web \
    > ~/Documents/{Company}/research/social_ceo.md 2>&1

  # Scan 3 (optional): Industry/competitive context
  python3 "${SKILL_ROOT}/scripts/last30days.py" "{INDUSTRY} market news" --emit=compact --include-web \
    > ~/Documents/{Company}/research/social_industry.md 2>&1
fi
```

Use a **timeout of 300000** (5 minutes) per scan. Run sequentially (each depends on the same script).

**Step 3: Extract key signals**

From the scan output, note:
- **Employee sentiment** — Reddit threads from r/[company], Glassdoor mentions, complaints or praise
- **Customer/user sentiment** — Product feedback, complaints, enthusiasm on Reddit/X
- **Investor/market sentiment** — X posts from analysts, VCs, or industry watchers
- **CEO reputation** — How the CEO is discussed publicly, any controversies, respect level
- **Polymarket odds** — Any prediction markets related to the company, industry, or key outcomes (these are high-signal — real money on the line)
- **Red flags** — Layoff threads, lawsuit mentions, product failures, leadership departures

**Step 4: Extract hypotheses and investigation directives**

Do NOT just pass file paths to agents. The orchestrator must read the social scan outputs and extract **specific, actionable hypotheses** that agents investigate deeply.

**4a. Read all social scan files and extract:**
- **Quantitative claims** that need verification (e.g., "acquisitions up 300%" — is this accurate per SEC filings?)
- **Sentiment patterns** — what's the dominant narrative? Is it retail hype, institutional skepticism, or mixed?
- **Unverified signals** — claims that appeared in social media but aren't confirmed (e.g., crypto payment acceptance, specific product launches)
- **Named sources** — specific analysts, insiders, or domain experts whose claims carry weight
- **Polymarket odds** — prediction market data with real money behind it
- **YouTube video IDs** — note any earnings call clips, CEO interviews, or analyst breakdowns for potential deep-dive

**4b. Generate per-agent investigation directives:**

For each agent, write 2-4 specific hypotheses derived from social data that the agent should investigate. These go in the agent prompt ALONGSIDE the file paths.

Example format:
```
SOCIAL SIGNAL DIRECTIVES (investigate these specifically):

1. VERIFY: Social media claims acquisitions up 300% since September — cross-reference with 10-Q/earnings data
2. INVESTIGATE: Multiple X posts reference "Cash Plus surge from 19% to [higher]" — find exact current percentage and trajectory
3. PROBE: @nickgerli1 reports housing demand index at 5/100 in mid-Feb 2026 — validate this metric and assess what it means for the company's near-term pipeline
4. FLAG: Unverified claim that company will accept Bitcoin for home purchases — determine if this is real or rumor

Pre-gathered social signal data (read FIRST, then investigate above):
- ~/Documents/{Company}/research/social_company.md
- ~/Documents/{Company}/research/social_ceo.md
- ~/Documents/{Company}/research/social_industry.md
```

**4c. Route signals to the right agents:**

- Employee/culture signals + specific Glassdoor/Reddit threads → **Culture & Team** agent
- CEO/leadership signals + specific X posts/interviews → **CEO & Leadership** / **Team & Execution** agent
- Customer/product signals + user experience Reddit threads → **Company & Market** / **Business Model** agent
- Investor/competitive signals + analyst posts → **Competitive Landscape** / **Competitive Moat** agent
- Polymarket odds + housing/macro predictions → **Risks & Scenarios** agent (for investments) or **Role & Equity** agent (for jobs)
- Quantitative claims needing verification → **Financial Deep Dive** / **Financial Health** agent

**If the script is NOT available** (missing skill or no API keys), skip this phase and proceed. Social data enriches DD but isn't required.

### Phase 2.75: SEC Filing Pre-Fetch (Public Companies Only)

**Skip this phase entirely if the company is private.**

For public companies, pre-fetch key SEC filing data before launching agents. This follows the `/sec` skill's playbook and the PDF pre-parsing protocol — agents cannot download or parse files themselves, so the orchestrator must do it.

**Step 1: Pull structured EDGAR data**

```bash
# Get filing history
curl -s -H "User-Agent: research@example.com" \
  "https://data.sec.gov/submissions/CIK{cik_10digit}.json" \
  > /tmp/{company}-submissions.json

# Get key financial metrics via XBRL (consolidated)
# Revenue
curl -s -H "User-Agent: research@example.com" \
  "https://data.sec.gov/api/xbrl/companyconcept/CIK{cik}/us-gaap/RevenueFromContractWithCustomerExcludingAssessedTax.json" \
  > /tmp/{company}-revenue.json

# Net income
curl -s -H "User-Agent: research@example.com" \
  "https://data.sec.gov/api/xbrl/companyconcept/CIK{cik}/us-gaap/NetIncomeLoss.json" \
  > /tmp/{company}-netincome.json

# Total assets
curl -s -H "User-Agent: research@example.com" \
  "https://data.sec.gov/api/xbrl/companyconcept/CIK{cik}/us-gaap/Assets.json" \
  > /tmp/{company}-assets.json
```

**Rate limit:** 10 requests/second max. Add `sleep 0.2` between calls.

**Step 2: Identify and fetch key filings**

From the submissions JSON, identify:
- Most recent 10-K (or 20-F for FPIs) — URL and filing date
- Most recent 10-Q (or 6-K for FPIs) — URL and filing date
- Most recent DEF 14A (proxy) — URL and filing date
- Any 8-Ks from the last 90 days — URLs and descriptions

Use WebSearch to find the HTML filing URLs on EDGAR, then WebFetch the **specific sections** needed (don't read entire filings):

| Section | Filing | What to extract |
|---------|--------|----------------|
| Item 1 (Business) | 10-K | Business overview, segments, customers, competition |
| Item 1A (Risk Factors) | 10-K | Top 10-15 most specific/unusual risks (skip boilerplate) |
| Item 7 (MD&A) | 10-K | Revenue drivers, segment performance, margin commentary |
| Revenue disaggregation note | 10-K | Granular revenue breakdown |
| Segment information note | 10-K | Revenue/profit by segment |
| Debt note | 10-K | Outstanding debt, maturities, covenants |
| Exec comp summary | DEF 14A | CEO/C-suite total comp |

**Step 3: Pre-parse any PDFs**

If key data is in PDF format (earnings releases, investor presentations):

```bash
mkdir -p /tmp/parsed-{company}/

# Download
curl -s -o /tmp/parsed-{company}/{name}.pdf -L "{URL}"

# Parse with PyMuPDF
python3 -c "
import pymupdf
doc = pymupdf.open('/tmp/parsed-{company}/{name}.pdf')
with open('/tmp/parsed-{company}/{name}.txt', 'w') as f:
    for i, page in enumerate(doc):
        f.write(f'=== PAGE {i+1} ===\n')
        f.write(page.get_text())
        f.write('\n---PAGE BREAK---\n')
"
```

**Step 4: Save extracted data**

Write a summary file at `~/Documents/{Company}/research/sec-prefetch.md` containing:
- Company identifiers (ticker, CIK, filer type, fiscal year-end)
- Key financial metrics extracted from XBRL (revenue, net income, assets — last 3-5 years)
- Filing URLs for each key filing identified
- Extracted section summaries from Step 2
- Paths to any pre-parsed PDFs

This file becomes the SEC data foundation that agents build on.

**For foreign private issuers (20-F filers):**
- Use 20-F instead of 10-K, 6-K instead of 10-Q
- Note the accounting standard (IFRS vs. US GAAP vs. home-country GAAP)
- Check if quarterly data is available (not required for FPIs)
- Currency will be in home currency — note the conversion rate

**If EDGAR API calls fail or return no data**, fall back to WebSearch for the filing on EDGAR and WebFetch the HTML version directly. SEC data enriches DD significantly for public companies but should not block the process.

### Phase 3: Create Master Overview Skeleton

1. Create the project directory if it doesn't exist: `~/Documents/[Company]/`
2. Create `~/Documents/[Company]/research/` for agent outputs
3. Create the master overview at `~/Documents/[Company]/[COMPANY]-MASTER-OVERVIEW.md` using the Master Overview Template (see Appendix at the end of this file)
   - **Metadata lines** (Status, Last updated, DD Type, Decision deadline) must use `<br>` at line ends to force line breaks in PDF output
4. **Section 3 must include sub-section headers Sections 3.1–3.6:**
   - `### 3.1 Revenue Overview` — `[Pending — populated during synthesis]`
   - `### 3.2 Business Anatomy` — `[Pending — populated during synthesis]`
   - `### 3.3 Margin Architecture` — `[Pending — populated during synthesis]`
   - `### 3.4 Cash Generation & Capital Allocation` — `[Pending — populated during synthesis]`
   - `### 3.5 Balance Sheet` — `[Pending — populated during synthesis]`
   - `### 3.6 Forward Outlook` — `[Pending — populated during synthesis]`
5. Pre-fill any sections with facts already known from Phase 2
6. Mark pre-filled sections with source attribution

### Phase 4: Plan Research Agents

Design 4-6 agent workstreams based on DD type. Standard workstreams:

#### For Job Opportunities:
| Agent | Scope | Key Questions |
|-------|-------|---------------|
| **CEO & Leadership** | CEO background, leadership style, reputation, track record | Who is this person? What's their playbook? What do former reports say? |
| **Company & Market** | Business model, market position, TAM, traction, trajectory | Is this company winning? What's the growth story? |
| **Financial Health** | Revenue, burn rate, funding history, unit economics, runway. **If public:** 10-K financials (Item 7 MD&A + Item 8), XBRL data, segment economics, debt/covenants. | Can they afford this role? Are the economics sound? |
| **Competitive Landscape** | Direct competitors, positioning, moats, threats; **cumulative funding raised per competitor** | Who else plays here? What's the defensibility? Who is best capitalized? |
| **Culture & Team** | Glassdoor, LinkedIn team composition, turnover signals, org structure | What's it actually like to work here? |
| **Role & Equity** | Comparable roles/comp at similar companies, equity valuation scenarios | Is the offer fair? What's the expected value? |

#### For Investments:
| Agent | Scope | Key Questions |
|-------|-------|---------------|
| **Market & TAM** | Market size, growth rate, secular trends, timing | How big can this get? |
| **Business Model (QUALITATIVE)** | How each product/segment works (mechanics, pricing structure, customer behavior). Competitive positioning per product line. Unit economics RATIOS (LTV/CAC, payback, NRR) with methodology. Revenue quality scorecard, flywheel narrative, pricing power evidence. **DOES NOT** include revenue/margin data tables, income statements, or quarterly financial data — that belongs in Financial Deep Dive. | How and why does the business work? |
| **Team & Execution** | Founders, key hires, track record, org gaps | Can this team actually build it? |
| **Competitive Moat** | Competitors, differentiation, switching costs, network effects; **cumulative funding raised per competitor** | Why won't someone else just copy this? Who has the capital to? |
| **Financial Deep Dive (QUANTITATIVE)** | Income statement (annual + quarterly), revenue disaggregation with exact $ figures, **Segment Revenue Detail table** (see below), gross margin by segment, opex breakdown, FCF bridge, SBC, balance sheet, model inputs, forward estimates. **If public:** 10-K Item 7 (MD&A) + Item 8 (financials & notes), XBRL multi-year trends, debt covenants, revenue disaggregation, segment profitability. **DOES NOT** include business model narratives, competitive positioning, or flywheel descriptions — that belongs in Business Model. | What do the numbers actually say? |
| **Risks & Scenarios** | Bear/base/bull cases, key risks, regulatory, technology risk. **If public:** 10-K Item 1A (risk factors), pending litigation (Item 3 + contingencies note), recent 8-Ks for material events. | What kills this? |

**Competitor funding standard (all DD types):** wherever a competitor is named, in agent files and in the master overview, record its **cumulative funding raised to date** (total across all rounds, latest round date and lead, inline source; write "undisclosed" or "bootstrapped/public" rather than omitting). Every competitor table carries a **Cumulative funding** column and is **sorted by it, descending**, unless another order is explicitly more useful (e.g., ranked by threat), in which case the funding column still appears. Include this instruction verbatim in every competitor-facing agent prompt.

Present the plan to the user:
```
## DD Plan: [Company]

### Agent Workstreams
[List each agent with scope and output file path]

### Already Known (from existing data)
[Bullet list of key facts already established]

### Key Unknowns
[What we need to find out, ranked by importance]

Launch [N] agents? [Y/N]
```

**Wait for approval before launching.**

### Phase 5: Launch & Monitor

Follow the `/research` skill's Phase 3-4 protocol exactly:

1. Create skeleton files in `~/Documents/[Company]/research/[agent-name].md`
2. Launch ALL agents in parallel with `run_in_background: true`
3. Every agent gets the critical write protocol verbatim from `/research`
4. Monitor on the escalating schedule: 30s, 2min, 5min, then every 5min
5. Kill and relaunch stuck agents immediately (see the monitoring rules in the `/research` skill)

**Additional DD-specific agent instructions:**
- Cross-reference numbers across sources — flag inconsistencies
- Distinguish between company claims and independent verification
- Note the date of every data point — stale data is misleading
- For private companies: Crunchbase, PitchBook references, news, LinkedIn
- **If social signal scan files exist from Phase 2.5, include their paths in each relevant agent's prompt.** Agents should read and build on this data rather than re-searching Reddit/X. Social signals with high engagement (upvotes, likes) should be weighted as strong evidence of public sentiment.

**SCOPE BOUNDARY instructions (include in agent prompts for Investment DD):**

Add the following to the **Business Model** agent prompt:
```
SCOPE BOUNDARY: You are the QUALITATIVE agent. Your job is to explain HOW and WHY the business works.
DO NOT build financial data tables (revenue tables, income statements, quarterly data, margin tables).
DO NOT duplicate quantitative work — the Financial Deep Dive agent handles all numbers.
You MAY reference specific numbers inline to support qualitative points (e.g., "NRR of 120% suggests strong expansion"),
but do not build structured financial exhibits. Focus on: product mechanics, pricing logic, customer behavior,
competitive positioning, unit economics RATIOS with methodology, revenue quality, and flywheel narratives.
```

Add the following to the **Financial Deep Dive** agent prompt:
```
SCOPE BOUNDARY: You are the QUANTITATIVE agent. Your job is to report WHAT THE NUMBERS SAY.
DO NOT write business model narratives, competitive positioning analysis, or flywheel descriptions.
DO NOT duplicate qualitative work — the Business Model agent handles strategic context.
You MAY include brief context to explain a number (e.g., "Gross margin expanded due to mix shift toward subscriptions"),
but do not build strategic narratives. Focus on: income statements, revenue disaggregation, segment revenue detail,
margin analysis, opex breakdown, FCF bridge, SBC, balance sheet, model inputs, and forward estimates.
```

**For Investment DD — Financial Model Data Collection:**

When the DD type is **Investment** and the company is **public**, one agent (typically the **Financial Deep Dive** agent) must be explicitly tasked with collecting enough structured quantitative data to build a simplified financial model. This is in addition to its normal analysis work.

Include this directive in the Financial Deep Dive agent prompt:

```
MODEL DATA COLLECTION (required for Investment DD):

In addition to your analysis, collect and structure the following data in a dedicated
section called "## Model Inputs" at the END of your output file. Format as markdown tables.

Required data (quarterly, last 8 quarters minimum):
1. Revenue per quarter
2. Cost of revenue / COGS per quarter
3. Gross profit and gross margin % per quarter
4. Key volume metric per quarter (e.g., homes sold, subscribers, orders — whatever drives revenue)
5. Average revenue per unit (revenue / volume metric)
6. Contribution profit per unit (if disclosed or calculable)
7. SG&A per quarter (separate fixed vs. variable if possible)
8. Operating income/loss per quarter
9. Adjusted EBITDA per quarter (if reported)
10. Free cash flow per quarter

Required data (annual, last 3-5 years):
11. Revenue
12. Gross margin %
13. Operating margin %
14. Net income/loss
15. Total assets, total debt, cash & equivalents
16. Shares outstanding (basic + diluted)
17. Capital expenditures

Required data (forward-looking, if available):
18. Management guidance or targets (revenue, margins, profitability timelines)
19. Analyst consensus estimates (revenue, EPS) for next 2 years
20. Key assumptions underlying management targets

Sources: 10-K Item 7/8, 10-Qs, earnings releases, earnings call transcripts, analyst reports.
Cite the source and date for every number.

Format example:
| Quarter | Revenue | COGS | Gross Margin | Units Sold | Rev/Unit | SG&A | Op Income | Adj EBITDA |
|---------|---------|------|-------------|------------|----------|------|-----------|------------|
| Q1 2024 | $X | $X | X% | X | $X | $X | ($X) | ($X) |
```

This structured data allows the orchestrator to later build or validate a simplified financial model without re-researching every number. If quarterly data isn't available in SEC filings (e.g., some FPIs only report semi-annually), collect the finest granularity available and note the limitation.

**Segment Revenue Detail (required for ALL Investment DD):**

In addition to model inputs, the Financial Deep Dive agent must produce a **Segment Revenue Detail** table decomposing every identifiable revenue line. Include this directive in the Financial Deep Dive agent prompt:

```
SEGMENT REVENUE DETAIL (required):

Build a table decomposing every identifiable revenue line for the most recent fiscal year.
Include SEC-reported segments PLUS sub-components estimated from 10-K notes, earnings calls,
and analyst reports. This table becomes the raw material for the master overview's Business Anatomy exhibit.

| Revenue Line | FY Revenue ($) | % of Total | YoY Growth | Gross Margin (est.) | Data Quality |
|-------------|---------------|-----------|-----------|-------------------|-------------|
| [Segment A] | $X.XB | XX% | +XX% | ~XX% | Reported |
|   └ Sub-product 1 | $X.XB | XX% | +XX% | ~XX% | Estimated |
|   └ Sub-product 2 | $X.XB | XX% | +XX% | ~XX% | Derived |
| [Segment B] | $X.XB | XX% | +XX% | ~XX% | Reported |
| **Total** | **$X.XB** | **100%** | **+XX%** | **~XX%** | — |

Data Quality flags:
- **Reported** = directly from SEC filings (10-K segment note, revenue disaggregation)
- **Estimated** = derived from earnings call commentary, investor presentations, or analyst reports
- **Derived** = calculated (e.g., total minus known segments)

IMPORTANT: Total MUST reconcile to reported total revenue. If sub-products don't sum to the segment,
show the gap as "Other / Unallocated."
```

**For public companies — SEC filing integration (Phase 2.75 data):**

If the company is public and SEC data was pre-fetched, include the following in **every relevant agent's prompt**:

```
SEC FILING DATA (PUBLIC COMPANY)

Ticker: {TICKER} | CIK: {CIK} | Filer type: {domestic/fpi} | FY ends: {month}

Pre-fetched SEC data is available at:
- ~/Documents/{Company}/research/sec-prefetch.md — XBRL financials, filing URLs, section summaries
- /tmp/parsed-{company}/*.txt — Pre-parsed PDF earnings releases/presentations (if any)

Read sec-prefetch.md FIRST before searching. It contains multi-year revenue, net income,
and asset data plus extracted summaries of key 10-K sections.

SEC FILING REFERENCE (use when you need to go deeper):

Filing type decision logic:
- Business overview / "what do they do" → 10-K Item 1
- Revenue by segment or product → 10-K Item 7 + revenue disaggregation note
- Risk factors → 10-K Item 1A
- Recent quarter performance → Latest 10-Q
- Executive pay → DEF 14A
- Board composition → DEF 14A
- What changed recently → Latest 8-Ks (last 90 days)
- Customer concentration → 10-K Item 1 + revenue disaggregation note
- Debt / capital structure → 10-K Item 8, debt note
- Pending litigation → 10-K Item 3 + commitments/contingencies note

Key 10-K sections:
- Item 1: Business description, segments, customers, competition
- Item 1A: Risk factors (20-50 pages — look for specific/unusual risks, skip boilerplate)
- Item 7: MD&A — THE GOLDMINE for financial analysis. Segment performance, margin commentary, trends.
- Item 8: Financial statements + Notes (revenue disaggregation, segments, debt, leases, litigation)

IMPORTANT:
- Fiscal year ≠ calendar year. Always check when FY ends.
- Segments change across years. Don't assume consistency.
- Distinguish GAAP vs. non-GAAP/pro forma numbers.
- Items 10-14 often say "incorporated by reference" — go to the DEF 14A.
- Cite precisely: "Source: {Company} 10-K FY{year}, Item 7, page X"
- Do NOT try to download or parse PDFs yourself. Use pre-parsed files only.
```

Route SEC knowledge to the right agents:
- **Financial Health / Financial Deep Dive** → Full SEC reference above + XBRL data + all pre-parsed financials
- **CEO & Leadership / Team & Execution** → DEF 14A references (exec comp, board composition)
- **Competitive Landscape / Competitive Moat** → 10-K Item 1 (competition section) + Item 1A (competitive risk factors)
- **Risks & Scenarios** → 10-K Item 1A (risk factors) + Item 3 (legal proceedings) + recent 8-Ks
- **Company & Market / Market & TAM** → 10-K Item 1 (business/market description) + Item 7 (MD&A trends)
- **Role & Equity** → DEF 14A (exec comp tables, equity awards) + proxy statement

### Phase 6: Synthesize into Master Overview

Once all agents are complete (or user decides to proceed):

1. Read all agent output files
2. **Build Section 3 (Financial Analysis) using the protocol below**
3. Update all other master overview sections:
   - Merge agent findings into the appropriate sections
   - Resolve contradictions (flag any that can't be resolved)
   - Add source citations inline
4. Complete the **Go / No-Go Assessment** section with:
   - Explicit recommendation (Go / No-Go / Need More Data)
   - Top 3 reasons for the recommendation
   - Top 3 risks if proceeding
   - What would change the recommendation
   - **If Polymarket data was gathered in Phase 2.5**, cite relevant prediction market odds as supporting evidence (e.g., "Polymarket gives X a Y% chance of Z, which supports/contradicts the assessment"). Real money on outcomes cuts through opinion.

   **VALUATION CONSISTENCY CHECK (required before writing Go/No-Go):**

   Before writing the recommendation and reasoning, pull the current multiples from Section 8 and ask:
   - At [X]x P/FCF and [Y]x P/S, what is the market ALREADY pricing in? (sustained growth? deceleration? perfection?)
   - Do any of my "Top 3 Reasons" claim the stock is undervalued, cheap, or "priced for disappointment"? If so, does that claim survive scrutiny against the actual multiples?
   - If the multiples imply the market expects 25%+ growth for years, you CANNOT say the stock is "priced for disappointment." The stock is priced for continued excellence. Say so.
   - The honest framing for a high-multiple stock with strong execution is: "you're paying a premium for [specific optionality/quality], and the bet is that [specific catalyst] justifies the price." Not: "the market is underpricing this."
   - If a bear case shows only -10% to -28% downside, question the bear assumptions. A real bear case for a stock at 15x revenue should model what happens at 6-8x — recession, multiple compression, and growth deceleration simultaneously.
   - Cross-reference with the Investor Lens: if Buffett says "no margin of safety," the Go/No-Go cannot claim the stock is cheap.

5. Complete the **Open Questions & Research Backlog** section:
   - Rank unknowns by importance
   - For each unknown, suggest how to get the answer (e.g., "Ask CEO directly," "Request data room access," "Check SEC filing")

#### Section 3 Synthesis Protocol

Build each sub-section by merging the right agent's data. This eliminates duplication and creates narrative flow.

| Sub-section | Primary Source | Enrichment Source | What to Build |
|-------------|---------------|-------------------|---------------|
| **Section 3.1 Revenue Overview** | Financial Deep Dive (income statement tables) | — | 5-year revenue summary table + one paragraph of context |
| **Section 3.2 Business Anatomy** | Financial Deep Dive (Segment Revenue Detail table — the numbers) | Business Model (what each product does, what's driving it — the qualitative context) | THE key exhibit: merge the quantitative segment table with 2-3 qualitative bullets per major segment. Also include: revenue quality scorecard, customer cohort structure ("wedding cake" table), asset tag / margin-accretive product economics if applicable |
| **Section 3.3 Margin Architecture** | Financial Deep Dive (gross margins by segment, opex breakdown) | Business Model (WHY margins move — mix shift, pricing power, scale) | Gross margin by segment + blended trend. Opex table (R&D/S&M/G&A as % of revenue). Key efficiency metrics: LTV/CAC, magic number, CAC payback, RPO, ARR/employee, NRR by cohort, gross retention estimate. Operating leverage verdict. |
| **Section 3.4 Cash Generation & Capital Allocation** | Financial Deep Dive | — | FCF bridge, SBC / "true owner earnings", capital allocation priorities |
| **Section 3.5 Balance Sheet** | Financial Deep Dive | — | Summary table, fortress-or-fragile verdict |
| **Section 3.6 Forward Outlook** | Financial Deep Dive (guidance, consensus) | Business Model (strategic context for assumptions) | Management guidance, analyst consensus, key assumptions, catalysts |

#### Required Detailed Exhibits by Section

The master overview is the single deliverable the user reads. Don't force them to dig into agent files for key analysis. Each exhibit below must be included in full, not summarized to a single line.

**Section 3.2 Business Anatomy:**
- Business Anatomy table (merged from Financial Deep Dive numbers + Business Model context)
- Revenue quality scorecard (table with factor/score/notes)
- Customer cohort structure table ("wedding cake") with count, ARR share, growth rate per tier
- Hardware-software flywheel narrative if applicable (why physical moat compounds)

**Section 3.3 Margin Architecture:**
- Key efficiency metrics table: LTV/CAC, magic number, CAC payback, RPO, ARR/employee, NRR by cohort, gross retention estimate
- Opex breakdown table with 3-year trends

**Section 4 Competitive Landscape:**
- Competitor map table(s): every named competitor with a **Cumulative funding raised** column (inline source), sorted by funding descending unless ranked by threat
- TAM segment breakdown table: segment, estimated TAM, company's current ARR in that segment, notes
- Third-party market estimates table: source, segment, current value, projection, CAGR
- Full 7 Powers table: all 7 powers with Status and multi-sentence Evidence column (not one-line summaries)
- Durability verdict with the key variable that determines outcome

**Section 8 Valuation Analysis (Investment DD):**
- Bear/Base/Bull case narratives: 2-3 sentence narrative for each scenario explaining the reasoning, not just the price target
- Valuation sensitivity matrix (revenue × multiple) showing price at different assumptions
- Scenario summary table with probability, price target, return from current price, and probability-weighted return

**Section 9 Go / No-Go Assessment:**
- Full Investor Lens: Buffett (moat, management, economics, margin of safety — each 2-4 sentences) + Thiel (monopoly test, secret, definite vision, power law potential — each 2-4 sentences)
- Agree/Disagree comparison table
- Synthesis paragraph

### Phase 6.25: Targeted Follow-Up Social Scans

After synthesis, the open questions and unresolved signals from Phase 6 should drive a second round of targeted social scans. This is NOT a repeat of Phase 2.5 — it's a focused deep-dive on specific gaps identified during research.

**Step 1: Identify scan targets from Open Questions**

Review the Open Questions & Research Backlog from Phase 6. For each HIGH or CRITICAL unknown, determine if social/web sources could help resolve it. Prioritize:
- CEO interviews and earnings call clips on YouTube (often contain forward guidance not in filings)
- Analyst deep-dives on specific product lines or unit economics
- Reddit/forum threads with insider or customer perspectives on specific features
- Industry expert commentary on the company's strategy

**Step 2: Run 2-4 targeted scans**

```bash
# Find last30days skill root
for dir in \
  "${CLAUDE_PLUGIN_ROOT:-}" \
  "$HOME/.claude/skills/last30days" \
  "$HOME/.agents/skills/last30days" \
  "$HOME/.codex/skills/last30days"; do
  [ -n "$dir" ] && [ -f "$dir/scripts/last30days.py" ] && SKILL_ROOT="$dir" && break
done

if [ -n "${SKILL_ROOT:-}" ]; then
  export PATH="$HOME/Library/Python/3.9/bin:$PATH"

  # Scan 4: Targeted at top open question (e.g., specific product economics)
  python3 "${SKILL_ROOT}/scripts/last30days.py" "{TARGETED_QUERY_1}" --emit=compact --include-web \
    > ~/Documents/{Company}/research/social_followup_1.md 2>&1

  # Scan 5: CEO/leadership interviews (YouTube-heavy)
  python3 "${SKILL_ROOT}/scripts/last30days.py" "{CEO_NAME} interview {COMPANY}" --emit=compact --include-web \
    > ~/Documents/{Company}/research/social_followup_2.md 2>&1

  # Scan 6 (optional): Specific analyst or competitive angle
  python3 "${SKILL_ROOT}/scripts/last30days.py" "{TARGETED_QUERY_3}" --emit=compact --include-web \
    > ~/Documents/{Company}/research/social_followup_3.md 2>&1
fi
```

Use a **timeout of 300000** (5 minutes) per scan. Run sequentially.

**Step 3: Integrate findings**

Read the follow-up scan outputs and:
1. **Resolve open questions** — update the master overview's Open Questions section. Mark any that are now answered.
2. **Surface new evidence** — add any new data points to the relevant master overview sections with `[Source: social follow-up scan]` attribution.
3. **Flag YouTube videos worth watching** — if CEO interviews or earnings call clips surfaced, list them with timestamps and key quotes in the master overview or a separate `~/Documents/{Company}/research/video-notes.md`.
4. **Update the Go/No-Go** if new evidence materially changes the assessment.

**Query design principles:**
- Be specific. "Opendoor Cash Plus unit economics" > "Opendoor news"
- Target the format. Include "interview" or "earnings call" to surface YouTube content.
- Name the unknown. "{CEO} on profitability timeline" > "{CEO} {Company}"
- Test contrarian angles. If the research is bullish, scan for bear cases and vice versa.

**If the script is NOT available**, skip. The DD is already complete from Phase 6 — this phase improves it.

### Phase 6.5: 7 Powers Analysis

Apply Hamilton Helmer's **7 Powers** framework to assess the company's strategic position. This is not optional — every DD gets a 7 Powers analysis.

For each power, assess: **Present / Emerging / Absent / N/A** — with a one-line evidence statement.

```
## 7 Powers Analysis: [Company]

| Power | Status | Evidence |
|-------|--------|----------|
| **Scale Economies** | [Present/Emerging/Absent] | [Does the company's unit cost decline as volume increases? Fixed cost spread, purchasing leverage, network infrastructure amortization.] |
| **Network Economies** | [Present/Emerging/Absent] | [Does the value to each user increase as more users join? Marketplace dynamics, data network effects, platform lock-in.] |
| **Counter-Positioning** | [Present/Emerging/Absent] | [Has the company adopted a business model that incumbents can't copy without damaging their existing business? What would incumbents have to sacrifice?] |
| **Switching Costs** | [Present/Emerging/Absent] | [What does it cost a customer (time, money, data loss, retraining) to switch to a competitor? Contractual, procedural, or data-based lock-in.] |
| **Branding** | [Present/Emerging/Absent] | [Does the brand command a price premium or drive habitual purchasing? Consumer trust, reputation, emotional attachment.] |
| **Cornered Resource** | [Present/Emerging/Absent] | [Does the company control a valuable resource others can't replicate? Key talent, patents, regulatory licenses, unique data sets, geographic rights.] |
| **Process Power** | [Present/Emerging/Absent] | [Has the company built deeply embedded organizational processes that are hard to copy? Operational excellence that took years to develop.] |

**Overall Power Assessment:**
- **Strongest power(s):** [which power(s) and why]
- **Biggest power gap:** [which absent power matters most and what it means]
- **Durability verdict:** [Can this company sustain above-normal returns for 10+ years? Why or why not?]
```

**How to assess each power:**
- Use agent research outputs as evidence — don't re-research
- For **Scale Economies**: look at gross margin trends, R&D as % of revenue over time, unit economics at different scales
- For **Network Economies**: look at user/customer growth flywheel, marketplace liquidity, data moat
- For **Counter-Positioning**: identify what incumbents would lose by copying this model
- For **Switching Costs**: look at contract lengths, integration depth, data portability, retraining cost
- For **Branding**: look at pricing power, NPS/customer loyalty signals, premium vs. commodity positioning
- For **Cornered Resource**: look at patents, exclusive partnerships, regulatory moats, proprietary data
- For **Process Power**: look at operational metrics vs. competitors, institutional knowledge, manufacturing/delivery excellence

Write the analysis to `~/Documents/[Company]/research/7-powers-analysis.md` and merge the **full 7 Powers table with evidence** into the master overview's **Competitive Landscape** section. Include the complete Status + Evidence for each power, plus the Overall Power Assessment (strongest powers, biggest gap, durability verdict). Do NOT reduce to a one-line summary — the full table belongs in the overview.

### Phase 6.75: Investor Lens Analysis

Stress-test the opportunity through the lens of two legendary investors with fundamentally different philosophies. This surfaces blind spots and forces intellectual honesty.

Write this to `~/Documents/[Company]/research/investor-lens.md` and merge the **full Investor Lens analysis** into the master overview's **Go / No-Go Assessment** section. Include:
- Buffett's full assessment: moat, management, economics, margin of safety (2-4 sentences each), verdict + quote
- Thiel's full assessment: monopoly test, secret, definite vision, power law potential (2-4 sentences each), verdict + quote
- The Agree/Disagree comparison table
- The synthesis paragraph
Do NOT reduce to a two-row summary table — the full analysis belongs in the overview.

```
## Investor Lens: [Company]

---

### Warren Buffett's Take

Evaluate as Buffett would — patient capital, durable competitive advantage, management quality, margin of safety.

**Moat assessment:**
[Is there a durable competitive advantage? What type — cost, brand, network, regulatory? How wide and how durable? Would Buffett call this a "wonderful business" or a "cigar butt"?]

**Management quality:**
[Are the operators honest, capable, and owner-oriented? Do they allocate capital well? Do they under-promise and over-deliver? Any red flags in compensation structure — are they aligned with shareholders or enriching themselves?]

**Economics:**
[Is this a business a child could run? Predictable cash flows? High ROIC? Low capital intensity? Or does it require constant reinvention and heavy capex? What are the owner earnings (net income + depreciation - maintenance capex)?]

**Circle of competence:**
[Is this business understandable? Can you explain how it makes money in one sentence? If it requires deep technical expertise to evaluate, flag that as a risk.]

**Margin of safety:**
[At the current valuation, what's the downside? What would Buffett pay vs. what's being asked? How much room for error?]

**Buffett's verdict:** [BUY / PASS / WATCHLIST]
**In his words:** "[One sentence channeling Buffett's folksy directness — e.g., 'I'd rather buy a wonderful business at a fair price than a fair business at a wonderful price, and this is...' ]"

---

### Peter Thiel's Take

Evaluate as Thiel would — monopoly potential, secrets, definite optimism, zero-to-one thinking.

**Monopoly test:**
[Is this company building a monopoly or competing? What's the path from small market dominance to large market dominance? Are they a "last mover" in their category? In 10 years, will this be one of a handful of companies that matter in its space — or one of dozens?]

**Secret:**
[What important truth does this company know that others don't? What contrarian bet are they making? If everyone agrees this is a good idea, it's probably not — what's the non-obvious insight?]

**Definite vs. indefinite:**
[Does the team have a definite plan to build a specific future? Or are they hedging, A/B testing, and "iterating"? Thiel favors bold, specific visions over incremental optimization.]

**Technology leverage:**
[Is this a 10x improvement over alternatives, or a marginal one? Thiel's bar: technology should be so good that competition is irrelevant, not just slightly better.]

**Distribution:**
[Can they sell it? Thiel emphasizes that even the best product fails without distribution. What's the go-to-market? Is there a scalable, repeatable sales motion? Or are they hoping the product sells itself (it won't)?]

**Power law:**
[Could this be a power-law outcome — one of a tiny number of investments that returns the entire fund? Or is this a "nice business" with linear returns? Thiel only cares about the former.]

**Thiel's verdict:** [FUND / PASS / INTERESTING BUT...]
**In his words:** "[One sentence channeling Thiel's contrarian intellectualism — e.g., 'Competition is for losers, and this company is...' ]"

---

### Where They Agree / Disagree

| Dimension | Buffett | Thiel | Signal |
|-----------|---------|-------|--------|
| Overall | [BUY/PASS] | [FUND/PASS] | [Agreement = high conviction. Disagreement = dig deeper on the tension.] |
| Biggest strength | [what] | [what] | [same or different?] |
| Biggest concern | [what] | [what] | [same or different?] |
| Time horizon | [Buffett: forever?] | [Thiel: 10yr monopoly?] | [alignment?] |
| Key question they'd each ask | [Buffett's Q] | [Thiel's Q] | [different angles on what matters] |

**Synthesis:** [2-3 sentences on what the agreement/disagreement pattern tells you about this opportunity. Where both legends would pass is a strong negative signal. Where they'd agree to invest for different reasons is a strong positive.]
```

**Tone guidance:**
- Channel each investor's actual voice and philosophy, not a generic "value investing" or "venture" take
- Buffett: folksy, deceptively simple, obsessed with durability and honesty. References Munger, baseball metaphors, "be fearful when others are greedy."
- Thiel: contrarian, philosophical, obsessed with monopoly and secrets. References competition-as-failure, definite optimism, zero-to-one.
- Don't hedge. Each investor gives a clear verdict — that's the whole point.
- If the DD type is a **job opportunity**, reframe: Buffett assesses "would I want to own this business for 20 years?" (signals role longevity), Thiel assesses "is this a zero-to-one company?" (signals career upside).

### Phase 7: Interview / Conversation Prep

If this DD is for a job opportunity or involves upcoming conversations:

1. Generate **interview questions** (for you to ask them):
   - 5-7 questions that probe the biggest unknowns
   - Frame them diplomatically but pointedly
   - Note what a good vs. concerning answer looks like

2. Generate **likely questions they'll ask you:**
   - 5-7 probable questions based on the role/context
   - Suggested talking points for each
   - Landmines to avoid

3. Generate a **1-page conversation prep brief:**
   ```
   CONVERSATION PREP: [Company] — [Person]

   YOUR OBJECTIVES:
   1. [What you need to learn]
   2. [What you need to signal]
   3. [What you need to protect]

   THEIR LIKELY OBJECTIVES:
   1. [What they want to learn about you]
   2. [What they're trying to sell]

   KEY QUESTIONS TO ASK:
   - [question] (probing: [what unknown this addresses])

   WATCH FOR:
   - [signal to look for and what it means]

   TONE: [recommendation]
   ```

### Phase 7.5: Export PDF

Generate an IB-standard PDF with cover page, clickable TOC, and polished body using WeasyPrint.

**Prerequisites:** Requires `weasyprint` and `markdown` Python packages, plus system libraries (Pango/GLib via Homebrew). Use Homebrew's Python (`/opt/homebrew/bin/python3.12`) which links to these libraries correctly.

**Configuration variables** — replace `[placeholders]` with actual values from the DD:

| Variable | Description | Example |
|----------|-------------|---------|
| `company` | Directory name | `"Coreweave"` |
| `company_display` | Full legal name | `"CoreWeave, Inc."` |
| `ticker` | Stock ticker (or `"Private"`) | `"CRWV"` |
| `exchange` | Exchange name (or `"N/A"`) | `"NASDAQ"` |
| `dd_type` | DD type string | `"Investment Due Diligence"` |
| `date_str` | Report date | `"March 2026"` |
| `status` | Current DD status | `"Active Research"` |
| `assessment` | Go/No-Go result | `"NO-GO"` |
| `prepared_by` | Author name | `"Your Name"` |

```bash
# Install if needed (one-time)
# brew install pango glib python@3.12
# /opt/homebrew/bin/python3.12 -m pip install --break-system-packages weasyprint markdown

/opt/homebrew/bin/python3.12 << 'PYEOF'
import markdown
from weasyprint import HTML
import os, re

# --- Configuration (replace placeholders) ---
company = "[Company]"
company_display = "[Company Display Name]"
ticker = "[TICKER]"
exchange = "[EXCHANGE]"
dd_type = "[DD Type]"
date_str = "[Month Year]"
status = "[Status]"
assessment = "[Assessment]"
prepared_by = "Your Name"

md_path = os.path.expanduser(f"~/Documents/{company}/{company.upper()}-MASTER-OVERVIEW.md")
pdf_path = os.path.expanduser(f"~/Documents/{company}/{company.lower()}-master-overview.pdf")

with open(md_path, "r") as f:
    md_content = f.read()

# --- Preprocess markdown ---
# Ensure blank lines before lists (markdown lib requires this)
md_content = re.sub(r'(\*\*[^*]+\*\*:?)\n(- )', r'\1\n\n\2', md_content)

# --- Convert markdown to HTML ---
html_body = markdown.markdown(md_content, extensions=['tables', 'fenced_code', 'toc'])

# --- Extract heading IDs for clickable TOC ---
# The toc extension adds id="..." to headings automatically
heading_pattern = re.compile(r'<(h[1-4])\s+id="([^"]+)">(.*?)</\1>', re.DOTALL)
toc_entries = []
for match in heading_pattern.finditer(html_body):
    tag, slug, text = match.group(1), match.group(2), match.group(3)
    clean_text = re.sub(r'<[^>]+>', '', text).strip()
    toc_entries.append((int(tag[1]), clean_text, slug))

# --- Build clickable TOC ---
toc_html_items = ""
for i, (lvl, txt, slug) in enumerate(toc_entries):
    if lvl > 2: continue  # Only h1/h2 in TOC
    num_match = re.match(r'^(\d+)\.\s*(.+)', txt)
    if num_match:
        num = num_match.group(1).zfill(2)
        title = num_match.group(2)
    else:
        if lvl == 1 and i == 0: continue  # Skip master title
        num = f"{i:02d}"
        title = txt
    toc_html_items += f'''
    <a href="#{slug}" class="toc-entry">
        <span class="toc-num">{num}</span>
        <span class="toc-title">{title}</span>
    </a>'''

# --- Cover page HTML ---
cover_html = f'''
<div class="cover-page">
    <div class="cover-content">
        <div class="cover-confidential">C O N F I D E N T I A L</div>
        <div class="cover-rule"></div>
        <div class="cover-company">{company_display}</div>
        <div class="cover-ticker">{exchange}:&nbsp; {ticker}</div>
        <div class="cover-type"><em>{dd_type}</em></div>
        <div class="cover-subtitle"><em>Master Overview</em></div>
        <div class="cover-rule2"></div>
        <div class="cover-date">{date_str}</div>
        <div class="cover-status">Status: {status}&ensp;|&ensp;Assessment: {assessment}</div>
        <div class="cover-prepared">Prepared by {prepared_by}</div>
    </div>
    <div class="cover-disclaimer">
        This document is strictly confidential and is intended solely for the use of the intended recipient. This material does not constitute an offer, solicitation, or recommendation to buy or sell any securities. All information herein is based on publicly available sources and has not been independently verified.
    </div>
</div>'''

toc_page_html = f'''
<div class="toc-page">
    <div class="toc-header">T A B L E&ensp; O F&ensp; C O N T E N T S</div>
    <div class="toc-divider"></div>
    <div class="toc-list">{toc_html_items}</div>
</div>'''

# --- Full HTML document ---
html_doc = f"""<!DOCTYPE html>
<html><head><meta charset="utf-8">
<style>
/* === PAGE SETUP === */
@page {{
    size: Letter;
    margin: 1in 0.9in;
    @bottom-left {{ content: "{dd_type}"; font-family: 'Garamond', 'Georgia', serif; font-size: 7.5pt; color: #999; }}
    @bottom-center {{ content: counter(page); font-family: 'Garamond', 'Georgia', serif; font-size: 9pt; color: #666; }}
    @bottom-right {{ content: "{date_str}"; font-family: 'Garamond', 'Georgia', serif; font-size: 7.5pt; color: #999; }}
    @top-left {{ content: "{company_display} ({ticker})"; font-family: 'Garamond', 'Georgia', serif; font-size: 7.5pt; color: #999; }}
    @top-right {{ content: "C O N F I D E N T I A L"; font-family: 'Garamond', 'Georgia', serif; font-size: 7pt; font-weight: bold; color: #c8a000; letter-spacing: 1pt; }}
}}
@page :first {{ margin: 0; @bottom-left {{ content: none; }} @bottom-center {{ content: none; }} @bottom-right {{ content: none; }} @top-left {{ content: none; }} @top-right {{ content: none; }} }}
@page toc-page {{ margin: 1in 0.9in; @bottom-left {{ content: none; }} @bottom-center {{ content: none; }} @bottom-right {{ content: none; }} @top-left {{ content: none; }} @top-right {{ content: none; }} }}

/* === BODY === */
body {{ font-family: 'Garamond', 'Georgia', 'Times New Roman', serif; font-size: 11pt; line-height: 1.5; color: #1a1a1a; margin: 0; padding: 0; }}

/* === COVER PAGE === */
.cover-page {{ width: 8.5in; height: 11in; background: linear-gradient(160deg, #001a3a 0%, #002855 40%, #003366 100%); display: flex; flex-direction: column; justify-content: center; align-items: center; text-align: center; color: #e0e8f0; page-break-after: always; box-sizing: border-box; border-top: 4pt solid #c8a000; }}
.cover-content {{ flex: 1; display: flex; flex-direction: column; justify-content: center; align-items: center; padding: 2in 1in 0; }}
.cover-confidential {{ font-family: 'Garamond', 'Georgia', serif; font-size: 11pt; font-weight: bold; letter-spacing: 4pt; color: #c8a000; margin-bottom: 20pt; }}
.cover-rule {{ width: 60pt; height: 2pt; background-color: #c8a000; margin-bottom: 40pt; }}
.cover-company {{ font-family: 'Garamond', 'Georgia', serif; font-size: 42pt; color: #ffffff; margin-bottom: 8pt; }}
.cover-ticker {{ font-family: 'Garamond', 'Georgia', serif; font-size: 13pt; color: #8ca8c8; letter-spacing: 3pt; margin-bottom: 36pt; }}
.cover-type {{ font-family: 'Garamond', 'Georgia', serif; font-size: 18pt; color: #c8a000; margin-bottom: 4pt; }}
.cover-subtitle {{ font-family: 'Garamond', 'Georgia', serif; font-size: 18pt; color: #c8a000; margin-bottom: 30pt; }}
.cover-rule2 {{ width: 80pt; height: 1.5pt; background-color: #c8a000; margin-bottom: 24pt; }}
.cover-date {{ font-family: 'Garamond', 'Georgia', serif; font-size: 12pt; color: #e0e8f0; margin-bottom: 6pt; }}
.cover-status {{ font-family: 'Garamond', 'Georgia', serif; font-size: 10pt; color: #8ca8c8; margin-bottom: 6pt; }}
.cover-prepared {{ font-family: 'Garamond', 'Georgia', serif; font-size: 10pt; color: #8ca8c8; margin-top: 8pt; }}
.cover-disclaimer {{ font-family: 'Garamond', 'Georgia', serif; font-size: 6.5pt; color: #7090b0; text-align: center; padding: 16pt 60pt 24pt; border-top: 0.5pt solid #405060; line-height: 1.4; }}

/* === TOC PAGE === */
.toc-page {{ page: toc-page; page-break-after: always; padding-top: 40pt; }}
.toc-header {{ font-family: 'Garamond', 'Georgia', serif; font-size: 14pt; font-weight: bold; color: #002855; letter-spacing: 3pt; margin-bottom: 6pt; }}
.toc-divider {{ border-bottom: 2.5pt solid #c8a000; margin-bottom: 28pt; }}
.toc-list {{ display: flex; flex-direction: column; }}
a.toc-entry {{ display: flex; align-items: baseline; padding: 10pt 0; border-bottom: 0.5pt solid #e0e0e0; text-decoration: none; color: #1a1a1a; }}
.toc-num {{ font-family: 'Garamond', 'Georgia', serif; font-size: 12pt; font-weight: bold; color: #002855; width: 36pt; flex-shrink: 0; }}
.toc-title {{ font-family: 'Garamond', 'Georgia', serif; font-size: 12pt; color: #1a1a1a; }}

/* === HEADINGS === */
h1 {{ font-family: 'Garamond', 'Georgia', serif; font-size: 22pt; font-weight: bold; color: #002855; border-bottom: 2.5pt solid #002855; padding-bottom: 8pt; margin-top: 28pt; margin-bottom: 12pt; page-break-after: avoid; }}
h2 {{ font-family: 'Garamond', 'Georgia', serif; font-size: 16pt; font-weight: bold; color: #002855; border-bottom: 1pt solid #bbb; padding-bottom: 5pt; margin-top: 22pt; margin-bottom: 10pt; page-break-after: avoid; }}
h3 {{ font-family: 'Garamond', 'Georgia', serif; font-size: 13pt; font-weight: bold; color: #1a3a5c; margin-top: 18pt; margin-bottom: 6pt; page-break-after: avoid; }}
h4 {{ font-family: 'Garamond', 'Georgia', serif; font-size: 11.5pt; font-weight: bold; color: #333; margin-top: 14pt; margin-bottom: 4pt; page-break-after: avoid; }}

/* === TABLES === */
table {{ border-collapse: collapse; width: 100%; font-size: 9.5pt; margin: 12pt 0; page-break-inside: auto; }}
thead {{ display: table-header-group; }}
tr {{ page-break-inside: avoid; }}
th {{ background-color: #002855; color: #ffffff; font-weight: bold; padding: 6pt 8pt; text-align: left; border: 0.5pt solid #002855; }}
td {{ padding: 5pt 8pt; border-bottom: 0.5pt solid #ddd; border-left: 0.5pt solid #eee; border-right: 0.5pt solid #eee; vertical-align: top; }}
tr:nth-child(even) {{ background-color: #f7f8fa; }}
tr:nth-child(odd) {{ background-color: #ffffff; }}

/* === INLINE === */
strong {{ color: #002855; }}
code {{ font-family: 'Menlo', 'Consolas', monospace; font-size: 9pt; background-color: #f0f0f0; padding: 1pt 3pt; border-radius: 2pt; }}
a {{ color: #002855; text-decoration: none; }}

/* === LISTS === */
ul, ol {{ margin-top: 4pt; margin-bottom: 4pt; }}
li {{ margin-bottom: 3pt; }}

/* === MISC === */
hr {{ border: none; border-top: 1.5pt solid #002855; margin: 18pt 0; }}
p {{ margin-top: 4pt; margin-bottom: 8pt; orphans: 3; widows: 3; }}
blockquote {{ border-left: 3pt solid #c8a000; background-color: #faf8f0; padding: 10pt 14pt; margin: 12pt 0; font-style: italic; page-break-inside: auto; orphans: 2; widows: 2; }}
</style>
</head>
<body>
{cover_html}
{toc_page_html}
{html_body}
</body>
</html>"""

html = HTML(string=html_doc)
html.write_pdf(pdf_path)
print(f"PDF saved to {{pdf_path}} ({{os.path.getsize(pdf_path):,}} bytes)")
PYEOF
```

**If WeasyPrint is not available** (missing brew, pango, or python3.12), fall back to `markdown-pdf`:
```bash
pip3 install --user markdown-pdf 2>/dev/null
python3 -c "
from markdown_pdf import MarkdownPdf, Section
import os
company = '[Company]'
md_path = os.path.expanduser(f'~/Documents/{company}/{company.upper()}-MASTER-OVERVIEW.md')
pdf_path = os.path.expanduser(f'~/Documents/{company}/{company.lower()}-master-overview.pdf')
with open(md_path) as f: content = f.read()
# Fallback CSS: NO dark background-color on th (causes MuPDF rendering artifacts)
css = 'body{font-family:Garamond,Georgia,serif;font-size:11pt;line-height:1.45}h1,h2{color:#002855}table{border-collapse:collapse;width:100%;font-size:9.5pt}th{background-color:#e8e8e8;color:#002855;font-weight:bold;padding:5pt 8pt;border-bottom:2px solid #002855}td{padding:4pt 8pt;border-bottom:1px solid #ddd}'
pdf = MarkdownPdf(toc_level=2)
pdf.add_section(Section(content, toc=True, paper_size='Letter', borders=(72,65,-72,-65)), user_css=css)
pdf.meta['title'] = f'{company} — Due Diligence Master Overview'
pdf.save(pdf_path)
print(f'Fallback PDF saved to {pdf_path}')
"
```

Include the PDF path in the Phase 8 summary output.

### Phase 8: Summary Output

```
DUE DILIGENCE COMPLETE: [Company]

Master Overview: ~/Documents/[Company]/[COMPANY]-MASTER-OVERVIEW.md
PDF Export: ~/Documents/[Company]/[COMPANY]-MASTER-OVERVIEW.pdf
Research Files: ~/Documents/[Company]/research/ ([N] agent outputs)
Social Scans: [N] initial + [N] follow-up scans
Existing Data Ingested: [N] files
[If public company]:
SEC Filings Analyzed: [list filings, e.g., "10-K FY2025, 10-Q Q3 2025, DEF 14A 2025, 3x 8-Ks"]
SEC Data: ~/Documents/[Company]/research/sec-prefetch.md
[If Investment DD]:
Model Inputs: ~/Documents/[Company]/research/financial-sec.md → "## Model Inputs" section
Model readiness: [Ready / Partial — list gaps]

ASSESSMENT: [Go / No-Go / Need More Data]
Top insight: [single most important finding]
Biggest risk: [single biggest concern]
Critical unknown: [most important thing we still don't know]

7 Powers: [strongest power] is [Present/Emerging] | Durability: [High/Medium/Low]
Buffett: [BUY/PASS/WATCHLIST] — "[one-line reason]"
Thiel: [FUND/PASS] — "[one-line reason]"

[If follow-up scans resolved open questions]:
Questions resolved by follow-up scans: [N] of [total]
New evidence surfaced: [1-line summary of most important new finding]

[If interview prep generated]:
Conversation prep: Ready for [Person] — [N] questions prepared

Next steps:
1. [most important action]
2. [second most important action]
3. [third most important action]
```

---

## Key Rules

1. **Ingest existing data first.** Never re-research what's already known. Scan the project directory before launching agents.
2. **Distinguish claims from evidence.** Company marketing says X. Revenue data shows Y. These are different.
3. **SEC filings are the gold standard for public companies.** If the company is public, SEC filings (10-K, 10-Q, DEF 14A) are primary sources — they're audited, legally required, and more reliable than press releases or aggregator sites. Always prefer SEC data over third-party summaries. Pre-fetch in Phase 2.75 so agents have it immediately.
4. **Date every data point.** "Revenue is $10M" is useless. "Revenue was $10M in FY2025" is useful.
5. **The Go/No-Go section must have a recommendation.** "It depends" is not allowed. State the recommendation, state the assumptions, state what would change it.
6. **Open questions are ranked.** The most important unknown comes first. Each unknown has a proposed method to resolve it.
7. **Interview prep is pithy.** No one reads a 5-page prep doc. One page max. The questions should be specific enough that vague answers are themselves a red flag.
8. **Follow the agent reliability protocol.** Agents that don't write within 2 minutes get killed. No exceptions.
9. **Agents cannot parse PDFs.** Background agents can't run Bash. The orchestrator must pre-parse any PDFs (earnings releases, investor presentations) and provide text file paths to agents. See Phase 2.75 Step 3.

---

## Appendix: Master Overview Template

```markdown
# [Company/Opportunity] — Master Overview

**Status:** [Active Research / Decision Pending / Closed]
**Last updated:** [date]
**Decision deadline:** [if applicable]

## 1. Executive Summary
[10 bullets max — what is this, why does it matter, what's the ask]

## 2. Team & Leadership
[Key people, backgrounds, strengths, red flags]

## 3. Financial Analysis

### 3.1 Revenue Overview
[5-year summary table (revenue, YoY growth, net income, EPS) + one paragraph of context]

### 3.2 Business Anatomy
[THE key exhibit. One table with every revenue line: product/segment, revenue ($), % of total, YoY growth, gross margin (est.), trend arrow. Then 2-3 bullets per major segment: what it does, what's driving growth/decline, strategic importance.]

### 3.3 Margin Architecture
[Gross margin by segment + blended trend + WHY margins move. Opex breakdown table (R&D / S&M / G&A as % of revenue, 3-year trend). Operating leverage verdict: is the company scaling efficiently?]

### 3.4 Cash Generation & Capital Allocation
[FCF bridge (net income → operating CF → FCF). SBC adjustment / "true owner earnings." Capital allocation priorities: buybacks, M&A, organic investment. Shareholder return analysis.]

### 3.5 Balance Sheet
[One summary table (cash, debt, net cash/debt, current ratio). Fortress-or-fragile verdict with reasoning.]

### 3.6 Forward Outlook
[Management guidance, analyst consensus estimates (revenue + EPS), key assumptions, catalysts, and risks to estimates.]

## 4. Competitive Landscape
[Direct competitors, positioning, moats, threats]

## 5. Go-to-Market & Sales
[GTM strategy, pipeline, customer segments, traction]

## 6. Technical / Product Assessment
[Product maturity, tech stack, IP, differentiation]

## 7. Risk Register
[Top 5-10 risks ranked by likelihood × impact]

## 8. Valuation Analysis (Investment DD) / Equity & Compensation Analysis (Job DD)
[Investment DD: multiples, DCF, scenario-based valuation, comps table. Job DD: offer details, equity valuation scenarios, expected value, comparables.]

## 9. Go / No-Go Assessment
[Recommendation with explicit reasoning]

## 10. Open Questions & Research Backlog
[What we don't know yet, ranked by importance]
```
