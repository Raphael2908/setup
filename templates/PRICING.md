# <PRODUCT> — Pricing & Margin Model

> Only needed if <PRODUCT> is usage-metered. Derives credit pricing from **real vendor cost** so every
> billable unit clears a target gross margin (architecture.md §14). The numbers live in code at
> `backend/app/services/credits.py`; this file is the rationale + sources. Re-run the margin check
> whenever a vendor rate changes.

## 1. Marginal vendor cost per <unit>

The dominant cost is <the expensive step>; everything else is small. Rates as of <month/year>.

| Step | Vendor | Rate | Per <unit> |
|---|---|---|---|
| <step> | <vendor> | <published rate> | **$<x>** |
| ... | | | |

**Total marginal cost per <unit>:** ≈ **$<total>**.

<If cost scales with a dimension (length, size, count), note it and meter that step per-unit.>

## 2. From cost to credit price (the margin rule)

For target gross margin `m`, list price = `cost / (1 − m)`. At `m = <0.70>`:

```
price = cost / <0.30>   →   $<cost> / <0.30> = $<price>
```

We bill in **credits**, where **1 credit = $<0.10> of list** (`CREDIT_USD_VALUE`). The per-step credit
map is calibrated so each step holds the target margin, so the whole unit holds it regardless of size:

| Step | Credits | Notes |
|---|---|---|
| <step> | <n> | <per-unit or flat> |
| compose / finalize | 1 | |

Worked example — <unit>: `<sum> credits = $<list>`, cost $<cost> → margin **<%>**. Verify across the
range:

| Variant | Credits | List | Cost | Margin |
|---|---|---|---|---|
| <small> | | | | |
| <large> | | | | |

## 3. Subscription tiers & packs

| Plan | Price | Credits / mo | ≈ Units | Notes |
|---|---|---|---|---|
| Starter | $<29> | <300> | ~<n> | |
| Pro | $<99> | <1000> | ~<n> | |
| Studio | $<299> | <3000> | ~<n> | |

**Top-up pack:** <n> credits for $<x>. **Trial pack** *(optional CAC line)*: $1 / <n> credits, one per
customer — the cheapest *complete* unit, treated like ad spend, not a profit centre.

## 4. Recalibrating

When a vendor changes price: update `VENDOR_COST_USD` in `credits.py`, then adjust only the credit map
(usually the dominant step) to hold target margin. Sanity-check with `credits.<unit>_margin(...)` ≥
`TARGET_MARGIN`. Workers persist realized cost on `jobs.cost_credits` — compare modelled vs actual and
re-tune if they drift.

## Sources
- <vendor pricing page links used to derive the rates above>
