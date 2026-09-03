# Purchasing Behavior Analysis

This analysis evaluates how historical purchasing behavior relates to future customer retention. Customer activity before the **December 1, 2025** cutoff date is compared with purchasing outcomes occurring after the cutoff.

The analysis focuses on four historical purchasing characteristics:
- Purchase frequency
- Recency of the last purchase
- Total customer spending
- Average order value

---

## 1. Returning Customer Rate by Order Frequency (Column Chart)

This query groups customers according to the number of purchases they made during the historical period. Customers are divided into five purchase-frequency groups ranging from **1–2 Orders** to **9+ Orders**.

Each group is then compared with future purchasing activity to calculate the number of customers who returned after the cutoff date, and the resulting returning customer rate.

```sql
WITH historical_activity AS (
    SELECT
        customer_id,
        COUNT(*) AS historical_orders
    FROM clean.orders
    WHERE order_date < '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

future_activity AS (
    SELECT
        customer_id,
        COUNT(*) AS future_orders,
        SUM(order_amount) AS future_revenue
    FROM clean.orders
    WHERE order_date >= '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

customer_metrics AS (
    SELECT
        h.customer_id,
        h.historical_orders,
        CASE
            WHEN h.historical_orders BETWEEN 1 AND 2 THEN '1-2 Orders'
            WHEN h.historical_orders BETWEEN 3 AND 4 THEN '3-4 Orders'
            WHEN h.historical_orders BETWEEN 5 AND 6 THEN '5-6 Orders'
            WHEN h.historical_orders BETWEEN 7 AND 8 THEN '7-8 Orders'
            ELSE '9+ Orders'
        END AS purchase_frequency_group,
        CASE
            WHEN COALESCE(f.future_orders, 0) > 0 THEN 1
            ELSE 0
        END AS future_repeat_customer,
        COALESCE(f.future_revenue, 0) AS future_revenue
    FROM historical_activity h
    LEFT JOIN future_activity f
        ON h.customer_id = f.customer_id
)

SELECT
    purchase_frequency_group,
    COUNT(*) AS historical_customers,
    SUM(future_repeat_customer) AS future_repeat_customers,
    ROUND(
        100.0 * SUM(future_repeat_customer) / COUNT(*),
        2
    ) AS returning_customer_rate,
    ROUND(
        AVG(future_revenue),
        2
    ) AS avg_future_revenue_per_customer
FROM customer_metrics
GROUP BY
    purchase_frequency_group,
    CASE purchase_frequency_group
        WHEN '1-2 Orders' THEN 1
        WHEN '3-4 Orders' THEN 2
        WHEN '5-6 Orders' THEN 3
        WHEN '7-8 Orders' THEN 4
        WHEN '9+ Orders' THEN 5
    END
ORDER BY
    CASE purchase_frequency_group
        WHEN '1-2 Orders' THEN 1
        WHEN '3-4 Orders' THEN 2
        WHEN '5-6 Orders' THEN 3
        WHEN '7-8 Orders' THEN 4
        WHEN '9+ Orders' THEN 5
    END;
```

### Query Output

![Headline Retention KPIs Output](01_headline_kpis.png)

### Dashboard Result
INSERT SCREENSHOT of purchase page skinny column chart 

---

## 2. Returning Customer Rate by Recency of Last Order (Column Chart)

This query measures customer recency as the number of days between each customer's most recent historical order and the cutoff date. Customers are then grouped into five recency ranges from **0–30 Days** to **366+ Days**.

Future purchasing activity is joined to these historical customer groups to measure how Returning Customer Rate and average future revenue vary as the time since a customer's last purchase increases.

```sql
WITH historical_activity AS (
    SELECT
        customer_id,
        MAX(order_date) AS last_historical_order_date,
        DATE '2025-12-01' - MAX(order_date) AS days_since_last_order
    FROM clean.orders
    WHERE order_date < '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

future_activity AS (
    SELECT
        customer_id,
        COUNT(*) AS future_orders,
        SUM(order_amount) AS future_revenue
    FROM clean.orders
    WHERE order_date >= '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

customer_metrics AS (
    SELECT
        h.customer_id,
        h.days_since_last_order,
        CASE
            WHEN h.days_since_last_order <= 30
                THEN '0-30 Days'
            WHEN h.days_since_last_order <= 90
                THEN '31-90 Days'
            WHEN h.days_since_last_order <= 180
                THEN '91-180 Days'
            WHEN h.days_since_last_order <= 365
                THEN '181-365 Days'
            ELSE '366+ Days'
        END AS recency_group,
        CASE
            WHEN COALESCE(f.future_orders, 0) > 0 THEN 1
            ELSE 0
        END AS future_repeat_customer,
        COALESCE(f.future_revenue, 0) AS future_revenue
    FROM historical_activity h
    LEFT JOIN future_activity f
        ON h.customer_id = f.customer_id
)

SELECT
    recency_group,
    COUNT(*) AS historical_customers,
    SUM(future_repeat_customer) AS future_repeat_customers,
    ROUND(
        100.0 * SUM(future_repeat_customer) / COUNT(*),
        2
    ) AS returning_customer_rate,
    ROUND(
        AVG(future_revenue),
        2
    ) AS avg_future_revenue_per_customer
FROM customer_metrics
GROUP BY
    recency_group,
    CASE recency_group
        WHEN '0-30 Days' THEN 1
        WHEN '31-90 Days' THEN 2
        WHEN '91-180 Days' THEN 3
        WHEN '181-365 Days' THEN 4
        WHEN '366+ Days' THEN 5
    END
ORDER BY
    CASE recency_group
        WHEN '0-30 Days' THEN 1
        WHEN '31-90 Days' THEN 2
        WHEN '91-180 Days' THEN 3
        WHEN '181-365 Days' THEN 4
        WHEN '366+ Days' THEN 5
    END;
```

### Query Output

![Customer Retention Segmentation Output](02_segmentation_bubble_chart.png)

### Dashboard Result
INSERT SCREENSHOT of purchase page fat column charT

---

## 3. Returning Customer Rate by Total Spending and Average Order Value (Double Bar Chart)

The final Purchasing Behavior visual compares retention across two measures of historical customer value: **Total Spending** and **Average Order Value (AOV)**.

Because the two measures are calculated independently, separate SQL queries were used. In both analyses, customers are divided into four equally sized groups using `NTILE(4)`, producing **Low, Medium, High, and Very High** value tiers. Future purchasing behavior is then evaluated for each tier.

The outputs of both queries are combined in Excel to compare their Returning Customer Rates within a single dashboard visual.

### 3.1 Historical Spending

This query calculates each customer's total order revenue during the historical period and ranks customers into four spending quartiles. It then measures future retention and revenue for each historical spending tier.

```sql
WITH historical_activity AS (
    SELECT
        customer_id,
        SUM(order_amount) AS historical_revenue
    FROM clean.orders
    WHERE order_date < '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

historical_with_quartiles AS (
    SELECT
        customer_id,
        historical_revenue,
        NTILE(4) OVER (
            ORDER BY historical_revenue
        ) AS spending_quartile
    FROM historical_activity
),

future_activity AS (
    SELECT
        customer_id,
        COUNT(*) AS future_orders,
        SUM(order_amount) AS future_revenue
    FROM clean.orders
    WHERE order_date >= '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

customer_metrics AS (
    SELECT
        h.customer_id,
        h.historical_revenue,
        h.spending_quartile,
        CASE h.spending_quartile
            WHEN 1 THEN 'Low Spend'
            WHEN 2 THEN 'Medium Spend'
            WHEN 3 THEN 'High Spend'
            WHEN 4 THEN 'Very High Spend'
        END AS historical_spending_group,
        CASE
            WHEN COALESCE(f.future_orders, 0) > 0 THEN 1
            ELSE 0
        END AS future_repeat_customer,
        COALESCE(f.future_revenue, 0) AS future_revenue
    FROM historical_with_quartiles h
    LEFT JOIN future_activity f
        ON h.customer_id = f.customer_id
)

SELECT
    historical_spending_group,
    COUNT(*) AS historical_customers,
    SUM(future_repeat_customer) AS future_repeat_customers,
    ROUND(
        100.0 * SUM(future_repeat_customer) / COUNT(*),
        2
    ) AS returning_customer_rate,
    ROUND(
        AVG(future_revenue),
        2
    ) AS avg_future_revenue_per_customer
FROM customer_metrics
GROUP BY
    spending_quartile,
    historical_spending_group
ORDER BY
    spending_quartile;
```
### Query Output
INSERT QUERY OUTPUT 
DO NOT INSERT ANY EXCEL CHART, DO AT END

### 3.2 Historical Average Order Value

This query calculates each customer's average historical order value and divides customers into four AOV quartiles. Future purchasing activity is then used to measure the Returning Customer Rate and average future revenue associated with each AOV tier.

```sql
WITH historical_activity AS (
    SELECT
        customer_id,
        AVG(order_amount) AS historical_avg_order_value
    FROM clean.orders
    WHERE order_date < '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

historical_with_quartiles AS (
    SELECT
        customer_id,
        historical_avg_order_value,
        NTILE(4) OVER (
            ORDER BY historical_avg_order_value
        ) AS aov_quartile
    FROM historical_activity
),

future_activity AS (
    SELECT
        customer_id,
        COUNT(*) AS future_orders,
        SUM(order_amount) AS future_revenue
    FROM clean.orders
    WHERE order_date >= '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

customer_metrics AS (
    SELECT
        h.customer_id,
        h.historical_avg_order_value,
        h.aov_quartile,
        CASE h.aov_quartile
            WHEN 1 THEN 'Low AOV'
            WHEN 2 THEN 'Medium AOV'
            WHEN 3 THEN 'High AOV'
            WHEN 4 THEN 'Very High AOV'
        END AS historical_aov_group,
        CASE
            WHEN COALESCE(f.future_orders, 0) > 0 THEN 1
            ELSE 0
        END AS future_repeat_customer,
        COALESCE(f.future_revenue, 0) AS future_revenue
    FROM historical_with_quartiles h
    LEFT JOIN future_activity f
        ON h.customer_id = f.customer_id
)

SELECT
    historical_aov_group,
    COUNT(*) AS historical_customers,
    SUM(future_repeat_customer) AS future_repeat_customers,
    ROUND(
        100.0 * SUM(future_repeat_customer) / COUNT(*),
        2
    ) AS future_repeat_purchase_rate,
    ROUND(
        AVG(future_revenue),
        2
    ) AS avg_future_revenue_per_customer
FROM customer_metrics
GROUP BY
    aov_quartile,
    historical_aov_group
ORDER BY
    aov_quartile;
```

### Query Output

![Customer Segment Summary Output](03_customer_segment_summary.png)

### Dashboard Result
INSERT SCREENSHOT of purchase page Double Bar Chart
