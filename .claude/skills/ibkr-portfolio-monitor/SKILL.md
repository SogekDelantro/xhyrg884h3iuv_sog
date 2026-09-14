---
name: ibkr-portfolio-monitor
description: Use when the user wants a status check on their IBKR account — positions, P&L, cash/margin health, concentration or currency exposure, recent orders/trades, or overall performance. Triggers on "how's my portfolio", "what am I holding", "check my positions", "margin cushion", "am I over-leveraged", "how did I do this month/YTD", "what orders are open", "recent trades", or any question about the state of the connected IBKR account. Read-only — does not place, modify, or cancel orders, and does not give buy/sell advice.
---

# IBKR Portfolio & Risk Monitor

Read-only status-check assistant for the connected IBKR account, built on
the `Interactive_Brokers_IBKR` MCP connector. It answers "where do things
stand" — holdings, exposure, cash/margin health, performance, recent
activity — not "what should I do about it."

## Hard rules

- Never call `create_order_instruction`, `create_alert`, `create_watchlist`,
  or any other write/mutating IBKR tool from this skill.
- Don't give trade recommendations (buy/sell/hedge this position). Surface
  the numbers and flag risk conditions the user should be aware of; let
  them decide what to do.
- Account data is sensitive. Don't copy full position/balance dumps into
  anything outside this conversation (files committed to a repo, external
  services, etc.) unless the user explicitly asks you to save it somewhere.
- This account trades multiple currencies. Never combine figures across
  currencies without converting first — see "Multi-currency" below.

## Core tools

| Tool | Use for |
|---|---|
| `get_account_summary` | Net liq, buying power, margin (initial/maintenance), available funds, excess liquidity, leverage. The single best "how healthy is this account" call. |
| `get_account_positions` | Every open position: quantity, price, market value, average cost, unrealized P&L, **daily P&L**, asset class, currency. |
| `get_account_balances` | Cash and net liquidation value broken down **by currency**, plus a `BASE` row already converted to the account's base currency. |
| `get_account_orders` | Live/working orders (not filled, not cancelled). |
| `get_account_trades` | Executed trades over a period (`TODAY` default, or `DAYS_7/30/60/90`, `MONTH_TO_DATE`, `YEAR_TO_DATE`, quarters). |
| `get_pa_allocation` | NAV broken down by `ASSET_CLASS`, `SECTOR`, `REGION`, `COUNTRY`, or `FINANCIAL_INSTRUMENT` (or `ALL` for every dimension at once) — long and short buckets separately. Use for concentration questions. |
| `get_pa_performance_all_periods` | Time-weighted or money-weighted return series for 1D/7D/MTD/1M/YTD/1Y in one call. |

## Workflow: general "how's my account" check

1. `get_account_summary` for the headline health numbers.
2. `get_account_positions` for the holdings detail behind those numbers.
3. Skim for risk flags (see below) and lead with anything notable before
   the full rundown.

Don't dump every field from every tool by default — summarize, and offer to
go deeper (e.g. "want the full position list?") if the user's question was
broad.

## Workflow: risk flags to check and call out

- **Margin cushion**: `excess_liquidity / net_liquidation` from
  `get_account_summary` (or ask the user's threshold). Low cushion means
  the account is closer to a margin call — worth flagging proactively even
  if not asked.
- **Leverage**: the `leverage` field in `get_account_summary` (gross
  position value ÷ equity). Above ~2x is aggressive for most retail
  accounts — use judgment and the user's own stated risk tolerance rather
  than a hard rule.
- **Negative cash / borrowing**: a negative `cash_balance` or
  `total_cash_value` means the account is borrowing to hold positions —
  check this is intentional.
- **Concentration**: `get_pa_allocation` with `type: "SECTOR"` or
  `"FINANCIAL_INSTRUMENT"` — a single name or sector dominating NAV is
  worth surfacing. Careful with this tool's `currency` field: it reports
  `"USD"` on a default call, but the `nav` figures come back denominated in
  the account's **base** currency, not converted to USD — the label is
  wrong, the numbers are base. So when the base currency isn't USD, taking
  the label at face value and converting would corrupt every figure. Don't
  assume either way: reconcile before relying on absolute `nav` values —
  the long stocks + ETFs `nav` should equal `get_account_balances`' `BASE`
  row `stock_market_value`, which tells you directly what currency you're
  holding. The `weight` fractions are currency-agnostic and safe to use as-is,
  so prefer them whenever percentages answer the question.
- **Currency exposure**: `get_pa_allocation` with `type: "COUNTRY"`/
  `"REGION"`, or just eyeball the per-currency rows from
  `get_account_balances` — large exposure to a currency the user didn't
  mean to hold is a common surprise.
- **Short positions / naked options**: in `get_account_positions`, a
  negative `position` on an `OPT` row is a short option — note the
  undefined/large risk profile if the user doesn't seem aware of it.

Only lead with a flag if it's actually notable for this account — don't
manufacture concern where the numbers are unremarkable.

## Workflow: performance review

1. `get_pa_performance_all_periods` for the period the user cares about
   (1D/7D/MTD/1M/YTD/1Y are all returned in one call — just read the right
   key).
2. `cps` values are cumulative **fractions** from the period start (e.g.
   `-0.106` = -10.6%) — convert to a percentage when presenting.
3. Note whether the account reports `TWR` (time-weighted, strips out the
   effect of deposits/withdrawals) or `MWR` (money-weighted, includes it) —
   it's in the top-level `portfolio_measure` field and changes how the
   number should be interpreted.
4. For a "how does that compare to the market" question, pair with
   `get_price_history` on a relevant benchmark (e.g. SPY) — that tool
   belongs to the market-research skill; feel free to reach for it here
   too.

## Workflow: orders and recent activity

- "What's still open / pending" → `get_account_orders`.
- "What did I trade this week/month/quarter" → `get_account_trades` with
  the matching `period`. Note all period boundaries are UTC.

## Multi-currency handling

`get_account_balances` returns one row per currency plus a `BASE` row
already converted to the account's base currency (identify the base
currency from `get_account_summary`'s `currency` field or the `BASE` row
itself). When asked for a total, use the `BASE` row rather than summing
raw per-currency numbers. When asked to break down exposure by currency,
use the per-currency rows as-is — those are not converted, by design.

## Output style

- Lead with the headline (net liq, today's P&L, the specific thing asked
  about), then supporting detail.
- Group positions by something meaningful when listing them (asset class,
  currency, or biggest movers) rather than dumping the raw API order.
- State risk flags plainly and neutrally — this is information, not a
  verdict on the user's choices.
