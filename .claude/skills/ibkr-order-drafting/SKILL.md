---
name: ibkr-order-drafting
description: Use when the user wants to prepare a stock, futures, single-leg option/futures-option, or option spread trade on IBKR — "buy X shares of Y", "sell to close", "draft an order for", "set up a limit order on", "put together a call spread on". Produces a draft instruction the user reviews and submits themselves in IBKR Desktop/Mobile/TWS/Client Portal — this skill never executes a live trade on its own. Does not give buy/sell recommendations.
---

# IBKR Order Drafting

Prepares trade instructions on the connected IBKR account via
`create_order_instruction`. **This tool does not place a live order.** Per
its own description: "Creates a new instruction. An instruction is not a
live order... After submission, the instruction is converted into a live
order." The call returns a URL that deep-links into IBKR Desktop/Mobile/TWS
where the user reviews and explicitly submits it. That review step is not
optional and not a formality — it's the entire safety model this skill
relies on.

## Hard rules

- **Never say or imply a trade has been placed, executed, or is live.** Say
  "drafted," "prepared," or "set up an instruction for you to review" — not
  "bought," "sold," "placed an order," or "executed."
- Always give the user the returned review URL and tell them explicitly
  that they still need to open it and submit it for the trade to actually
  happen.
- This skill executes decisions the user has already made. It clarifies
  mechanics (order types, what a limit price does, combo structure) but
  does not recommend direction, sizing, strikes, or timing. If asked "should
  I buy this" / "is this a good trade," decline to advise and offer to draft
  whatever the user decides instead.
- Confirm the full trade in plain language before calling
  `create_order_instruction` — symbol, side, quantity, order type, limit
  price (if any), time in force. Even though it's a draft rather than a
  live trade, it's a real object created in the account, and a
  misread quantity or side is exactly the kind of mistake confirmation
  catches. Skip re-confirming only when the user's own request was already
  fully specified and unambiguous.
- Double check side (BUY/SELL) against what the user actually wants,
  especially "sell to close" / "buy to close" phrasing on an existing
  position — cross-check against `get_account_positions` (portfolio-monitor
  skill) if there's any doubt about direction or whether they're opening or
  closing.

## What's supported

- Stocks (STK)
- Futures (FUT)
- Single-leg options (OPT) and single-leg futures options (FOP)
- OPT–OPT option combos/spreads only (both legs must be standard equity
  options from an OPT chain)

**Not supported:** multi-leg combos involving FOP, FUT, or STK legs, or any
mixed-type combo. If the user wants a futures-options spread, say plainly
that multi-leg FOP combos aren't supported here, and offer to draft each leg
as a separate single-leg FOP instruction instead.

## Step 1: resolve the contract

Reuse the resolution approach from the market-research skill:

- **STK**: `search_contracts`, exact `symbol` match, disambiguate by
  `description`/`country_code` if needed. Use `underlying_contract_id`,
  **stringified**, as `contract_id_ex`.
- **FUT**: `search_futures` — use its `contract_id_ex` verbatim (already
  includes the exchange, e.g. `"12345@CME"`).
- **OPT/FOP** (single leg): `get_option_parameters` → `get_option_data` for
  the right expiration/strike, then use `call_contract_id_ex` or
  `put_contract_id_ex` verbatim as `contract_id_ex`.
- **OPT–OPT combo/spread**: get each leg's `contract_id_ex` from
  `get_option_data` (equity options only), then call `get_combo_identifier`
  with each leg's `contract_id_ex` and `size` (positive for BUY, negative
  for SELL) to get a combo `contract_id_ex`. Its `description` and
  `strategy_name` are useful to read back to the user to confirm you built
  the spread they meant (e.g. confirming it's actually a "Bull" call
  spread) before drafting the order.

`contract_id_ex` is always a JSON **string**, even when it looks numeric
(e.g. `"8314"`, never the bare number). Never surface raw contract IDs to
the user — refer to instruments by symbol/description/expiry/strike.

## Step 2: confirm the order parameters

- `side`: `BUY` or `SELL` (uppercase, case-sensitive).
- `quantity`: shares for STK; contracts for FUT/OPT/FOP/combos.
- `order_type`: `MARKET` or `LIMIT`. If `LIMIT`, `limit_price` is required —
  ask for it if the user didn't give one; don't invent a price.
- `time_in_force`: `DAY`, `GTC`, `OVT`, `OND`, or `OPG`. If the user doesn't
  specify, it's fine to omit and let IBKR apply its default — but say
  you're doing that, don't silently pick one on their behalf if they seem
  to care about timing.

Read the full order back to the user in plain language before submitting
the draft: e.g. "Buy 10 AAPL @ market, day order" or "Sell to open 1 TSLA
Jan 2028 $250 call, limit $12.50, GTC."

## Step 3: create the draft

Call `create_order_instruction`. Report back:

- What was drafted, in the same plain-language form as the confirmation.
- The returned review URL, with an explicit instruction to open it and
  submit it in IBKR Desktop/Mobile/TWS/Client Portal — this skill does not
  and cannot submit it for them.

## Managing existing drafts

- List drafts not yet submitted: `get_order_instructions` (id, description,
  contract, side, quantity, order type, limit price, time in force,
  creation/expiration times). Don't confuse this with `get_account_orders`
  (portfolio-monitor skill), which lists already-**live** orders — drafts
  from this skill are a separate, pre-submission list.
- Cancel a draft the user no longer wants: `delete_order_instruction` with
  its id. Confirm which draft (by description) before deleting —
  irreversible.

## Output style

- Plain-language trade description first, mechanics/ids never shown.
- Always pair a newly created draft with the review URL and the
  not-yet-live caveat, every time — this is the one caveat that must never
  be dropped for brevity.
