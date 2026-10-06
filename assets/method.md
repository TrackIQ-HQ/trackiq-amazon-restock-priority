# Method

## The arithmetic

Everything is computed per **ASIN**, after rolling SKUs up (see
`assets/pulls.md`).

```
daily_units    = units in window / days in window
daily_revenue  = revenue in window / days in window

cover          = on_hand / daily_units                  # days, stock in FCs today
cover_inbound  = (on_hand + inbound) / daily_units      # days, counting the boat

runs_out       = today + cover_inbound
order_by       = runs_out - lead_time
                 ( = today + cover_inbound - lead_time )

days_uncovered = max(0, horizon - cover_inbound)
revenue_at_risk = daily_revenue x days_uncovered

units_to_ship  = max(0, daily_units x (target_cover + lead_time)
                        - (on_hand + inbound))
```

## Rank by revenue at risk, not by days of cover

Days of cover sorts the catalogue by panic, not by money. A slow SKU with 9
days of cover and $20/day of revenue belongs below a fast one with 30 days of
cover and $3,200/day — the second is worth a hundred times more and needs the
PO sooner in absolute dollars.

**Revenue at risk is the ranking.** Days of cover is a column.

## The four states

| State | Test | What it means |
|---|---|---|
| **Out now** | `on_hand == 0` and sales in window | already losing sales today |
| **Order overdue** | `order_by < today` | the PO needed placing already; some loss is now locked in |
| **Order now** | `order_by` within the next 14 days | the decision is live this week |
| **Healthy** | everything else | say nothing about it |

Report the first three. The fourth belongs in the table for completeness but
never in the summary or the email subject.

**"Order overdue" is the most useful line this skill produces** and the one a
client will argue with, so show the arithmetic beside it: cover, lead time,
and the date the order needed placing.

## Revenue at risk is an opportunity cost, not a loss

`revenue_at_risk` prices what the ASIN would have sold in the horizon at its
current rate, for the days it will be out of stock. Three things that means:

- It assumes the historical rate holds. A seasonal product going into its peak
  is understated; one coming out of peak is overstated. Say which applies if
  the category is obviously seasonal.
- For an ASIN **already** at zero, the window's velocity includes days it was
  in stock, so the daily rate is a blend. It is still the right basis for "what
  this is costing", but it is not a measured loss.
- It ignores substitution. A buyer who cannot get the 8.75oz may buy the
  17.6oz. Never present the total as money the brand definitely loses.

## Units to ship

`daily_units x (target_cover + lead_time) - (on_hand + inbound)`

Enough to cover the lead time *and* leave the target cover standing when the
shipment lands. Defaults: 60-day target, 45-day lead time.

This is a starting quantity, not a purchase order. It contains no case-pack
rounding, no MOQ, no storage-fee optimisation and no seasonality. Say so.

## What this skill does not do

- **No forecast.** Velocity is a trailing average, not a projection. Seasonality,
  promotions and launches all break it, and pretending otherwise would make the
  order-by dates look more precise than they are. Forecasting is a separate
  skill on the roadmap.
- **No supplier or PO logic.** No MOQs, case packs, container fill, or costs.
- **Nothing is ordered.** This produces a ranked proposal a human acts on.
