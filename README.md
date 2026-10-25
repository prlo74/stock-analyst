# stock-analyst

A Claude skill that turns a pasted list of stock tickers into a ranked, sourced, visual recommendation page.

Paste tickers (plus optional instructions such as horizon, theme, or risk appetite) and the skill researches each one, scores it, and publishes a single page with a ranking, a rating per ticker, the deals and press behind each call, and the conditions that would reverse it. It then gives a three-line summary in chat.

> This is research tooling, not personal investment advice. It does not know your finances.

## What it does

- **Researches from primary sources first.** Filings (10-K, 10-Q, 8-K and exhibits), earnings releases and transcripts come before data aggregators, and press and social come last.
- **Builds a deal ledger.** Every deal from the last 12 months is recorded with its binding status (`definitive`, `announced`, `soft`, `reported`) and its size relative to trailing revenue, so a press-release "framework" is not treated like a signed contract.
- **Scores six pillars** from 1 to 5, each with a one-clause reason: demand and deals, growth, profitability and quality, balance sheet, valuation, and market setup.
- **Rates absolutely, ranks relatively.** The rating (Buy, Accumulate, Hold, Avoid) is separate from the rank order, so the page can say that none of the tickers clears the bar.
- **Applies hard gates.** Going-concern language, restatements, a low Altman Z, covenant breaches, or a legal threat to a core product cap the rating at Hold.
- **Stress-tests every call.** Each ticker gets the strongest case against its rating, a specific main risk, and observable triggers that would change the call.
- **Shows its work.** Every figure on the page carries a source and an as-of date, and a figure that could not be fetched is shown as "n/a" rather than estimated.

## Scoring at a glance

| Pillar | Default weight |
|---|---|
| Demand and deals | 20 |
| Growth | 15 |
| Profitability and quality | 15 |
| Balance sheet | 15 |
| Valuation | 20 |
| Market setup | 15 |

| Composite | Rating |
|---|---|
| 80 and up | Buy |
| 68 to 79 | Accumulate |
| 52 to 67 | Hold |
| Under 52 | Avoid |

Weight presets exist for short-term (under 3 months) and long-term (3 years or more) horizons, and you can supply your own weights. Each ticker also gets a confidence label (Certain, Likely, or Guessing) that is independent of its rating. The full rubric is in [`SKILL.md`](SKILL.md).

## Output

One published page, in this order:

1. Headline conclusion and thesis
2. Leaderboard of composite scores
3. Pillar heatmap
4. Price versus growth scatter
5. 52-week range strips
6. Deal timeline
7. Per-ticker rows with deals and press
8. Shared dependencies across the set
9. Plain-language definitions and method
10. Sources and fine print

Without an Artifact tool, the skill delivers a single self-contained `.html` file instead.

## Usage

Install the skill in Claude, then ask in plain language:

```
Research NVDA, AMD, AVGO and rank them for a 12-month horizon.
```

```
Which of these is the best buy? TSM, ASML, INTC. Weight toward valuation.
```

Follow-ups such as "add MU", "re-weight toward value", or "refresh" reuse the saved evidence and re-fetch only what changed.

## Repository contents

| File | Purpose |
|---|---|
| [`SKILL.md`](SKILL.md) | The skill itself: workflow, scoring rubric, gates, and page design rules |

## Notes

- Markets outside the US are supported through the home regulator's disclosures (for example TDnet and EDINET for Japan), with figures quoted in the reporting currency.
- Sector-specific metrics replace standard multiples where the standard ones mislead (banks, insurers, REITs, software, biotech, cyclicals).
- Lists of more than 10 tickers get a quick first pass on all of them and full depth on the top 8.
