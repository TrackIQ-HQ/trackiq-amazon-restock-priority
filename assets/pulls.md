# The pull sequence

## 0. Account and inputs

`list_marketplaces` first — several TrackIQ MCPs can be connected with
identical tool names. Never print `account_id`.

Then **ask for the lead time** before computing anything. It is days from
placing a purchase order to units being sellable in an FBA fulfilment centre,
and it is not in any tool. Without it the report can say when stock runs out
but not when to order, which is the only part anyone acts on.

If the user does not know it, use 45 days, label it an assumption everywhere,
and state what changes if it is wrong. Two other inputs, both with defaults:

| Input | Default | What it is |
|---|---|---|
| Lead time | 45 days | PO to sellable |
| Target cover | 60 days | stock to hold once a shipment lands |
| Horizon | 90 days | how far ahead revenue at risk is priced |

## 1. Inventory

```
get_inventory_snapshot(account_id, limit=100, offset=…)
```

**Paginate.** It caps at 100 rows and a real catalogue exceeds that — the
account this was built against returned 146. Page until a call returns fewer
rows than the limit.

Fields: `on_hand`, `inbound`, `reserved`, `unfulfillable`, `researching`,
`total`, plus `out_of_stock` (0/1), `asin`, `sku`, `title`.

**`total` is not available stock.** It is
`on_hand + inbound + reserved + unfulfillable + researching`, so it includes
units that are on a boat, already committed to orders, damaged, or lost in a
warehouse. Using it as cover overstates every figure on the page.

- **Available now** = `on_hand`
- **Available with inbound** = `on_hand + inbound`
- `reserved` is already promised to orders — never add it to cover
- `unfulfillable` is dead stock; report the total separately and move on

## 2. Filter the rows

A real snapshot is mostly noise. Drop:

- rows with `total == 0` **and** no sales in the window — dead listings
- variation **parents** — SKU contains `parent`, `-P`, `Set`, or the title is
  the generic family name with no stock
- FBM shadow SKUs — `_FBM` suffix, zero stock, null title
- rows with a null ASIN, or an ASIN that is a GUID rather than a B0… code

Keep anything with stock **or** sales; a SKU with sales and zero stock is the
most urgent row on the page, not one to filter out.

## 3. Velocity

```
get_product_performance(account_id, start_date, end_date,
                        group_by='product', limit=200)
```

A trailing **30 days**. Shorter is too noisy per SKU; longer misses a trend.
Returns `units`, `revenue`, `orders`, `sessions` per product row with `asin`
and `sku`.

`daily_units = units / days_in_window`, `daily_revenue = revenue / days`.

**Use units for cover and revenue for ranking.** Cover is a unit question;
priority is a money question.

## 4. Join on ASIN, not SKU — the trap

Inventory and sales both come back per SKU, so a per-SKU join looks obvious.
It is wrong, and it fails loudly in both directions.

On the account this was built against, ASIN `B0C848NYN6` had two SKUs:

| SKU | on hand | units / 30d | SKU-level cover |
|---|---|---|---|
| `MDMID376US` | 11,032 | 0 | never runs out |
| `6782-12Count` | 2,178 | 11,861 | **6 days** |
| **ASIN total** | **13,210** | **11,861** | **33 days** |

A per-SKU report raises a six-day emergency on the brand's biggest seller and
simultaneously reports 11,032 units as healthy. Neither is true. **Sum both
inventory and units to ASIN level, then compute cover.**

Show the SKU split underneath, because stock sitting on a dormant SKU while
another sells is a real operational problem worth naming — but never let it
drive the cover figure.

## 5. Optional context

- `get_bsr` — for a note on whether a low-cover ASIN is also climbing
- `get_vendor_inventory_health` / `get_vendor_forecasting` — **vendor accounts
  only.** `list_marketplaces` returns `account_type`; branch on it rather than
  calling and hoping.
