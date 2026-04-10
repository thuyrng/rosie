# DWH | nguyenthelam | sale_order_ytd

## Overview

Order header table for Online/Omni-Channel POS system. One row per order ticket. Covers last 2 years data — for full history use nguyenthelam.fact_sale_order.

---

## Columns

---

## Scale & Cardinality

- ~10,000 orders/day, ~3M rows a year
- One order_code → 1 to ~20 rows in sale_order_item_ytd
- site_code has ~60 distinct values across all regions
---

## Rules & Gotchas

> ⚠️ Never use COUNT(DISTINCT order_code) for ticket count

> Always use SUM(num_tickets) instead.

> Reason: Some orders contain only a shipping fee line — not a qualified sale ticket. Counting distinct order_code overcounts real tickets.

- Always join to brand_site via site_code = siteid to get region, sitename
- sale_date should always be in the WHERE clause — table is large, date filter is critical for performance
- region column exists on this table but prefer pulling region from brand_site for consistency with other tables
---

## Sample Query Pattern

```sql
SELECT
    y.sale_date,
    bs.sitename        AS site_name,
    bs.region,
    SUM(num_tickets)   AS num_ticket,
    SUM(gross_sales)   AS gross_sales,
    SUM(net_sales)     AS net_sales,
    SUM(gold_margin)   AS margin
FROM nguyenthelam.sale_order_ytd y
JOIN nguyenthelam.brand_site bs ON y.site_code = bs.siteid
WHERE y.sale_date >= TRUNC(SYSDATE, 'MM')  -- current month
GROUP BY
    y.sale_date,
    bs.sitename,
    bs.region
ORDER BY y.sale_date
```

---

## Related Tables

