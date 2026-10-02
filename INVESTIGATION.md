# May-13 Revenue Cliff Investigation

## Problem

The case says revenue dropped sharply on May 13. I wanted to first check if the drop was actually visible in the data and then find out what changed on that day.

---

## 1. Baseline Analysis

I started by checking daily orders, revenue and AOV from May 1 to May 20.

### Query

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
```
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


### What I found

May 13 had the lowest revenue in this period.

- Orders: 298
- Revenue: ₹18,84,969.96
- AOV: ₹6,325.40

The number of orders was not much different from the days around it. The bigger change was in AOV, which was the lowest during the period.

The data also did not show a 60% revenue drop. So I did not assume the case description was correct and moved on to payment failures.

---

## 2. Payment Status

I checked payment status around May 13 to see whether failed payments increased on the same day.

### Query

```sql
SELECT
    DATE(created_at) AS order_date,
    payment_status,
    COUNT(DISTINCT order_id) AS orders,
    SUM(total) AS order_value
FROM ecom.orders
WHERE created_at >= '2026-05-10'
  AND created_at < '2026-05-20'
GROUP BY
    DATE(created_at),
    payment_status
ORDER BY
    order_date,
    payment_status;
```
<img width="2164" height="644" alt="Metabase-New question-10_2_2026, 4_41_49 PM" src="https://github.com/user-attachments/assets/2a3bd94f-9954-4000-b30e-1bca36ebb7c1" />

### What I found

There was a clear increase in failed payments on May 13 compared with the surrounding days.

This made payment failure one of the main areas to investigate further.

---

## 3. Payment Gateway Investigation

Next, I checked whether the payment failures were coming from one particular gateway.

### Query

```sql
SELECT
    DATE(o.created_at) AS order_date,
    o.payment_status,
    o.status,
    pm.method_name,
    pt.gateway,
    COUNT(DISTINCT o.order_id) AS failed_orders,
    SUM(o.total) AS total_failed_amount
FROM ecom.orders o
JOIN ecom.payment_intents pi
    ON o.order_id = pi.order_id
JOIN ecom.payment_methods pm
    ON pi.payment_method_id = pm.payment_method_id
JOIN ecom.payment_transactions pt
    ON pi.payment_intent_id = pt.payment_intent_id
WHERE
    o.created_at >= '2026-05-10'
    AND o.created_at < '2026-05-20'
    AND o.payment_status = 'failed'
    AND pm.method_name = 'upi'
    AND pt.status = 'failed'
GROUP BY
    DATE(o.created_at),
    o.payment_status,
    o.status,
    pm.method_name,
    pt.gateway
ORDER BY order_date;
```

### What I found

On May 13, failure rates increased across multiple gateways:

| Gateway | May 13 failure rate |
|---|---:|
| Razorpay | 79% |
| PayU | 63.83% |
| Stripe | 64.86% |
| Cash | 87.5% |

Since the spike was present across multiple gateways, it did not look like an issue with just one provider.

---

## 4. Payment Failure Reason Investigation

I then checked the error codes for failed transactions on May 13.

### Query

```sql
SELECT
    gateway,
    error_code,
    error_message,
    COUNT(*) AS failed_transactions
FROM ecom.payment_transactions
WHERE txn_time >= '2026-05-13'
  AND txn_time < '2026-05-14'
  AND status = 'failed'
GROUP BY
    gateway,
    error_code,
    error_message
ORDER BY failed_transactions DESC;
```

### What I found

The main error was:

`GATEWAY_TIMEOUT - Gateway did not respond within 30s`

There were 168 failed transactions with this error:

| Gateway | Timeout failures |
|---|---:|
| Razorpay | 90 |
| PayU | 28 |
| Cash | 27 |
| Stripe | 23 |
| **Total** | **168** |

Other errors such as bank declines, network errors and fraud checks were much smaller in comparison.

The same timeout appearing across different gateways made a shared payment infrastructure or dependency a more likely explanation than a problem with one gateway.

---

## 5. Business Impact

I then checked which orders were linked to the gateway timeout failures.

### Query

```sql
WITH timeout_orders AS (
    SELECT DISTINCT
        pi.order_id
    FROM ecom.payment_transactions pt
    JOIN ecom.payment_intents pi
        ON pt.payment_intent_id = pi.payment_intent_id
    WHERE pt.txn_time >= '2026-05-13'
      AND pt.txn_time < '2026-05-14'
      AND pt.status = 'failed'
      AND pt.error_code = 'GATEWAY_TIMEOUT'
)
SELECT
    o.payment_status,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(o.total) AS order_value
FROM ecom.orders o
JOIN timeout_orders t
    ON o.order_id = t.order_id
GROUP BY o.payment_status
ORDER BY orders DESC;
```

### Result

| payment_status | orders | order_value |
|---|---:|---:|
| failed | 134 | ₹8,66,262.50 |

### What this means

134 orders were linked to at least one gateway timeout on May 13.

The total value of these orders was ₹8,66,262.50, and all of them had a final payment status of `failed`.

I am treating this as affected order value rather than confirmed lost revenue, since the data does not tell me whether a customer later came back and completed the purchase through another order.

---

## Conclusion

The main issue on May 13 appears to have been payment failures.

The problem was not limited to one gateway. Multiple gateways showed a large increase in failures, and `GATEWAY_TIMEOUT` was by far the most common error.

There were 168 timeout failures linked to 134 orders worth ₹8,66,262.50.

The data points to a payment infrastructure or shared dependency issue as the likely reason for the increase in failed payments and the resulting revenue decline.

I could identify the failure pattern and its business impact, but the available data does not show the exact technical root cause behind the timeout.
