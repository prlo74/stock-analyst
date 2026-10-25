---
name: "stock-analyst"
description: "Researches pasted stock tickers from filings, signed deals and recent press, scores and ranks them, and publishes a visual buy-recommendation page. Use when the user pastes tickers to research, compare, rank or rate, asks which stock to buy, or wants a stock report or tear sheet."
---

# Stock Analyst

Turn a pasted list of tickers into a ranked, sourced, visual recommendation page.

**Input:** tickers, plus optional instructions (horizon, theme, risk appetite, pillars to stress, a benchmark, a basket to size).
**Output:** one published page with a ranking, a rating per ticker, the deals and press behind each call, and the conditions that would reverse it. Then a three-line summary in chat.

The reader is a smart non-specialist. They should be able to see the answer in ten seconds, understand why in two minutes, and audit any number in one click.

## What makes this hard (read before starting)

Three failure modes account for most bad stock write-ups. The whole workflow is built to prevent them.

1. **Numbers from memory.** Prices, multiples and backlog go stale in weeks. A remembered figure is indistinguishable from a real one once it is on the page. So every figure on the page comes from something fetched in this run, and carries a source and an as-of date. A figure that cannot be fetched is shown as "n/a" and lowers confidence; it is never estimated silently.
2. **Press-release optimism.** Companies announce "partnerships" and "frameworks" that bind nobody. A deal counts toward the score only according to how binding and how large it is. Announced is not signed, and signed is not revenue.
3. **Forced ranking.** Ranking five stocks always produces a #1, which reads as "buy" even when all five are expensive. Rank order and rating are separate judgments: the rating is absolute, and it is acceptable, and useful, to say none of them clears the bar.

## Workflow

Track these as tasks. Do not ask clarifying questions unless a ticker is ambiguous (same symbol on two exchanges, or a name that maps to several listings); otherwise state the assumptions at the top of the page and proceed.

```
- [ ] 1. Scope: parse tickers and instructions, set horizon, benchmark, weights
- [ ] 2. Evidence: build the evidence pack for every ticker
- [ ] 3. Deals and press: build the deal ledger for every ticker
- [ ] 4. Score: pillar scores, composite, gates, rating, confidence
- [ ] 5. Stress test: argue against each call, write the reversal triggers
- [ ] 6. Build the page from the evidence file
- [ ] 7. Verify, publish, summarize
```

### 1. Scope

- Normalize tickers and resolve each to a company, exchange, sector and reporting currency. Flag anything delisted, acquired, or an ETF (an ETF gets a holdings-and-cost summary, not a company score).
- Defaults unless the user says otherwise: **12-month horizon**, **benchmark = the main index of the listing market** (S&P 500 for US), **default weights** below.
- Read the user's extra instructions as changes to weights, horizon, or emphasis, and record them. A theme ("AI data center suppliers") becomes the lens for the thesis line and for which deals matter.
- More than 10 tickers: do a quick first pass (price, valuation, last earnings, deals) on all of them, then full depth on the top 8 and one line each for the rest. Say so on the page.

### 2. Evidence pack

Collect per ticker, running independent lookups in the same turn. Work down the source ladder and stop at the highest rung that has the figure.

| Rung | Source | Use for |
|---|---|---|
| 1 | Company filings: 10-K, 10-Q, 8-K and exhibits; or the home-market equivalent | Revenue, margins, cash flow, debt, backlog or RPO, share count, risk factors, deal terms |
| 2 | Company earnings release, slides, call transcript, IR newsroom | Guidance, segment commentary, order trends, management tone |
| 3 | Data aggregators (StockAnalysis, Yahoo Finance, Finviz, Macrotrends) | Price, 52-week range, multiples, consensus estimates and targets, short interest, Altman Z, Piotroski F |
| 4 | Reputable press (Reuters, Bloomberg, WSJ, FT, CNBC, trade press) | Recent events, deal context, disputes, regulatory news |
| 5 | Blogs, forums, social | Leads only. Never the sole source for a figure or a deal |

Reported figures come from rungs 1 and 2. Rung 3 is fine for market data and consensus, and a `/statistics/` page on StockAnalysis usually gives the whole valuation and balance sheet block with its date in one fetch.

**Minimum fields per ticker** (mark each as reported, consensus, or own estimate):

- Market: price and date, market cap, 52-week high and low, 3, 6 and 12-month return versus benchmark, short interest.
- Growth: revenue and EPS growth, latest quarter year on year and trailing twelve months; next-year consensus growth; direction of estimate revisions since the last report.
- Profitability: gross, operating and free-cash-flow margin with direction; ROIC; operating cash flow versus net income.
- Balance sheet: cash, total debt, net debt to EBITDA, interest coverage, share count change year on year, Altman Z, Piotroski F.
- Valuation: forward P/E, EV/EBITDA, P/FCF, PEG; the same multiples for sector peers or the stock's own 5-year range; mean analyst target and analyst count.
- Demand: backlog or RPO and its growth, book-to-bill, guidance changes this year, customer concentration.
- Calendar: next earnings date and any dated catalyst inside the horizon.

**Sector substitutions.** Standard multiples mislead in some sectors, so swap rather than force:

- Banks and insurers: P/TBV, ROTCE, CET1, net interest margin, credit costs. Skip EV/EBITDA and Altman Z.
- REITs: P/FFO, AFFO payout, debt to EBITDA, occupancy.
- Software: RPO and current RPO, net revenue retention, Rule of 40, stock-based compensation as a share of revenue.
- Pre-profit and biotech: cash runway in quarters, dilution history, dated pipeline or product catalysts, EV/sales.
- Cyclicals and commodities: mid-cycle earnings rather than peak, cost-curve position.

**Non-US listings.** Use the home regulator and exchange disclosure (for Japan: TDnet timely disclosures and EDINET filings, kessan tanshin earnings summaries) plus the company's IR page. Quote figures in the reporting currency and say which.

### 3. Deals and press ledger

This is the part most reports skip, and it is where the edge is. Look back **12 months for deals, 90 days for press**.

**Finding US deals (this route is tested and works with WebFetch):**

1. CIK lookup: `https://efts.sec.gov/LATEST/search-index?keysTyped=TICKER`
2. List 8-Ks with their item codes (pad the CIK to 10 digits):
   `https://efts.sec.gov/LATEST/search-index?q=&forms=8-K&dateRange=custom&startdt=YYYY-MM-DD&enddt=YYYY-MM-DD&ciks=0001234567`
3. Open the filing folder (CIK without padding, accession number without dashes):
   `https://www.sec.gov/Archives/edgar/data/CIK/ACCESSIONNODASHES/`
4. Read the main 8-K and the exhibits. EX-99.1 is the press release; EX-10.x is the contract itself.

The old `cgi-bin/browse-edgar` pages are blocked to fetchers; do not use them. If SEC pages fail, fall back to the company IR newsroom and say the deal is press-sourced.

**8-K item codes worth reading.** Filings are due within four business days of the event, so these are the freshest primary source there is.

| Item | Meaning | Reads as |
|---|---|---|
| 1.01 | Entry into a material definitive agreement | A real, binding deal. The strongest deal evidence |
| 1.02 | Termination of a material agreement | Lost deal. Check which revenue it removes |
| 2.01 | Acquisition or disposal completed | M&A closed |
| 2.02 | Results of operations | Earnings release, guidance |
| 2.03 | New direct financial obligation | New debt |
| 2.05 / 2.06 | Restructuring / impairment | Trouble or cleanup |
| 3.02 | Unregistered equity sale | Dilution |
| 4.01 / 4.02 | Auditor change / prior financials not reliable | Red flag. Triggers a gate |
| 5.02 | Officer or director change | CFO exits matter most |
| 7.01 / 8.01 | Reg FD / other events | Often contract wins, investor days |

Then search press and the IR newsroom for what the filings miss: customer wins below the materiality line, partnerships, government awards, product launches, lawsuits, regulatory action, analyst rating changes, insider buying or selling.

**Record every deal as one row:**

```
date | counterparty | type | disclosed value | term | binding status | source rung | materiality | one-line read
```

- **Type:** customer contract, supply agreement, partnership, licensing, M&A, financing, government award.
- **Binding status:** `definitive` (signed, in an 8-K or contract exhibit) · `announced` (company press release, terms stated) · `soft` (MOU, LOI, framework, "selected as", "in talks") · `reported` (press only, company has not confirmed).
- **Materiality:** annualized deal value divided by trailing revenue. Under 1% is noise, 1 to 5% is notable, over 5% is thesis-relevant. If value is undisclosed, say so; do not guess one.

**Order-book terms, since they are routinely confused:**

- **RPO** (remaining performance obligations) is a required accounting disclosure in the revenue note: contracted revenue not yet recognized. It is the most comparable figure.
- **Backlog** is defined by each company. It may include unfunded, cancellable or option work. Read the definition before comparing two companies.
- **Bookings** are orders taken in the period. **Book-to-bill** is bookings divided by revenue; above 1.0 means the order book is growing.

On the page, call backlog "signed work" on first use and define it once.

### 4. Score

Score each pillar from 1 to 5 (half points allowed). **3 means average for its sector, or mixed evidence.** Anchor every score on the evidence pack, and write a one-clause reason for each, because a score without a reason cannot be audited.

| Pillar | Weight | 5 looks like | 1 looks like |
|---|---|---|---|
| Demand and deals | 20 | Order book growing faster than revenue, book-to-bill above 1.1, several definitive deals over 1% of revenue, guidance raised | Order book shrinking, terminations, deals are all soft, guidance cut |
| Growth | 15 | Revenue and EPS growing well above sector, estimates revised up | Shrinking, estimates falling |
| Profitability and quality | 15 | Margins expanding, ROIC well above cost of capital, cash flow exceeds net income, Piotroski 7 to 9 | Margins contracting, earnings not backed by cash, Piotroski 0 to 3 |
| Balance sheet | 15 | Net cash or under 1x net debt to EBITDA, coverage above 10x, no dilution | Over 4x leverage, coverage under 2x, serial dilution, short runway |
| Valuation | 20 | Clearly cheap versus peers and own history for its growth, PEG under 1 | Priced for perfection: top of own range and peers with slowing growth |
| Market setup | 15 | Outperforming benchmark, well off the high without broken fundamentals, insiders buying, dated catalyst ahead | Underperforming on rising volume, heavy insider selling, crowded and extended |

**Composite** = Σ (weight × score ÷ 5), giving 20 to 100. All 3s gives 60. Compute it in code, not by hand, so the page and the arithmetic cannot disagree.

**Weight presets**, in pillar order above. Use the one that fits the stated horizon, or the user's own weights (rescale to 100):

- Default, 12 months: 20 / 15 / 15 / 15 / 20 / 15
- Short term, under 3 months: 25 / 15 / 10 / 10 / 10 / 30
- Long term, 3 years or more: 15 / 15 / 25 / 15 / 25 / 5

**Rating** (always shown with its horizon and benchmark, because a rating without them means nothing):

| Composite | Rating | Meaning |
|---|---|---|
| 80 and up | Buy | Expected to beat the benchmark over the horizon; evidence is broad |
| 68 to 79 | Accumulate | Attractive, with one real weakness. Build on pullbacks |
| 52 to 67 | Hold | Balanced. Not worth new money at this price |
| Under 52 | Avoid | Expected to lag, or risk is not paid for |

**Gates.** Some facts outweigh any score. If one applies, cap the rating at Hold and print the gate next to the rating:

- Going-concern language, or a material weakness in internal controls
- An 8-K Item 4.02, or an unexplained auditor change (4.01)
- Altman Z below 1.81 for a non-financial company
- A covenant breach, or a financing the company needs and has not secured
- A regulatory or legal action that could remove a core product or market

**Confidence**, per ticker, separate from the rating:

- **Certain:** every pillar rests on rung 1 or 2 data from the latest quarter.
- **Likely:** one pillar rests on aggregators or inference.
- **Guessing:** two or more pillars lack primary data, or the key figures conflict across sources. Cap the rating at Accumulate and say what is missing.

**Rank** by composite. Break ties by confidence, then by the Demand and deals score. If no ticker reaches Accumulate, lead the page with that fact.

Only add **basket weights** when the user asks to size positions. Size by rating and confidence, cap any single name at 30%, and label the weights as a judgment call.

### 5. Stress test

Before writing, switch sides. For each ticker, write the strongest honest case against the rating in two sentences: the bear case for a Buy, the bull case for an Avoid. If that case is stronger than expected, move the score, not the wording.

Then write, per ticker:

- **Main risk:** one line, specific and checkable. "Fixed-price contracts with electrician wages rising" is a risk. "Macroeconomic uncertainty" is not.
- **What would change the call:** one or two observable triggers with a date or a threshold. "Backlog falls quarter on quarter", "two of the four largest customers cut capex guidance".

And one check across the set: do the tickers share a single dependency (same customers, same commodity, same rate sensitivity)? If so, say it near the top, because owning five names with one risk is one bet.

### 6. Build the page

First write everything to one `evidence.json` and generate the page from it, so no figure is typed twice:

```json
{
  "as_of": "YYYY-MM-DD", "horizon": "12 months", "benchmark": "S&P 500",
  "weights": {"demand": 20, "growth": 15, "quality": 15, "balance": 15, "valuation": 20, "setup": 15},
  "tickers": [{
    "ticker": "", "name": "", "role": "what it does, in eight words",
    "price": 0, "price_date": "", "hi52": 0, "lo52": 0, "target_mean": 0,
    "metrics": {"fwd_pe": {"value": 0, "kind": "consensus", "source": "", "url": "", "date": ""}},
    "scores": {"demand": {"score": 0, "why": ""}},
    "composite": 0, "rank": 0, "rating": "", "gate": null, "confidence": "",
    "case": ["", "", ""], "main_risk": "", "reversal": [""],
    "deals": [{"date": "", "counterparty": "", "type": "", "value": "", "term": "",
               "status": "definitive", "materiality_pct": null, "url": "", "read": ""}],
    "press": [{"date": "", "headline": "", "tone": "positive|neutral|negative", "url": ""}]
  }]
}
```

Load the `artifact-design` and `dataviz` skills if they are available, then publish with the Artifact tool. Without an Artifact tool, deliver a single self-contained `.html` file.

**Page order.** Answer first, evidence after, method last.

1. **Header.** Eyebrow with scope and price date. Headline is the conclusion as a sentence ("Two of these five are buys; the rest are priced for good news"), not a label. A two-sentence thesis under it.
2. **Leaderboard.** One horizontal bar per ticker, sorted by composite, score printed at the bar end, rating chip and confidence beside it. This is the ten-second view.
3. **Pillar heatmap.** Tickers down, six pillars across, one cell per score with the number printed in it. This answers "ranked on different aspects" at a glance and shows why #1 is #1.
4. **Price versus growth scatter.** Forward P/E on one axis, expected growth on the other, each ticker labelled directly. Cheap-and-growing sits in one corner; say which.
5. **52-week range strips.** One row per ticker: a track from low to high, a marker at the current price, a tick at the mean analyst target.
6. **Deal timeline.** A shared 12-month axis, one lane per ticker, one dot per deal. Dot size is materiality; filled is definitive, outlined is announced, dashed is soft. The largest deals are labelled.
7. **Ticker rows.** One ruled row per ticker in rank order, three columns: identity (ticker large, name, role, rating chip) · case (three evidence bullets, main risk, what would change the call) · figures (a short definition list in tabular numerals). Under each row, its deals and its three most relevant press items, each linked.
8. **What ties them together.** The shared dependency, and the conditions for cutting the whole set.
9. **How to read this.** Plain definitions of every term used, the rating scale with horizon and benchmark, the weights, and the confidence labels.
10. **Sources and fine print.** Every source with its date. Then one sentence: this is research, not personal investment advice, and it does not know the reader's finances.

**Design rules.** The model is a one-page tear sheet: calm, dense, ruled, with figures the eye can run down.

- Charts are inline SVG or CSS drawn from `evidence.json`. Draw a chart only from fetched numbers. In particular, no price line chart unless a real daily series was fetched; the range strip replaces it.
- One accent hue. Encode score with a single-hue light-to-dark ramp and print the number in the cell, so the page still reads in grayscale and for color-blind readers. Never rely on red versus green; ratings are words in chips.
- Label marks directly instead of using a legend wherever there is room. Every chart gets a one-line title that states its finding, and a source line.
- Display face for tickers and headings, a readable sans for text, monospace tabular numerals for figures. Define colors as tokens with light and dark themes. Rows collapse to one column on a phone, and nothing scrolls sideways.
- Plain words. Write "signed work" before "backlog", "price divided by next year's expected profit" before "forward P/E". Give every term one definition, in section 9.
- Keep prose short: three bullets per case, one line per risk. The visuals carry the comparison.

### 7. Verify, publish, summarize

Check before publishing; fix and re-check rather than noting the problem:

- [ ] Every figure on the page exists in `evidence.json` with a source and a date
- [ ] Composite scores recomputed in code match the page; weights sum to 100
- [ ] Rank order follows composite; every rating matches its band, gate and confidence cap
- [ ] Price date is the latest close available, and no financial figure is older than the latest reported quarter
- [ ] Each deal has a binding status, and no `soft` or `reported` deal is described as signed
- [ ] Reported, consensus and own-estimate figures are labelled as such
- [ ] Every ticker has a main risk and a reversal trigger specific enough to check
- [ ] Links open the source; the page reads correctly in dark mode and at phone width

Then reply in chat with at most three lines: the ranking with ratings, the single most important reason for the top call, and the biggest caveat or data gap. Do not restate the page.

## Judgment notes

- **Disagree with consensus when the evidence does.** Analyst targets are an input to Market setup and a cross-check on Valuation, never the rating itself. Sell-side ratings skew heavily to buy.
- **A cheap stock with a shrinking order book is not cheap**, and an expensive one with accelerating definitive deals may not be expensive. When Valuation and Demand disagree by two points or more, say so explicitly in the case.
- **Recency beats volume.** One filing from last week outweighs ten articles from last quarter. If results are due within two weeks, flag that the call may be stale shortly and name the date.
- **Say what was not checked.** If transcripts were unreachable, or a company discloses no backlog, put that on the page. A stated gap is a feature; a hidden one is a defect.
- **Follow-ups.** "Add TICKER", "re-weight toward value", or "refresh" should reuse `evidence.json`, re-fetch only what has changed, and republish to the same page.