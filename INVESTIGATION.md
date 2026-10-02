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
