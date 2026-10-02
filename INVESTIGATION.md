# Ran the baseline query.

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


| order_date   | orders | revenue      | aov      |
|--------------|-------:|-------------:|---------:|
| May 1, 2026  | 286    | 22,88,592.82 | 8,002.07 |
| May 2, 2026  | 433    | 33,22,423.58 | 7,673.03 |
| May 3, 2026  | 424    | 33,17,579.74 | 7,824.48 |
| May 4, 2026  | 385    | 28,91,274.50 | 7,509.80 |
| May 5, 2026  | 320    | 23,06,352.84 | 7,207.35 |
| May 6, 2026  | 326    | 27,37,478.66 | 8,397.17 |
| May 7, 2026  | 341    | 27,87,220.84 | 8,173.67 |
| May 8, 2026  | 316    | 21,98,813.56 | 6,958.27 |
| May 9, 2026  | 351    | 27,11,131.66 | 7,724.02 |
| May 10, 2026 | 361    | 25,89,363.88 | 7,172.75 |
| May 11, 2026 | 297    | 21,68,488.58 | 7,301.31 |
| May 12, 2026 | 302    | 20,27,728.88 | 6,714.33 |
| **May 13, 2026** | **298** | **18,84,969.96** | **6,325.40** |
| May 14, 2026 | 289    | 20,17,223.04 | 6,980.01 |
| May 15, 2026 | 288    | 24,06,585.72 | 8,356.20 |
| May 16, 2026 | 335    | 26,44,893.96 | 7,895.21 |
| May 17, 2026 | 303    | 23,78,738.34 | 7,850.62 |
| May 18, 2026 | 387    | 28,66,389.42 | 7,406.69 |
| May 19, 2026 | 324    | 23,92,945.10 | 7,385.63 |
| May 20, 2026 | 314    | 25,00,107.96 | 7,962.13 |

### Initial Observation

May 13 had the lowest revenue in the May 1–20 period at ₹18,84,969.96.

Compared with the May 1–12 average, revenue on May 13 appears lower, but the decline is not close to the ~60% cliff described in the case prompt.

Order volume on May 13 was 298, which is in line with the surrounding days. However, AOV fell to ₹6,325.40, the lowest AOV in the period.



# 








