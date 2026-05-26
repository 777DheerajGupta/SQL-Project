SELECT * FROM geolocation;
select * from orders;
select * from payments;
select * from reviews;

-- Q1: Monthly Revenue Trend
SELECT
    DATE_FORMAT(o.order_purchase_timestamp, '%Y-%m-01')  AS order_month,
    COUNT(DISTINCT o.order_id)                            AS total_orders,
    ROUND(SUM(p.payment_value), 2)                        AS total_revenue,
    ROUND(AVG(p.payment_value), 2)                        AS avg_order_value
FROM orders o
JOIN payments p ON o.order_id = p.order_id
WHERE o.order_status = 'delivered'
GROUP BY order_month
ORDER BY order_month;

-- Q2: Order Count by Status
SELECT
    order_status,
    COUNT(*)                                                          AS order_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2)               AS percentage
FROM orders
GROUP BY order_status
ORDER BY order_count DESC;

-- Q3: Top 10 Product Categories by Revenue

SELECT     pr.product_category_name                   AS category,
    COUNT(DISTINCT oi.order_id)                       AS total_orders,
    ROUND(SUM(p.payment_value), 2)                    AS total_revenue,
    ROUND(AVG(p.payment_value), 2)                    AS avg_order_value
FROM items oi
JOIN products pr       ON oi.product_id = pr.product_id
JOIN payments p  ON oi.order_id   = p.order_id
group by pr.product_category_name
ORDER BY total_revenue DESC
LIMIT 10;


-- Q4: Customers by State

SELECT
    c.customer_state,
    COUNT(DISTINCT c.customer_id)                                      AS total_customers,
    COUNT(DISTINCT o.order_id)                                         AS total_orders,
    ROUND(
        COUNT(DISTINCT o.order_id) * 100.0
        / SUM(COUNT(DISTINCT o.order_id)) OVER ()
    , 2)                                                               AS order_pct
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_state
HAVING COUNT(DISTINCT o.order_id) > 100
ORDER BY total_orders DESC;


-- Q5: Average Delivery Time by State

SELECT
    c.customer_state,
    COUNT(o.order_id)                                                AS delivered_orders,
    ROUND(AVG(
        TIMESTAMPDIFF(DAY, o.order_purchase_timestamp,
                           o.order_delivered_customer_date)
    ), 1)                                                            AS avg_delivery_days,
    ROUND(AVG(
        TIMESTAMPDIFF(DAY, o.order_delivered_customer_date,
                           o.order_estimated_delivery_date)
    ), 1)                                                            AS avg_days_early_vs_estimated
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_status = 'delivered'
  AND o.order_delivered_customer_date IS NOT NULL
GROUP BY c.customer_state
ORDER BY avg_delivery_days DESC;


-- Q6: Payment Method Breakdown
SELECT
    payment_type,
    COUNT(DISTINCT order_id)                                          AS total_orders,
    ROUND(
        COUNT(DISTINCT order_id) * 100.0
        / SUM(COUNT(DISTINCT order_id)) OVER ()
    , 2)                                                              AS pct_of_orders,
    ROUND(AVG(payment_installments), 1)                               AS avg_installments,
    ROUND(AVG(payment_value), 2)                                      AS avg_payment_value,
    ROUND(SUM(payment_value), 2)                                      AS total_revenue
FROM payments
GROUP BY payment_type
ORDER BY total_orders DESC;


-- Q7: Seller Performance Ranking

SELECT
    s.seller_id,
    s.seller_city,
    s.seller_state,
    COUNT(DISTINCT oi.order_id)                               AS total_orders,
    ROUND(SUM(oi.price), 2)                                   AS total_revenue,
    RANK() OVER (ORDER BY SUM(oi.price) DESC)                 AS revenue_rank,
    COUNT(CASE WHEN o.order_delivered_customer_date
                    > o.order_estimated_delivery_date
               THEN 1 END)                                    AS late_deliveries,
    ROUND(AVG(oi.price), 2)                                   AS avg_item_price
FROM items oi
JOIN sellers s   ON oi.seller_id = s.seller_id
JOIN orders o    ON oi.order_id  = o.order_id
GROUP BY s.seller_id, s.seller_city, s.seller_state
ORDER BY revenue_rank
LIMIT 20;

-- Q8: Review Score vs Delivery Delay

SELECT
    CASE
        WHEN TIMESTAMPDIFF(DAY,
                 o.order_estimated_delivery_date,
                 o.order_delivered_customer_date) <= -5
                                              THEN '5+ days early'
        WHEN TIMESTAMPDIFF(DAY,
                 o.order_estimated_delivery_date,
                 o.order_delivered_customer_date) BETWEEN -5 AND 0
                                              THEN 'On time / slightly early'
        WHEN TIMESTAMPDIFF(DAY,
                 o.order_estimated_delivery_date,
                 o.order_delivered_customer_date) BETWEEN 1 AND 7
                                              THEN '1-7 days late'
        ELSE '7+ days late'
    END                                                       AS delivery_bucket,
    COUNT(r.review_id)                                        AS review_count,
    ROUND(AVG(r.review_score), 2)                             AS avg_review_score
FROM orders o
JOIN reviews r ON o.order_id = r.order_id
WHERE o.order_status = 'delivered'
  AND o.order_delivered_customer_date IS NOT NULL
GROUP BY delivery_bucket
ORDER BY avg_review_score DESC;


-- Q9: Seller Location vs Delivery Speed (4-table JOIN)

SELECT
    s.seller_state,
    COUNT(DISTINCT o.order_id)                                AS total_orders,
    ROUND(AVG(
        TIMESTAMPDIFF(DAY, o.order_purchase_timestamp,
                           o.order_delivered_customer_date)
    ), 1)                                                     AS avg_delivery_days,
    ROUND(AVG(oi.price), 2)                                   AS avg_item_price
FROM sellers s
JOIN geolocation g  ON s.seller_zip_code_prefix = g.geolocation_zip_code_prefix
JOIN items oi ON s.seller_id = oi.seller_id
JOIN orders o       ON oi.order_id              = o.order_id
WHERE o.order_status = 'delivered'
  AND o.order_delivered_customer_date IS NOT NULL
GROUP BY s.seller_state
ORDER BY avg_delivery_days;


-- Q10: Running Cumulative Revenue + MoM Growth

WITH monthly_revenue AS (
    SELECT
        DATE_FORMAT(o.order_purchase_timestamp, '%Y-%m-01')   AS order_month,
        ROUND(SUM(p.payment_value), 2)                         AS revenue
    FROM orders o
    JOIN payments p ON o.order_id = p.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY order_month
)
SELECT
    order_month,
    revenue,
    ROUND(SUM(revenue) OVER (ORDER BY order_month), 2)         AS cumulative_revenue,
    LAG(revenue) OVER (ORDER BY order_month)                   AS prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY order_month))
        * 100.0
        / NULLIF(LAG(revenue) OVER (ORDER BY order_month), 0)
    , 2)                                                       AS mom_growth_pct
FROM monthly_revenue
ORDER BY order_month;



-- Q13: RFM Customer Segmentation
--      Tables  : olist_orders_dataset
--                olist_order_customer_dataset
--                olist_order_payments_dataset
--      Join    : customer_id, order_id
--      Concepts: NTILE(5) OVER(), 3-CTE chain, CASE WHEN labels
--      Change  : removed ::NUMERIC cast
-- ------------------------------------------------------------
WITH rfm_raw AS (
    SELECT
        c.customer_unique_id,
        MAX(o.order_purchase_timestamp)          AS last_purchase_date,
        COUNT(DISTINCT o.order_id)               AS frequency,
        ROUND(SUM(p.payment_value), 2)           AS monetary
    FROM orders o
    JOIN customers c  ON o.customer_id = c.customer_id
    JOIN payments p  ON o.order_id    = p.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY c.customer_unique_id
),
rfm_scores AS (
    SELECT
        customer_unique_id,
        last_purchase_date,
        frequency,
        monetary,
        NTILE(5) OVER (ORDER BY last_purchase_date DESC)  AS r_score,
        NTILE(5) OVER (ORDER BY frequency ASC)            AS f_score,
        NTILE(5) OVER (ORDER BY monetary ASC)             AS m_score
    FROM rfm_raw
),
rfm_labels AS (
    SELECT
        customer_unique_id,
        last_purchase_date,
        frequency,
        monetary,
        r_score,
        f_score,
        m_score,
        (r_score + f_score + m_score)                     AS rfm_total,
        CASE
            WHEN r_score >= 4 AND f_score >= 4            THEN 'Champion'
            WHEN r_score >= 3 AND f_score >= 3            THEN 'Loyal Customer'
            WHEN r_score >= 4 AND f_score < 2             THEN 'New Customer'
            WHEN r_score >= 3 AND f_score < 3             THEN 'Potential Loyalist'
            WHEN r_score = 2  AND f_score >= 2            THEN 'At Risk'
            WHEN r_score <= 2 AND f_score <= 2            THEN 'Lost'
            ELSE 'Needs Attention'
        END                                               AS customer_segment
    FROM rfm_scores
)
SELECT
    customer_segment,
    COUNT(customer_unique_id)          AS customer_count,
    ROUND(AVG(frequency), 1)           AS avg_orders,
    ROUND(AVG(monetary), 2)            AS avg_spend,
    ROUND(SUM(monetary), 2)            AS total_revenue
FROM rfm_labels
GROUP BY customer_segment
ORDER BY total_revenue DESC;




