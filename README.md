# IBKR Trading Assistant

A Claude Code skill that turns a Claude session connected to the
[Interactive Brokers (IBKR)](https://www.interactivebrokers.com/) MCP
connector into a market research assistant.

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

## Scope

Both skills are **read-only**: neither places, modifies, or cancels an
order, and neither creates alerts or watchlists. They answer "what's this
instrument doing" and "where does my account stand" — not "what should I
trade."

## Requirements

Use it from a Claude Code session (CLI, desktop, or web) that has the
`Interactive_Brokers_IBKR` MCP connector attached to an IBKR account. No
separate install or server is needed — the skill is picked up automatically
from `.claude/skills/` once this repo is your working directory (or once
the skill is copied/symlinked into a project that is).

## Roadmap

Not included yet, but natural follow-ups as separate skills:

- Alerts and watchlist management
- Order drafting with a mandatory human-confirmation step before submission
