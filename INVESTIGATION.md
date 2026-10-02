# May-13 Revenue Cliff Investigation

## Problem

The case says revenue dropped sharply on May 13. I wanted to first check if the drop was actually visible in the data and then find out what changed on that day.

---

## 1. Baseline Analysis

I started by checking daily orders, revenue and AOV from May 1 to May 20.

```sql
SELECT
    DATE(created_at) AS order_date,
    COUNT(DISTINCT order_id) AS orders,
    SUM(total) AS revenue,
    SUM(total) / COUNT(DISTINCT order_id) AS aov
FROM ecom.orders
WHERE created_at >= '2026-05-01'
  AND created_at < '2026-05-21'
GROUP BY DATE(created_at)
ORDER BY order_date;
