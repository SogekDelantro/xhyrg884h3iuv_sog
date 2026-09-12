---
name: ibkr-alerts-watchlists
description: Use when the user wants to create, view, change, pause, or delete a price/volume/margin/P&L alert, or create, view, edit, or delete a watchlist, on the connected IBKR account. Triggers on "alert me when", "notify me if", "let me know when X hits", "watchlist", "add X to my watchlist", "track these symbols", "pause/delete that alert". This skill mutates account state (unlike the read-only research/portfolio skills) — always confirm before creating, editing, or deleting anything.
---

# IBKR Alerts & Watchlists

Manages price/condition alerts and watchlists on the connected IBKR account
via the `Interactive_Brokers_IBKR` MCP connector. Unlike the market-research
and portfolio-monitor skills, the tools here **write** to the account —
treat every create/edit/delete as an action that needs the user's explicit
go-ahead, not just their initial request.

## Hard rules

- Never call `create_order_instruction` or any other order-placement tool
  from this skill — alerts and watchlists are not orders.
- Before creating an alert or watchlist: state exactly what you're about to
  create (symbol, condition, threshold, notification method / instrument
  list) and get confirmation if there's any ambiguity. A precise, unambiguous
  request ("alert me when AAPL hits $200") doesn't need a second
  confirmation round-trip — just state what you're creating as part of doing
  it.
- Before deleting an alert or watchlist: always confirm first and show what
  will be deleted (name/symbol) — both `delete_alert` and `delete_watchlist`
  are irreversible.
- Before editing a watchlist: `edit_watchlist` is a **full replace**, not an
  add/remove diff. Always call `get_watchlist` first to fetch the current
  name + full instrument list, then submit the complete new list. Never
  guess at current contents.
- Before editing an alert: `update_alert` is also a **full replace** of that
  alert's settings. Always call `get_alert` first and carry forward every
  field the user isn't explicitly changing (tif, expiry_date, exchange,
  email, email_note) — a naive update will silently clear them.

## Workflow: creating an alert

1. Resolve the instrument's contract ID via `search_contracts` (or
   `search_futures`) unless the condition is account-level (see below).
2. Map the user's request to a `condition_type` + `operator` + `value`:
   - "alert when price goes above/hits X" → `LAST` (or `BID_ASK` if they
     specifically mean bid/ask), `GTE`, value = X.
   - "alert when it drops below X" → `LTE`, value = X.
   - "alert on volume over X" → `VOLUME`.
   - "alert on % move" → `PERCENT_CHANGE` (value is a percentage number,
     e.g. `5` for 5%, not `0.05`).
   - "alert if my margin cushion drops below X%" → `MARGIN_CUSHION`, no
     `contract_id`/`exchange` needed (account-level).
   - "alert if I'm down X today" → `DAILY_PNL`, account-level.
   - **MARGIN_CUSHION and DAILY_PNL only support percentage thresholds.**
     If the user gives a dollar amount for either of these, don't convert
     it yourself — tell them these condition types are percentage-only and
     ask them to restate the threshold as a percentage.
3. Ask about email notification if the user didn't specify one:
   **without an email, the alert only surfaces inside IBKR Desktop** — no
   push, SMS, or other notification. If they want to be notified anywhere
   else (mobile, email inbox), they need to supply an email.
4. Always tell the user, before or immediately after creating the alert,
   that alerts created this way are **only visible/manageable in IBKR
   Desktop** — they will not appear in IBKR Mobile, TWS, or Client Portal.
   This is a platform limitation worth surfacing every time, not just once.
5. Call `create_alert`.

## Workflow: viewing / managing alerts

- List alerts: `get_alerts` (id, name, status — ACTIVE or PAUSED).
- Full detail on one: `get_alert` with its id. Do this before `update_alert`
  (see full-replace rule above).
- Pause/resume without deleting: `set_alert_status` with `action: "PAUSE"`
  or `"RESUME"`.
- Delete: `delete_alert` — confirm first, irreversible.

## Workflow: creating a watchlist

1. Resolve each instrument via `search_contracts` (stocks/options
   underlyings — use `underlying_contract_id`, stringified) or
   `search_futures` (`contract_id_ex` verbatim) or `get_option_data`
   (`call_contract_id_ex`/`put_contract_id_ex` verbatim for a specific
   option contract).
2. Confirm the name and full instrument list with the user before calling
   `create_watchlist` if there's any ambiguity in what they asked for.
3. `contract_id_ex` values are always JSON **strings**, even when they look
   numeric — `"8314"`, never the bare number `8314`.

## Workflow: viewing / editing a watchlist

1. `get_watchlists` to list all watchlists and resolve a name to an `id`.
   **Names are not guaranteed unique** — this account currently has two
   different watchlists both named "Favorites" (different ids). If the
   user's name is ambiguous, show them the candidates (id + name) and ask
   which one, rather than guessing.
2. `get_watchlist` with the id for the full current instrument list
   (`contract_id_ex` + `contract_description` per row) and to confirm the
   current name.
3. To add/remove instruments: take the list from step 2, apply the user's
   change, and submit the **complete** new list via `edit_watchlist` along
   with the id and name (both required even if unchanged).
4. To rename only: same call, with the instruments list unchanged from
   step 2.

## Workflow: deleting a watchlist

1. `get_watchlists` to resolve the name (disambiguating duplicates as
   above) to an id.
2. Confirm with the user, showing the watchlist's name.
3. `delete_watchlist`.

## Output style

- After creating/editing/deleting, confirm plainly what happened (symbol,
  condition/threshold, or watchlist name and contents) — don't just say
  "done."
- Never show raw `contract_id`/`contract_id_ex`/`id` values to the user;
  refer to instruments by symbol/description and to watchlists by name.
- For alerts, always mention the IBKR Desktop-only visibility caveat and,
  when no email was given, the no-notification-elsewhere caveat.
