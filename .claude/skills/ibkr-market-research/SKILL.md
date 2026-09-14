---
name: ibkr-market-research
description: Use when the user wants market research to support an IBKR trading decision — quote/price lookups, price history and charts, option chain analysis, or thematic/competitor context on a company. Triggers on ticker symbols with a research request, "look up", "quote", "price history", "option chain", "IV", "competitors of X", "what sector/theme is X in", or any question about a stock/future/option available through the connected Interactive Brokers (IBKR) account. Read-only — does not place, modify, or cancel orders.
---

# IBKR Market Research

Read-only research assistant built on the `Interactive_Brokers_IBKR` MCP
connector. It answers "what's happening with this instrument" and "how does
this company fit into its sector/competitors" questions using live IBKR
data. It never places, drafts, or implies submission of an order — that is
out of scope for this skill.

## Hard rule: no trading actions

Do not call `create_order_instruction`, `create_alert`, `create_watchlist`,
or any other write/mutating IBKR tool from this skill. If the user asks to
actually place a trade, set an alert, or build a watchlist, tell them that's
a separate capability not covered here, and stop — don't improvise a
workaround with research tools.

## Step 0: resolve the instrument

Almost every workflow below starts by turning a name/ticker into a contract
ID:

- Stocks/options underlyings/most instruments: `search_contracts`. Match on
  the row whose `symbol` exactly equals the ticker the user means, and check
  `sections` for the security type you need (e.g. must include `OPT` before
  you can look up options on it).
- Futures: `search_futures` instead — it returns `contract_id_ex` values
  (e.g. `"12345@CME"`) already scoped to an exchange.

A single symbol commonly matches many unrelated companies across different
countries (e.g. `IAG` alone returns International Consolidated Airlines on
LSE *and* Bolsa de Madrid, Insurance Australia Group on ASX, iA Financial
on TSE, IAMGOLD on NYSE). When the user names an exchange or market
explicitly ("on Bolsa de Madrid", "the London listing", "the ASX one"),
treat that as the primary disambiguator: match the row whose `exchange`
field corresponds to the named market before falling back to
`description`/`country_code`. If a symbol is still ambiguous after that
(multiple listings, multiple countries, or a name that maps to several
companies) with no exchange named, ask the user to disambiguate rather than
guessing — a wrong contract ID silently produces answers about the wrong
instrument.

Never show raw `contract_id` / `contract_id_ex` / `expiration_id` values to
the user — they're internal plumbing for chaining calls. Present results by
symbol, company name, expiry date, and exchange instead.

## Workflow: quote / snapshot

1. Resolve the contract ID (Step 0).
2. `get_price_snapshot` for the live/last price, bid/ask, volume, etc.

Use this for "what's X trading at", "what's the bid/ask on this option", or
as a sanity check before discussing a price level with the user.

**"Closing price" is a different question from "last price."**
`get_price_snapshot`'s `last` field is whatever the most recent trade
happened to be — during market hours that's a live tick; outside hours or
on thin instruments it can still be stale or mid-session. The `last` object
carries an `is_close` flag, which appears to mark whether the price is a
settled close (it read `false` on both legs of a 2026-09-14 option lookup
during the session). That reading is an inference from the field name and
observed values, not something confirmed against IBKR documentation — so
treat `is_close: false` as a reason to double-check rather than as proof of
anything, and don't quote the flag's meaning to the user as fact.

For a genuine closing-price question, cross-check the last bar's `close`
value from `get_price_history` (`step: ONE_DAY`, a short `period` like
`TWO_DAYS` or `step_count: 1-2`), and say explicitly which source the
number came from. Note this cross-check has not yet been validated against
a known-good close on this connector; if the two sources agree, you have
real confidence, and if they disagree, report both rather than silently
picking one. Don't silently present a non-close `last` price as "today's close" —
flag the distinction to the user, especially if the two sources disagree
(thin/illiquid options in particular can show a stale last-trade print well
away from the current bid/ask or the settlement price).

## Workflow: price history / chart

1. Resolve the contract ID (Step 0).
2. `get_price_history` with an appropriate lookback/bar size for what's being
   asked (a "how has it done this year" question needs daily bars over
   months, not minute bars).
3. Summarize the trend/range/volatility in prose; only dump raw bar data if
   the user explicitly wants a table.

If you pass the contract's native `exchange` (e.g. a non-US listing like
`BM`, `SBF`, `ASX`) and get back `"No market data permissions"`, don't
immediately conclude the data is unavailable — retry the same `contract_id`
with `exchange` omitted (SMART default). This worked for a Bolsa de Madrid
listing whose `BM` call was rejected on entitlements; it is one observed
case, not a guaranteed rule, so treat it as worth trying rather than as
something that always works.

**Sanity-check what comes back before using it.** A fallback that silently
resolves to a different venue or listing of the same company will produce
confidently wrong analysis. Check that the price scale and currency match
the listing you intended — e.g. IAG on Madrid trades around €4-5 while the
London line trades in pence around 300-400, so a returned series in the
wrong scale is obvious if you look for it, and invisible if you don't. If
the scale doesn't match the market you meant, say so rather than analyzing
it.

This fallback does not apply to futures/FOP, which genuinely require the
native exchange — the `get_price_snapshot` and `get_price_history` tool
descriptions both spell out that SMART routing only covers SMART-eligible
instruments (US equities and equity options).

## Workflow: technical analysis (trend, moving averages, support/resistance)

1. Resolve the contract ID (Step 0).
2. `get_price_history` with daily bars (`step: ONE_DAY`) over a lookback
   matched to the ask — `ONE_YEAR` is a reasonable default for a general
   "technical analysis" request; shorten it if the user asks about a
   specific recent window.
3. Compute, from the daily closes:
   - Moving averages: 20/50/100/200-day SMA (use whichever fit the
     lookback length — don't compute a 200-day SMA off 90 days of bars).
   - RSI(14) from daily closes, using **Wilder's smoothing** — seed with a
     14-period simple average of gains/losses, then roll it forward as
     `avg = (prev_avg * 13 + current) / 14`. This is what IBKR and standard
     charting platforms show, so it's what the user will be comparing
     against. Do not substitute a plain 14-period simple average of
     gains/losses: it diverges sharply in a sustained trend and will flip
     the overbought/oversold verdict (IAG on 2026-09-14 read 25.2 on the
     simple method — "oversold" — versus 36.1 on Wilder's, which is not).
   - Annualized volatility from the stdev of daily log/simple returns
     (`stdev * sqrt(252)`), for later use in any volatility-based range.
4. Support/resistance zones: find local pivots across the lookback — a bar
   that is the max/min within a symmetric window, e.g. Β±5 trading days.
   Take resistance pivots from the bar **highs** and support pivots from
   the bar **lows**, not from closes: closes understate the extremes the
   market actually traded and tested, and the two sources give materially
   different levels (on IAG, close-based pivots put support at 4.33 where
   the lows put it at 4.14). Use that same high/low basis for any 52-week
   high/low you quote alongside the zones, so the levels in one answer are
   all derived consistently. Cluster nearby pivots into zones rather than
   citing single-cent price points — real S/R is a range, not an exact tick.
5. State where price sits relative to the moving averages and the nearest
   support/resistance zones, and what RSI implies, in plain prose.

This produces a chart-based read (trend direction, momentum, levels), not a
fundamental one — say so if the user's question mixes in fundamentals
(earnings, valuation, news) that this workflow doesn't cover.

## Workflow: option chain analysis

1. Resolve the **underlying's** contract ID via `search_contracts` (equity
   options) or `search_futures` (futures options) — confirm the row's
   `sections` includes `OPT` or `FOP` respectively.
2. `get_price_snapshot` on the underlying first, so you know the spot price.
3. `get_option_parameters` with that underlying contract ID to get available
   expirations. Watch for duplicate dates with different `trading_class`
   values (e.g. `TSLA` vs `2TSLA`, or quarterly vs. weekly futures options
   classes) — default to the expiration with no special trading class, or
   ask the user which one they mean, if it's not obvious.
4. `get_option_data` with the chosen `expiration_id`, bounding `min_strike`/
   `max_strike` to roughly 5 strikes on either side of the spot price so the
   response stays manageable.
5. For prices/IV/greeks/open interest on specific strikes, feed the numeric
   `call_contract_id`/`put_contract_id` from that response into
   `get_price_snapshot` (using the chain response's top-level `exchange`
   field) — `get_option_data` itself only returns contract structure, not
   pricing.

For a multi-leg strategy (spread, combo) the user wants to *understand*
(not place), you can still use `get_combo_identifier` to get IBKR's
`strategy_name`/`description` for what the legs amount to (e.g. confirming
"yes, that's a bull call spread") — just don't follow it with
`create_order_instruction`.

## Workflow: probability / statistical projections

In scope: a user can ask quantitative questions like "what's the
probability this reaches price X within N months" or "what's a plausible
3-month range." Answering with a statistical model (e.g. a random-walk /
Monte Carlo simulation calibrated to historical volatility) is research,
not advice, as long as it stays a probability/range estimate and not a
directional call.

1. Get annualized volatility from the technical-analysis workflow (stdev of
   daily returns Γ— `sqrt(252)`), or compute it fresh from `get_price_history`
   daily bars if that wasn't already done.
2. Default to a **zero-drift (martingale) assumption** — i.e., simulate
   with no built-in bullish or bearish edge — unless the user asks for a
   specific drift/trend assumption. Zero-drift is the defensible neutral
   default; picking a drift yourself edges toward a directional call, which
   is out of scope.
3. **If the user names a directional scenario, don't silently model
   something else.** A request like "take the base case" or "assume the
   bullish scenario" refers to a directional view, but a zero-drift
   simulation encodes no direction at all — so the output does not answer
   the question they think they asked. Say so in one line ("the simulation
   is direction-neutral, so this isn't conditioned on the bounce scenario"),
   and either keep it neutral or, if they want the scenario priced in, ask
   what drift to assume rather than inventing one. Never let a
   scenario-framed question imply the model agreed with the scenario.
4. **State the starting price you simulated from, and flag it if it differs
   from live spot.** Users often quote a hypothetical entry ("if I buy at
   4.70") that sits away from the current price, and the gap moves the
   answer materially — for IAG, simulating from a 4.70 entry gave a 56.6%
   chance of touching 5.10 within three months versus 62.6% from the 4.77
   spot, with the median touch at 17 versus 13 trading days. Report which
   basis you used, and give both when the difference is meaningful.
5. Simulate a lognormal random walk (daily steps, `mu = -0.5 * sigma_daily^2`,
   `sigma_daily = ann_vol / sqrt(252)`) out to the requested horizon, and
   report:
   - Probability the path *ever touches* the target level (not just probability
     it's above/below the target exactly at the horizon end — these are
     different numbers and both are worth giving).
   - If it touches, the distribution of *when* (median/mean days, and a
     25th-75th percentile range) rather than a single point estimate.
6. **If the user gave a position size, carry it through to money.** When
   someone says "300 shares at 4.70", the share count is part of the
   question — translate the target into what it's worth (300 shares Γ— the
   €0.40 move to 5.10 = €120 on a ~€1,410 position) instead of answering
   only in percentages and dropping the size they specified. Keep this
   descriptive: converting a price move into currency is arithmetic on
   their own stated hypothetical, whereas judging whether the size is
   appropriate for their account is advice and stays out of scope.
7. Always state plainly: this is a statistical extrapolation of historical
   volatility under a no-edge assumption, not a fundamental forecast — it
   ignores earnings, macro events, and anything that would make future
   volatility or drift differ from the historical sample. Don't let the
   output read as a prediction of what the instrument will actually do.

## Workflow: thematic / competitor / sector context

1. Resolve the company's contract ID via `search_contracts`.
2. For "what sector/trend is X in" or "who are X's closest peers":
   `get_company_themes` — returns top themes plus ranked peers per theme.
3. For a fuller picture (products, competitors, geographic footprint) or
   when the user wants to know *why* a connection exists:
   `get_company_connections`, adding `include: ["link_info"]` for the
   supporting evidence and/or `link_types` to scope to just competitors,
   products, or geography.
4. To browse a theme itself (e.g. "what else is in this trend") rather than
   starting from a company, use `search_investment_topics` to find it, then
   `get_theme_details`.

## Output style

- Lead with the answer (price, trend, peer list), not the tool-call
  mechanics.
- Note the data's timeliness where it matters (a snapshot is a point in
  time; markets move).
- Flag clearly when something is IBKR-provided analysis/classification
  (themes, competitor links) versus a hard market data fact (price,
  volume) — they carry different confidence levels.
- This is research, not advice: don't tell the user what to do with their
  position or portfolio: presenting data and context is in scope, a
  buy/sell recommendation is not.
