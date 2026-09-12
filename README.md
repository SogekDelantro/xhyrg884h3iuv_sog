# IBKR Trading Assistant

A set of Claude Code skills that turn a Claude session connected to the
[Interactive Brokers (IBKR)](https://www.interactivebrokers.com/) MCP
connector into a trading assistant: market research, portfolio/risk
monitoring, alerts and watchlists, and order drafting.

## What's here

- [`.claude/skills/ibkr-market-research/`](.claude/skills/ibkr-market-research/SKILL.md) —
  a skill for quote/price lookups, price history, option chain analysis, and
  thematic/competitor research on companies, using the IBKR MCP tools
  already available in a connected session (`search_contracts`,
  `get_price_snapshot`, `get_price_history`, `get_option_parameters`,
  `get_option_data`, `get_company_themes`, `get_company_connections`,
  `search_investment_topics`, `get_theme_details`, and futures equivalents).
- [`.claude/skills/ibkr-portfolio-monitor/`](.claude/skills/ibkr-portfolio-monitor/SKILL.md) —
  a skill for account status checks: positions, P&L, cash/margin health,
  concentration and currency exposure, performance, and recent
  orders/trades, using `get_account_summary`, `get_account_positions`,
  `get_account_balances`, `get_account_orders`, `get_account_trades`,
  `get_pa_allocation`, and `get_pa_performance_all_periods`.
- [`.claude/skills/ibkr-alerts-watchlists/`](.claude/skills/ibkr-alerts-watchlists/SKILL.md) —
  a skill for creating, viewing, editing, pausing, and deleting price/
  volume/margin-cushion/daily-P&L alerts, and for creating, viewing,
  editing, and deleting watchlists, using `create_alert`, `get_alerts`,
  `get_alert`, `update_alert`, `set_alert_status`, `delete_alert`,
  `create_watchlist`, `get_watchlists`, `get_watchlist`, `edit_watchlist`,
  and `delete_watchlist`.
- [`.claude/skills/ibkr-order-drafting/`](.claude/skills/ibkr-order-drafting/SKILL.md) —
  a skill for preparing stock, futures, single-leg option/futures-option,
  and OPT–OPT spread trades via `create_order_instruction` (plus
  `get_combo_identifier`, `get_order_instructions`, and
  `delete_order_instruction`). It never executes a live trade — every draft
  comes with a review URL the user must open and submit themselves in IBKR
  Desktop/Mobile/TWS/Client Portal.

## Scope

The research and portfolio-monitor skills are **read-only**: they never
place, modify, or cancel an order, and never create alerts or watchlists.
They answer "what's this instrument doing" and "where does my account
stand" — not "what should I trade."

The alerts/watchlists skill mutates account state (it's the whole point),
but only alerts and watchlists — never orders. It always confirms before
creating, editing, or deleting anything.

The order-drafting skill goes one step further into trade mechanics, but
stays draft-only by construction: `create_order_instruction` creates an
instruction, not a live order, and every draft requires the user's own
review and explicit submission on an IBKR platform to become a real trade.
None of these skills give buy/sell recommendations — they execute or report
on decisions the user has already made.

## Requirements

Use these from a Claude Code session (CLI, desktop, or web) that has the
`Interactive_Brokers_IBKR` MCP connector attached to an IBKR account. No
separate install or server is needed — skills are picked up automatically
from `.claude/skills/` once this repo is your working directory (or once
they're copied/symlinked into a project that is).
