---
name: trackiq-amazon-restock-priority
description: Works out which Amazon products to ship next and in what order — days of cover at the current sales rate, the date each one runs out, the date a purchase order needed placing to beat it, the revenue exposed if it is late, and a starting shipment quantity per product. Ranked by revenue at risk rather than by days of cover. Use when the user asks about restocking, reordering, replenishment, days of cover, days of supply, what to ship, inventory planning, stockout risk, which products are running low, or when to place a purchase order.
---

# Restock Priority

Answers the Monday morning question: **what do we ship, in what order, and
what does being late cost?**

Output is a branded HTML report — the products needing a decision, then the
full catalogue, then the shipment quantities. Ranked by revenue at risk, so
the list is already in the order someone should work it.

## Requires

- The TrackIQ MCP, for `list_marketplaces`, `get_inventory_snapshot` and
  `get_product_performance`.
- **A lead time** — days from purchase order to sellable in an FC. Ask for it.
  It is in no tool, and without it the report can say when stock runs out but
  not when to order, which is the only part anyone acts on. Default 45 days,
  labelled an assumption.
- Nothing else. No filesystem, no shell, no internet.
- **Without the MCP:** works from an FBA inventory export plus units and
  revenue by ASIN for a trailing month.

## First run

Fill in a copy of `assets/account.example.md` saved as account.md beside the
skill. Every TrackIQ skill reads the same file, so an account already set up
for another TrackIQ report needs nothing added here.

If the runtime has no filesystem, print the same block and ask the user to
paste it into their project instructions once.

## Read first

- `assets/pulls.md` — the calls, the filtering, and the ASIN-vs-SKU trap.
  Read before writing any join.
- `assets/method.md` — the arithmetic, the four states, and what this
  deliberately does not do
- `assets/checks.md` — what to verify before anything is sent

Copy `assets/report-template.html` and replace every `{{TOKEN}}`.

## Non-negotiables

1. **Compute cover per ASIN, never per SKU.** Inventory and sales both arrive
   per SKU, so a per-SKU join looks obvious and is wrong. On the account this
   was built against, one ASIN had 11,032 units on a SKU that sold nothing and
   2,178 units on the SKU doing 11,861 units a month. Per SKU that reads as a
   six-day emergency *and* an infinite supply on the same product; the truth at
   ASIN level is 33 days. Roll both sides up to ASIN, then divide.
2. **Cover uses `on_hand`. Never `total`.** `total` is
   `on_hand + inbound + reserved + unfulfillable + researching` — it counts
   units on a boat, promised to orders, damaged, and lost. Using it inflates
   every cover figure on the page.
3. **`reserved` is never available.** Those units are already attached to
   orders. Keep them out of the arithmetic entirely.
4. **Rank by revenue at risk, not days of cover.** Days of cover sorts by
   panic. A product with 30 days of cover at $3,200/day outranks one with 9
   days at $20/day, and a report sorted the other way sends people to the wrong
   PO first.
5. **Paginate the inventory pull.** It caps at 100 rows; a real catalogue
   exceeds it.
6. **Filter parents, FBM shadows and dead rows — but never a SKU with sales
   and no stock.** That row is the most urgent one on the page. State how many
   rows were filtered.
7. **Lead time is stated on the report,** with whether it came from the client
   or is the default, and what changes if it is wrong.
8. **Revenue at risk is an opportunity cost, not a loss.** It assumes the
   trailing rate holds and ignores substitution between sizes. Never present
   the total as money certainly lost.
9. **Units to ship is a starting quantity.** No MOQ, no case pack, no
   container fill, no storage-fee optimisation. Say so beside the number.
10. **No forecasting.** Velocity is a trailing average. Seasonality,
    promotions and launches break it — flag an obviously seasonal category
    rather than implying the dates are precise.
11. **Nothing is ordered.** This is a ranked proposal a human acts on.
12. **Never print `account_id`.**

## Vendor versus seller

`list_marketplaces` returns `account_type`. Seller accounts use
`get_inventory_snapshot`; vendor accounts have `get_vendor_inventory_health`
and `get_vendor_forecasting` instead and a different fulfilment model. Branch
on the type rather than calling and hoping.

## What it pairs with

`trackiq-listing-monitor` catches an ASIN that has already gone unbuyable;
this one is meant to make that alert unnecessary. The Inventory Risk → Ad
Spend Throttle skill takes this skill's days-of-cover figures and finds the
campaigns still paying to advertise them.

## Delivery

The output is produced in the chat first. Delivery is the last step and the
method comes from the Delivery block in account.md — never ask per run.

| Method | What to do | Needs |
|---|---|---|
| `in-chat` | Return the report. The default, and the fallback for every other method. | nothing |
| `file` | Write it beside the skill, dated. | a filesystem |
| `slack` | Post the headline findings as text, then upload the file. | a connected Slack tool |
| `n8n` | POST it to the configured webhook. | network access |
| `email` | Hand it to the connected mail tool. | a connected mail tool |

Confirm before the first outward send of a session, fall back to in-chat
loudly when a method is unavailable, and never substitute a different
outward channel.

## Version

`trackiq-amazon-restock-priority` v1.0.0 (2026-09-18).

If the user asks whether this skill is current, fetch
`https://trackiq.com/skills/registry.json`, compare the `version` field for
`trackiq-amazon-restock-priority`, and if it is newer, give them the download link
and the one-line changelog. Do not fetch at any other time.
