# Before you send it

## 1. The join

- **Cover is computed per ASIN, not per SKU.** Spot-check any ASIN that has
  more than one SKU: does its cover match the summed stock over the summed
  units? If a single-SKU cover leaked through, the report will show a false
  emergency and a false all-clear on the same product.
- Where an ASIN has several SKUs, the SKU split is shown underneath, and any
  SKU holding stock while selling nothing is called out.
- Every ASIN with sales in the window appears, including ones at zero stock.

## 2. The stock figures

- **Cover uses `on_hand`, never `total`.** `total` includes reserved, inbound,
  unfulfillable and researching units. If the report's cover figures look
  generous, this is the first thing to check.
- `reserved` is nowhere in a cover calculation.
- Unfulfillable units are reported as one catalogue-level number, not mixed
  into availability.
- Parents, FBM shadow SKUs, GUID ASINs and dead listings are filtered, and the
  count filtered out is stated.
- The inventory pull was paginated to exhaustion — it caps at 100 rows.

## 3. The inputs

- **Lead time is stated on the report**, with whether it came from the client
  or is the 45-day default.
- Target cover and horizon are stated.
- If lead time is a default, the report says what changes if it is wrong.

## 4. The arithmetic

- Ranking is by revenue at risk, not days of cover.
- `order_by = runs_out - lead_time`. Dates in the past are flagged **overdue**,
  not quietly shown as a date.
- Revenue at risk is labelled an opportunity cost, never a loss, and the
  substitution caveat appears.
- Units to ship is labelled a starting quantity — no MOQ, case pack or
  seasonality is claimed.
- No ASIN shows negative units to ship.

## 5. Sanity

- Every "order overdue" row shows its cover, the lead time and the date, so the
  claim can be audited on the page.
- An ASIN with 200+ days of cover is not in the summary.
- Total units to ship is plausible against the catalogue's monthly volume — if
  it exceeds about three months of total units, the target cover or lead time
  is probably wrong.

## 6. Render check

```js
// Scope the state counts to the ACTION table. Every action row also appears in
// the full-catalogue table, so a document-wide count doubles them.
const action = document.querySelectorAll('table')[0];
({ overflows: document.documentElement.scrollWidth > document.documentElement.clientWidth,
   tables: document.querySelectorAll('table').length,
   rows: [...document.querySelectorAll('table')].map(t => t.querySelectorAll('tbody tr').length),
   logos: [...document.images].map(i => i.naturalWidth > 0),
   overdue: action.querySelectorAll('.state-overdue').length,
   out: action.querySelectorAll('.state-out').length })
```

`overflows` false, `logos` all true, and `overdue` + `out` matching the summary
tiles and summing to the action table's row count. Then look at it; if it will
not paint, say the check was structural.

## 7. Ship

Save as `<client>-restock-priority-<YYYY-MM-DD>.html`. Dated by run day —
inventory is a snapshot and a week-old file is misleading without its date.

Offer the shipment quantities as a plain list the client can take to their 3PL
or supplier. That list is what they do next.
