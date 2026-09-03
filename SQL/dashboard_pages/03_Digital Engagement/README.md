# Digital Engagement Analysis

This analysis evaluates how historical digital engagement relates to future customer retention and revenue. Digital activity occurring before the **December 1, 2025** cutoff date is compared with purchasing outcomes occurring after the cutoff.

The analysis considers four digital behaviors recorded in the clickstream data: **Page View, Search, Add to Cart, and Login**. These behaviors are evaluated both collectively through overall engagement levels and individually by comparing customers who performed each behavior with those who did not.

---

## 1. Returning Customer Rate by Digital Engagement Level (Line Chart)

This query measures each customer's overall historical digital engagement by counting their clickstream events before the cutoff date. Customers are then grouped into four engagement levels based on their total number of digital interactions: **1–4 Events, 5–6 Events, 7–8 Events, and 9+ Events**.

Each engagement group is compared with future purchasing activity to calculate the number of customers who returned after the cutoff and the resulting Returning Customer Rate.

```sql
WITH historical_customers AS (
    SELECT DISTINCT
        customer_id
    FROM clean.orders
    WHERE order_date < '2025-12-01'
      AND customer_id IS NOT NULL
),

historical_engagement AS (
    SELECT
        c.customer_id,
        COUNT(*) AS historical_events
    FROM clean.clickstream c
    INNER JOIN historical_customers h
        ON c.customer_id = h.customer_id
    WHERE c.event_date < '2025-12-01'
    GROUP BY c.customer_id
),

future_activity AS (
    SELECT
        customer_id,
        COUNT(*) AS future_orders
    FROM clean.orders
    WHERE order_date >= '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

customer_metrics AS (
    SELECT
        e.customer_id,
        e.historical_events,
        CASE
            WHEN e.historical_events BETWEEN 1 AND 4
                THEN '1-4 Events'
            WHEN e.historical_events BETWEEN 5 AND 6
                THEN '5-6 Events'
            WHEN e.historical_events BETWEEN 7 AND 8
                THEN '7-8 Events'
            ELSE '9+ Events'
        END AS engagement_group,
        CASE
            WHEN COALESCE(f.future_orders, 0) > 0 THEN 1
            ELSE 0
        END AS future_repeat_customer
    FROM historical_engagement e
    LEFT JOIN future_activity f
        ON e.customer_id = f.customer_id
)

SELECT
    engagement_group,
    COUNT(*) AS historical_customers,
    SUM(future_repeat_customer) AS future_repeat_customers,
    ROUND(
        100.0 * SUM(future_repeat_customer) / COUNT(*),
        2
    ) AS returning_customer_rate
FROM customer_metrics
GROUP BY
    engagement_group,
    CASE engagement_group
        WHEN '1-4 Events' THEN 1
        WHEN '5-6 Events' THEN 2
        WHEN '7-8 Events' THEN 3
        WHEN '9+ Events' THEN 4
    END
ORDER BY
    CASE engagement_group
        WHEN '1-4 Events' THEN 1
        WHEN '5-6 Events' THEN 2
        WHEN '7-8 Events' THEN 3
        WHEN '9+ Events' THEN 4
    END;
```

### Query Output

![Headline Retention KPIs Output](01_headline_kpis.png)

### Dashboard Result
INSERT SCREENSHOT of DIGITAL ENGAGE LINE CHART 

---

## 2. Difference in Returning Customer Rate by Digital Behavior (Bar Chart)

This query evaluates the four digital behaviors individually. For each customer, it identifies whether **Page View, Search, Add to Cart, and Login** were performed at least once during the historical period.

For each behavior, customers are separated into **Performed** and **Not Performed** groups. The query calculates the Returning Customer Rate for both groups and the difference between them in percentage points, allowing the retention association of each digital behavior to be compared.

```sql
WITH historical_customers AS (
    SELECT DISTINCT customer_id
    FROM clean.orders
    WHERE order_date < '2025-12-01'
      AND customer_id IS NOT NULL
),

historical_behaviors AS (
    SELECT h.customer_id,
        MAX(CASE WHEN c.event_type = 'page_view'
                 THEN 1 ELSE 0 END)
                    AS had_page_view,
        MAX(CASE WHEN c.event_type = 'search'
                 THEN 1 ELSE 0 END)
                    AS had_search,
        MAX(CASE WHEN c.event_type = 'add_to_cart'
                 THEN 1 ELSE 0 END)
                    AS had_add_to_cart,
        MAX(CASE WHEN c.event_type = 'login'
                 THEN 1 ELSE 0 END)
                    AS had_login
    FROM historical_customers h
    INNER JOIN clean.clickstream c
        ON h.customer_id = c.customer_id
       AND c.event_date < '2025-12-01'
    GROUP BY h.customer_id
),

future_customers AS (
    SELECT DISTINCT customer_id
    FROM clean.orders
    WHERE order_date >= '2025-12-01'
      AND customer_id IS NOT NULL
),

customer_metrics AS (
    SELECT b.customer_id,
        b.had_page_view,
        b.had_search,
        b.had_add_to_cart,
        b.had_login,
        CASE
            WHEN f.customer_id IS NOT NULL THEN 1
            ELSE 0
        END AS future_repeat_customer
    FROM historical_behaviors b
    LEFT JOIN future_customers f
        ON b.customer_id = f.customer_id
),

behavior_results AS (
    SELECT 'Page View' AS behavior,
        1 AS behavior_order,
        had_page_view AS performed_behavior,
        COUNT(*) AS customers,
        SUM(future_repeat_customer) AS repeat_customers
    FROM customer_metrics
    GROUP BY had_page_view

    UNION ALL

    SELECT 'Search',
        2,
        had_search,
        COUNT(*),
        SUM(future_repeat_customer)
    FROM customer_metrics
    GROUP BY had_search

    UNION ALL

    SELECT 'Add to Cart',
        3,
        had_add_to_cart,
        COUNT(*),
        SUM(future_repeat_customer)
    FROM customer_metrics
    GROUP BY had_add_to_cart

    UNION ALL

    SELECT 'Login',
        4,
        had_login,
        COUNT(*),
        SUM(future_repeat_customer)
    FROM customer_metrics
    GROUP BY had_login
),

behavior_rates AS (
    SELECT behavior,
        behavior_order,
        MAX(customers) FILTER (WHERE performed_behavior = 1)
            AS customers_performed,
        MAX(customers) FILTER (WHERE performed_behavior = 0)
            AS customers_not_performed,
        MAX(100.0 * repeat_customers / customers)
        FILTER (WHERE performed_behavior = 1)
            AS repeat_rate_performed,
        MAX(100.0 * repeat_customers / customers)
        FILTER (WHERE performed_behavior = 0)
            AS repeat_rate_not_performed
    FROM behavior_results
    GROUP BY behavior,
        behavior_order
)

SELECT behavior,
    customers_performed,
    customers_not_performed,
    ROUND(repeat_rate_performed, 2)
        AS returning_customer_rate_performed,
    ROUND(repeat_rate_not_performed, 2)
        AS returning_customer_rate_not_performed,
    ROUND(repeat_rate_performed - repeat_rate_not_performed, 2)
        AS returning_customer_rate_difference_percentage_points
FROM behavior_rates
ORDER BY behavior_order;
```

### Query Output

![Customer Retention Segmentation Output](02_segmentation_bubble_chart.png)

### Dashboard Result
INSERT SCREENSHOT of digital engagement page bar chart 

---

## 3. Average Future Revenue by Digital Behavior (Dumbbell Chart)

This query evaluates the same four historical digital behaviors from a future customer value perspective. Customers are again separated according to whether they **Performed** or **Did Not Perform** each behavior before the cutoff date.

For each behavior, the query calculates average future revenue for both customer groups, along with the absolute and percentage difference in future revenue. This allows the relationship between individual digital behaviors and future revenue to be compared separately from their relationship with customer retention.

```sql
WITH historical_customers AS (
    SELECT DISTINCT customer_id
    FROM clean.orders
    WHERE order_date < '2025-12-01'
      AND customer_id IS NOT NULL
),

historical_behaviors AS (
    SELECT h.customer_id,
        MAX(CASE
                WHEN c.event_type = 'page_view' THEN 1
                ELSE 0
            END)
                AS had_page_view,
        MAX(CASE
                WHEN c.event_type = 'search' THEN 1
                ELSE 0
            END)
                AS had_search,
        MAX(CASE
                WHEN c.event_type = 'add_to_cart' THEN 1
                ELSE 0
            END)
                AS had_add_to_cart,
        MAX(CASE
                WHEN c.event_type = 'login' THEN 1
                ELSE 0
            END)
                AS had_login
    FROM historical_customers h
    INNER JOIN clean.clickstream c
        ON h.customer_id = c.customer_id
       AND c.event_date < '2025-12-01'
    GROUP BY h.customer_id
),

future_activity AS (
    SELECT customer_id,
        SUM(order_amount) AS future_revenue
    FROM clean.orders
    WHERE order_date >= '2025-12-01'
      AND customer_id IS NOT NULL
    GROUP BY customer_id
),

customer_metrics AS (
    SELECT b.customer_id,
        b.had_page_view,
        b.had_search,
        b.had_add_to_cart,
        b.had_login,
        COALESCE(f.future_revenue, 0)
            AS future_revenue
    FROM historical_behaviors b
    LEFT JOIN future_activity f
        ON b.customer_id = f.customer_id
),

behavior_results AS (
    SELECT 'Page View' AS behavior,
        1 AS behavior_order,
        had_page_view AS performed_behavior,
        COUNT(*) AS customers,
        AVG(future_revenue) AS avg_future_revenue
    FROM customer_metrics
    GROUP BY had_page_view

    UNION ALL

    SELECT 'Search',
        2,
        had_search,
        COUNT(*),
        AVG(future_revenue)
    FROM customer_metrics
    GROUP BY had_search

    UNION ALL

    SELECT 'Add to Cart',
        3,
        had_add_to_cart,
        COUNT(*),
        AVG(future_revenue)
    FROM customer_metrics
    GROUP BY had_add_to_cart

    UNION ALL

    SELECT 'Login',
        4,
        had_login,
        COUNT(*),
        AVG(future_revenue)
    FROM customer_metrics
    GROUP BY had_login
),

behavior_values AS (
    SELECT behavior,
        behavior_order,
        MAX(customers) FILTER (WHERE performed_behavior = 1)
            AS customers_performed,
        MAX(customers) FILTER (WHERE performed_behavior = 0)
            AS customers_not_performed,
        MAX(avg_future_revenue) FILTER (WHERE performed_behavior = 1)
            AS avg_revenue_performed,
        MAX(avg_future_revenue) FILTER (WHERE performed_behavior = 0)
            AS avg_revenue_not_performed
    FROM behavior_results
    GROUP BY behavior,
        behavior_order
)

SELECT behavior,
    customers_performed,
    customers_not_performed,
    ROUND(avg_revenue_performed, 2)
        AS avg_future_revenue_performed,
    ROUND(avg_revenue_not_performed, 2)
        AS avg_future_revenue_not_performed,
    ROUND(avg_revenue_performed - avg_revenue_not_performed, 2)
        AS future_revenue_difference,
    ROUND(
        100.0 *
        (avg_revenue_performed - avg_revenue_not_performed)
        / NULLIF(avg_revenue_not_performed, 0),
        2
    ) AS future_revenue_difference_pct
FROM behavior_values
ORDER BY behavior_order;
```

### Query Output

![Customer Segment Summary Output](03_customer_segment_summary.png)

### Dashboard Result
INSERT SCREENSHOT of DIGITAL ENGAGE PAGE DUMBBELL CHART 
