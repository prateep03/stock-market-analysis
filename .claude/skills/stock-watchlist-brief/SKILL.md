---
name: stock-watchlist-brief
description: Generate a weekly (or daily) stock market research brief for a project's watchlist. Use whenever the user asks for stock analysis, market brief, watchlist review, weekly performance summary, investment recommendations based on their watchlist, or phrases like 'recommend stocks from my watchlist', 'how did my stocks do this week', 'flag anything unusual', or 'run the data agent'. The skill reads the project's watchlist YAML for tickers and source policy, fetches price/change/volume data from approved sources only, and produces a structured brief with ranked flagged candidates. Trigger even if the user doesn't say 'brief' — any request to analyze or summarize watchlist stocks qualifies.
metadata:
  - author: prateep.mukherjee
  - created: 09-09-2026
  - version: 1.0
---

# Stock Watchlist Brief

A research-screening workflow that turns a watchlist YAML and a set of approved data sources into a structured weekly brief: benchmark snapshot, per-ticket performance table, and up to five flagged candidates for further research. It is an attention-direction tool - it answers *where should I look next?*, not *what should I buy or sell?*.

---

## Step 1 - Find and read the watchlist config

Look for a watchlist config in this order:

1. `.claude/watchlist.yaml` in the project repo (userr `project_search` or the Github connector to read it).
2. If the watchlist file is not found in the project repo, prompt the user to upload the same in the same context.
3. Ask the user where their watchlist lives if neither is found.

Parse from the YAML:
- **benchmarks**: index symbols and exchanges (e.g. NIFTY 50, BANKNIFTY on NSE)
- **equities**: symbol, exchange, sector for every ticker.
- **source_policy**: which sources are primary (authoritative) vs secondary (discovery/cross-check). If the YAML has no explicit policy section, note that and default to: exchange sites (NSE/BSE) as primary; Moneycontrol, Screener.in, TradingView, Economic Times as secondary.

---

## Step 2 - Source resolution strategy

Work through the approved sources in priority order and stop as soon as you get a value. The goal is to respect the watchlist's source policy - **never pull from a source that is not listed in the YAML**.

**Typical priority for India-focussed watchlists:**

1. NSE (`nseindia.com`) - primary for prices, volume, F&O, OI, corporate actions.
  > *Caveat:* NSE renders data via JavaScript, so WebFetch often returs empty tables. Attempt it; if the table is empty, mark as attempted and move on.

2. BSE (`bseindia.com`) - parallel primary; same JavaScript caveat.

3. Company investor-relations pages - primary for earnings dates, dividends, guidance.

4. TradingView (`tradingview.com/symbols/NSE-<TICKER>/`) - rich secondary; usuallly renders price, weekly change, 52-week range, and community analysis ideas.

5. Screener.in, MoneyControl, Economic Times - secondary for context and cross-checking.

For each ticker fetch at minimum:
- Current price
- Weekly % change (or daily change if weekly is not available)
- 52-week high/low when available.
- Any notable scheduled event (earnings, dividend) in the next 30 days
- Options volume and OI change if the source exposes it. 

If a value is unavailable for all approved sources, record it as **UNAVIALABLE** - never estimate or carry forward a stale figure.

---

## Step 3 - Data collection approach

Use `WebFetch` for individual stock pages (e.g. TradingView symbol pages return real data).
Use `WebSearch` sparingly and only within allowed domains to find recent news or events.

Batch your fetches; for TradingView, the market-movers page (`/markets/stocks-india/market-movers-large-cap/`) gives a fast snapshot of many tickers at once before you fetch individual pages. Use it as a quick screener, then deep-dive on tickers that show unusual movement.

Document every data point with:
- source [URL]
- retrieval timestamp (today's data is sufficient in a brief)
- whether the data is real-time, delayed or end-of-day.

---

## Step 4 - Produce the brief

Structure the output exactly as follows:

### Header
```
## Weekly Watchlist Brief — Week ending [DATE]
> Attention-direction tool only. Not financial advice.
> Sources: [list approved sources actually used]
```

### Benchmark Snapshot
A two-column table: index | level | weekly change | notes.
If a benchmark's data is UNAVAILABLE, say so explicitly.

### Watchlist Performance Table
One row per equity. Columns: Symbol | Sector | Price (₹) | Weekly Δ | Signal.

Signal uses a simple traffic-light convention:
- 🟢 Mild positive (+0.5% or more, in line with or beating the benchmark)
- 🟡 Flat (within ±0.5%)
- 🟠 Mild underperformer (−0.5% to −2%)
- 🔴 Notable weakness (more than −2%, or a significant earnings miss)

### Flagged Candidates (0–5)
Return **at most five** tickers worth researching further. Return fewer when the
evidence doesn't justify more flags, and say plainly when nothing is genuinely unusual.

For each flag provide:
- The specific unusual measurement and its baseline (e.g., "down 5% the week of an
  earnings beat")
- Whether open interest or volume data supports newly opened positions (if available)
- The most likely *ordinary* explanation (earnings, index rebalance, sector rotation,
  known hedge, macro event)
- The missing data that would determine whether the activity matters
- Confidence level: use **LOW** when evidence is weak or ambiguous

### Data Gaps
List every field that came back UNAVAILABLE and which source was tried. This section is
important — it lets the user know what a manual check of NSE/BSE would add.

### Single Most Useful Research Question
End with one sentence: the most leveraged question to answer first, given what the data
showed. This is the handoff to the human researcher.

---

## Guardrails

- **Never fabricate a price.** If WebFetch returns loading spinners or empty tables,
  mark the value UNAVAILABLE and note which source was attempted.
- **Never infer direction from options volume alone.** Every trade has a buyer and a
  seller. Describe the observation, not a story about intent.
- **Always distinguish** between what the data shows (fact) and what it might mean
  (interpretation). Use "may indicate", "could suggest", not "confirms" or "proves".
- **Source attribution is mandatory.** Every figure in the output must trace back to an
  approved source URL. No orphaned numbers.
- Append a reminder at the end that primary NSE/BSE confirmation is required before
  acting on any flag — secondary-source data is not sufficient for a trade decision.

---

## Handling source failures gracefully

NSE and Moneycontrol are frequently inaccessible via WebFetch (JS rendering and proxy
blocks, respectively). This is normal — don't retry more than once per source.
The workflow degrades gracefully:
- Primary source blocked → fall to the next approved source
- All approved sources for a field blocked → UNAVAILABLE + note in Data Gaps
- If TradingView is the only source that responds, use it, state it, and remind the
  user to cross-check on NSE before acting

---

## Example flag write-up

**ICICIBANK — unusual weekly drop despite earnings beat**
- Down ~5% on the week; earnings came in at ₹14.60 vs ₹14.02 estimate (+4.1% beat).
- A stock falling on good earnings can signal institutional de-risking or sector
  rotation; it could also be catch-up to prior over-performance.
- Most likely ordinary explanation: private banking sector-wide rotation (check
  BANKNIFTY weekly change to confirm or rule this out).
- Missing data: options OI change, FII/DII flow data from NSE, BANKNIFTY level.
- Confidence: **LOW** — insufficient data to distinguish stock-specific from sector-wide.