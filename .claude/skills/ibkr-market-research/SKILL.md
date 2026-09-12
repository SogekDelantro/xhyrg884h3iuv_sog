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

If a symbol is ambiguous (multiple listings, multiple countries/exchanges,
or a name that maps to several companies), ask the user to disambiguate
rather than guessing — a wrong contract ID silently produces answers about
the wrong instrument.

Never show raw `contract_id` / `contract_id_ex` / `expiration_id` values to
the user — they're internal plumbing for chaining calls. Present results by
symbol, company name, expiry date, and exchange instead.

## Workflow: quote / snapshot

1. Resolve the contract ID (Step 0).
2. `get_price_snapshot` for the live/last price, bid/ask, volume, etc.

Use this for "what's X trading at", "what's the bid/ask on this option", or
as a sanity check before discussing a price level with the user.

## Workflow: price history / chart

1. Resolve the contract ID (Step 0).
2. `get_price_history` with an appropriate lookback/bar size for what's being
   asked (a "how has it done this year" question needs daily bars over
   months, not minute bars).
3. Summarize the trend/range/volatility in prose; only dump raw bar data if
   the user explicitly wants a table.

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
