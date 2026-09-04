# Support Experience Analysis

This analysis evaluates how historical customer support experiences relate to future customer retention and revenue. Support activity occurring before the **December 1, 2025** cutoff date is compared with purchasing outcomes occurring after the cutoff.

The analysis focuses on three aspects of the historical support experience:

- Support Request frequency
- Average Support Resolution Time
- Most Recent Support Sentiment
- 
---

## 1. Returning Customer Rate by Support Request Frequency (Column Chart)

This query counts the number of support requests associated with each historical customer before the cutoff date and groups customers into four frequency levels: **0, 1, 2, and 3+ Support Requests**.

Each group is then compared with future purchasing activity to calculate the number of customers who returned after the cutoff, the resulting Returning Customer Rate, and average future revenue per customer.

```sql
WITH historical_customers AS (
    SELECT DISTINCT customer_id
    FROM clean.orders
    WHERE order_date < '2025-12-01'
      AND customer_id IS NOT NULL
),

historical_support AS (
    SELECT h.customer_id,
        COUNT(s.ticket_id) AS historical_support_tickets
    FROM historical_customers h
    LEFT JOIN clean.support_tickets s
        ON h.customer_id = s.customer_id
       AND s.ticket_created < '2025-12-01'
    GROUP BY h.customer_id
),

future_activity AS (
    SELECT customer_id,
        COUNT(*) AS future_orders,
        SUM(order_amount) AS future_revenue
    FROM clean.orders
    WHERE order_date >= '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

customer_metrics AS (
    SELECT s.customer_id,
        CASE
            WHEN s.historical_support_tickets = 0
                THEN '0 Tickets'
            WHEN s.historical_support_tickets = 1
                THEN '1 Ticket'
            WHEN s.historical_support_tickets = 2
                THEN '2 Tickets'
            ELSE '3+ Tickets'
        END AS support_frequency_group,
        CASE
            WHEN COALESCE(f.future_orders, 0) > 0 THEN 1
            ELSE 0
        END AS future_repeat_customer,
        COALESCE(f.future_revenue, 0) AS future_revenue
    FROM historical_support s
    LEFT JOIN future_activity f
        ON s.customer_id = f.customer_id
)

SELECT support_frequency_group,
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
GROUP BY support_frequency_group,
    CASE support_frequency_group
        WHEN '0 Tickets' THEN 1
        WHEN '1 Ticket' THEN 2
        WHEN '2 Tickets' THEN 3
        WHEN '3+ Tickets' THEN 4
    END
ORDER BY
    CASE support_frequency_group
        WHEN '0 Tickets' THEN 1
        WHEN '1 Ticket' THEN 2
        WHEN '2 Tickets' THEN 3
        WHEN '3+ Tickets' THEN 4
    END;
```

### Query Output

![Headline Retention KPIs Output](01_headline_kpis.png)

### Dashboard Result
INSERT SCREENSHOT of support exp column CHART 

---

## 2. Average Future Revenue by Average Support Resolution Time (Line Chart)

This query calculates each customer's **average resolution time across their historical support requests** before the cutoff date. Customers are then grouped into four resolution-time ranges: **0–72 Hours, 73–120 Hours, 121–168 Hours, and 169+ Hours**.

Future purchasing activity is joined to these customer groups to calculate both Returning Customer Rate and average future revenue. The average future revenue measure is used in the dashboard to compare future customer value across historical support resolution-time groups.

```sql
WITH historical_customers AS (
    SELECT DISTINCT customer_id
    FROM clean.orders
    WHERE order_date < '2025-12-01'
      AND customer_id IS NOT NULL
),

historical_resolution AS (
    SELECT s.customer_id,
        AVG(s.resolution_time_hours) AS avg_resolution_hours
    FROM clean.support_tickets s
    INNER JOIN historical_customers h
        ON s.customer_id = h.customer_id
    WHERE s.ticket_created < '2025-12-01'
      AND s.customer_id IS NOT NULL
      AND s.resolution_time_hours IS NOT NULL
    GROUP BY s.customer_id
),

future_activity AS (
    SELECT customer_id,
        COUNT(*) AS future_orders,
        SUM(order_amount) AS future_revenue
    FROM clean.orders
    WHERE order_date >= '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

customer_metrics AS (
    SELECT r.customer_id,
        CASE
            WHEN r.avg_resolution_hours <= 72
                THEN '0-72 Hours'
            WHEN r.avg_resolution_hours <= 120
                THEN '73-120 Hours'
            WHEN r.avg_resolution_hours <= 168
                THEN '121-168 Hours'
            ELSE '169+ Hours'
        END AS resolution_time_group,
        CASE
            WHEN COALESCE(f.future_orders, 0) > 0 THEN 1
            ELSE 0
        END AS future_repeat_customer,
        COALESCE(f.future_revenue, 0) AS future_revenue
    FROM historical_resolution r
    LEFT JOIN future_activity f
        ON r.customer_id = f.customer_id
)

SELECT resolution_time_group,
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
GROUP BY resolution_time_group,
    CASE resolution_time_group
        WHEN '0-72 Hours' THEN 1
        WHEN '73-120 Hours' THEN 2
        WHEN '121-168 Hours' THEN 3
        WHEN '169+ Hours' THEN 4
    END
ORDER BY
    CASE resolution_time_group
        WHEN '0-72 Hours' THEN 1
        WHEN '73-120 Hours' THEN 2
        WHEN '121-168 Hours' THEN 3
        WHEN '169+ Hours' THEN 4
    END;
```

### Query Output

![Customer Retention Segmentation Output](02_segmentation_bubble_chart.png)

### Dashboard Result
INSERT SCREENSHOT of SUPPORT EXP line chart 

---

## 3. Average Future Revenue by Support Sentiment (Bar Chart)

This query evaluates future customer outcomes according to each customer's **most recent historical support sentiment**. Where a customer has multiple support requests before the cutoff, `ROW_NUMBER()` is used to identify the most recent request based on its creation date.

Customers are then grouped by **Positive, Neutral, or Negative** support sentiment. For each group, the query calculates Returning Customer Rate and average future revenue per customer, allowing retention and future customer value to be evaluated together.

```sql
WITH historical_customers AS (
    SELECT DISTINCT customer_id
    FROM clean.orders
    WHERE order_date < '2025-12-01'
      AND customer_id IS NOT NULL
),

ranked_support AS (
    SELECT s.customer_id,
        s.sentiment,
        ROW_NUMBER() OVER (
            PARTITION BY s.customer_id
            ORDER BY s.ticket_created DESC
        ) AS rn
    FROM clean.support_tickets s
    INNER JOIN historical_customers h
        ON s.customer_id = h.customer_id
    WHERE s.ticket_created < '2025-12-01'
      AND s.customer_id IS NOT NULL
),

latest_support AS (
    SELECT customer_id,
        sentiment
    FROM ranked_support
    WHERE rn = 1
),

future_activity AS (
    SELECT customer_id,
        COUNT(*) AS future_orders,
        SUM(order_amount) AS future_revenue
    FROM clean.orders
    WHERE order_date >= '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

customer_metrics AS (
    SELECT l.customer_id,
        l.sentiment,
        CASE
            WHEN COALESCE(f.future_orders, 0) > 0 THEN 1
            ELSE 0
        END AS future_repeat_customer,
        COALESCE(f.future_revenue, 0)
            AS future_revenue
    FROM latest_support l
    LEFT JOIN future_activity f
        ON l.customer_id = f.customer_id
)

SELECT sentiment AS most_recent_support_sentiment,
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
GROUP BY sentiment
ORDER BY
    CASE sentiment
        WHEN 'positive' THEN 1
        WHEN 'neutral' THEN 2
        WHEN 'negative' THEN 3
    END;
```

### Query Output

![Customer Segment Summary Output](03_customer_segment_summary.png)

### Dashboard Result
INSERT SCREENSHOT of SUPPORT EXP bar chart
