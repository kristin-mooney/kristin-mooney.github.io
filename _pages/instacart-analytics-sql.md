---
layout: page
title: "Instacart Analytics — SQL Queries"
subtitle: "Staging, Intermediate, Marts & Analytical Queries"
permalink: /instacart-analytics-sql/
author_profile: true
toc: true
toc_label: "SQL Models"
toc_icon: "code"
---


---

[← Back to Project Overview](/instacart-analytics/){: .btn .btn--info}

---

All SQL models are organized following a staging → analysis → marts architecture. Each layer builds on the previous to produce clean, analysis-ready tables for Tableau.

---

## 1. Staging Models

Staging models clean and standardize raw source tables. Each model maps directly to one source CSV file and handles nulls, adds readable labels, and applies consistent naming conventions.

---
/*create staging tables--------------------------------------------------------*/
```sql
USE instacart;

select * from orders limit 10;

CREATE OR REPLACE VIEW stg_orders AS
SELECT
    order_id,
    user_id,
    eval_set,
    order_number,
    order_dow,
    CASE order_dow
        WHEN 0 THEN 'Sunday'
        WHEN 1 THEN 'Monday'
        WHEN 2 THEN 'Tuesday'
        WHEN 3 THEN 'Wednesday'
        WHEN 4 THEN 'Thursday'
        WHEN 5 THEN 'Friday'
        WHEN 6 THEN 'Saturday'
    END AS order_day_name,
    order_hour_of_day,
    CASE
        WHEN order_hour_of_day BETWEEN 5  AND 8  THEN 'Early Morning (5-8am)'
        WHEN order_hour_of_day BETWEEN 9  AND 11 THEN 'Morning (9-11am)'
        WHEN order_hour_of_day BETWEEN 12 AND 14 THEN 'Lunch (12-2pm)'
        WHEN order_hour_of_day BETWEEN 15 AND 17 THEN 'Afternoon (3-5pm)'
        WHEN order_hour_of_day BETWEEN 18 AND 20 THEN 'Evening (6-8pm)'
        WHEN order_hour_of_day BETWEEN 21 AND 23 THEN 'Night (9-11pm)'
        ELSE                                          'Overnight (0-4am)'
    END AS time_of_day_bucket,
    COALESCE(days_since_prior_order, 0) AS days_since_prior_order,
    CASE
        WHEN days_since_prior_order IS NULL  THEN 'First Order'
        WHEN days_since_prior_order <= 3     THEN '0-3 days'
        WHEN days_since_prior_order <= 7     THEN '4-7 days'
        WHEN days_since_prior_order <= 14    THEN '8-14 days'
        WHEN days_since_prior_order <= 21    THEN '15-21 days'
        ELSE                                      '22-30 days'
    END AS reorder_cycle_bucket,
    CASE WHEN order_number = 1 THEN 1 ELSE 0 END AS is_first_order
FROM orders;

select * from products limit 10;

CREATE OR REPLACE VIEW stg_products AS
SELECT
    product_id,
    product_name,
    aisle_id,
    department_id,
    CASE
        WHEN LOWER(product_name) LIKE '%organic%' THEN 1
        ELSE 0
    END AS is_organic
FROM products;

select * from order_products__prior limit 10;

select * from order_products__train limit 10;

CREATE OR REPLACE VIEW stg_order_products AS
SELECT
    order_id,
    product_id,
    add_to_cart_order,
    reordered,
    'prior' AS source_set
FROM order_products__prior
UNION ALL
SELECT
    order_id,
    product_id,
    add_to_cart_order,
    reordered,
    'train' AS source_set
FROM order_products__train;


SHOW FULL TABLES IN instacart WHERE TABLE_TYPE = 'VIEW';

#DATA INTEGRITY CHECKS
SELECT * FROM stg_orders LIMIT 10;

/*3,421,083*/
SELECT COUNT(*) FROM stg_orders LIMIT 10;


SELECT * FROM stg_products LIMIT 10;

/*49,355*/
select count(*) from stg_products;

-- Should return rows from both 'prior' and 'train' source sets
SELECT * FROM stg_order_products LIMIT 10;
/*33,819,106*/
select count(*) from stg_order_products;
```


---

## 2. Analysis Models

---

/*create analysis tables---------------------------------------------------------*/

```sql
#dim_products - full product catelog with aisle, department, and organic flag
CREATE TABLE dim_products AS
SELECT
    p.product_id,
    p.product_name,
    p.is_organic,
    a.aisle_id,
    a.aisle,
    d.department_id,
    d.department
FROM stg_products p
LEFT JOIN aisles      a 
	ON p.aisle_id = a.aisle_id
LEFT JOIN departments d 
	ON p.department_id = d.department_id;

SELECT * FROM dim_products LIMIT 5;

#create dim_users--one row summarizing full order history
CREATE TABLE dim_users AS
WITH user_orders AS (
    SELECT
        o.user_id,
        COUNT(DISTINCT o.order_id)              AS total_orders,
        AVG(o.days_since_prior_order)           AS avg_days_between_orders,
        SUM(op.item_count)                      AS total_items_purchased,
        AVG(op.item_count)                      AS avg_basket_size,
        AVG(op.reorder_rate)                    AS reorder_rate
    FROM stg_orders o
    LEFT JOIN (
        SELECT
            order_id,
            COUNT(*)                            AS item_count,
            AVG(reordered)                      AS reorder_rate
        FROM stg_order_products
        GROUP BY order_id
    ) op ON o.order_id = op.order_id
    GROUP BY o.user_id
),

-- Find each user's most common order day
fav_day AS (
    SELECT user_id, order_day_name AS favorite_order_day
    FROM (
        SELECT
            user_id,
            order_day_name,
            ROW_NUMBER() OVER (
                PARTITION BY user_id
                ORDER BY COUNT(*) DESC
            ) AS rn
        FROM stg_orders
        GROUP BY user_id, order_day_name
    ) ranked
    WHERE rn = 1
),

-- Find each user's most common time of day
fav_time AS (
    SELECT user_id, time_of_day_bucket AS favorite_time_of_day
    FROM (
        SELECT
            user_id,
            time_of_day_bucket,
            ROW_NUMBER() OVER (
                PARTITION BY user_id
                ORDER BY COUNT(*) DESC
            ) AS rn
        FROM stg_orders
        GROUP BY user_id, time_of_day_bucket
    ) ranked
    WHERE rn = 1
)

SELECT
    uo.user_id,
    uo.total_orders,
    uo.total_items_purchased,
    ROUND(uo.avg_basket_size, 2)        AS avg_basket_size,
    ROUND(uo.avg_days_between_orders, 1) AS avg_days_between_orders,
    ROUND(uo.reorder_rate, 3)           AS reorder_rate,
    fd.favorite_order_day,
    ft.favorite_time_of_day
FROM user_orders uo
LEFT JOIN fav_day  fd 
	ON uo.user_id = fd.user_id
LEFT JOIN fav_time ft 
	ON uo.user_id = ft.user_id;

#quick data integrity check
select * from dim_users limit 5;

select count(*) from dim_users;

#fct_orders - one row per order with basket stats joined in
CREATE TABLE fct_orders AS
SELECT
    o.order_id,
    o.user_id,
    o.order_number,
    o.order_day_name,
    o.order_dow,
    o.order_hour_of_day,
    o.time_of_day_bucket,
    o.reorder_cycle_bucket,
    o.days_since_prior_order,
    o.is_first_order,
    COUNT(op.product_id)        AS basket_size,
    ROUND(AVG(op.reordered), 3) AS reorder_rate,
    SUM(op.reordered)           AS reordered_items,
    SUM(CASE WHEN op.reordered = 0 THEN 1 ELSE 0 END) AS new_items
FROM stg_orders o
LEFT JOIN stg_order_products op 
	ON o.order_id = op.order_id
GROUP BY
    o.order_id,
    o.user_id,
    o.order_number,
    o.order_day_name,
    o.order_dow,
    o.order_hour_of_day,
    o.time_of_day_bucket,
    o.reorder_cycle_bucket,
    o.days_since_prior_order,
    o.is_first_order;
    
    #data integrity check
    SELECT * FROM fct_orders LIMIT 5;

SELECT COUNT(*) FROM fct_orders;

SELECT COUNT(*) FROM dim_users;

show tables IN instacart;
```

---

## 3. Mart Tables Prepped for Tableau

---

/*prep tables for tableau csv load--------------------------------------*/

```sql
select * from stg_order_products limit 5;

select * from dim_products limit 5;

#this will help analyze which products drive the most volume and loyalty
CREATE TABLE mart_product_performance AS
SELECT
    dp.product_id,
    dp.product_name,
    dp.aisle,
    dp.department,
    dp.is_organic,
    COUNT(op.order_id)                          AS total_orders,
    SUM(op.reordered)                           AS total_reorders,
    ROUND(AVG(op.reordered), 3)                 AS reorder_rate,
    ROUND(AVG(op.add_to_cart_order), 1)         AS avg_cart_position
FROM stg_order_products op
JOIN dim_products dp 
	ON op.product_id = dp.product_id
GROUP BY
    dp.product_id,
    dp.product_name,
    dp.aisle,
    dp.department,
    dp.is_organic;
  
  #49,677
  select count(*) from mart_product_performance;
  
   select * from mart_product_performance limit 10;


#this table will help analyze user segmentation by shopping behavior

select * from dim_users limit 5;

CREATE TABLE mart_user_rfm AS
WITH rfm_scores AS (
    SELECT
        user_id,
        total_orders,
        avg_days_between_orders,
        total_items_purchased,
        avg_basket_size,
        reorder_rate,
        favorite_order_day,
        favorite_time_of_day,
        NTILE(4) OVER (ORDER BY total_orders DESC)              AS frequency_score,
        NTILE(4) OVER (ORDER BY avg_days_between_orders ASC)    AS recency_score,
        NTILE(4) OVER (ORDER BY total_items_purchased DESC)     AS volume_score
    FROM dim_users
)
SELECT
    user_id,
    total_orders,
    avg_days_between_orders,
    total_items_purchased,
    avg_basket_size,
    reorder_rate,
    favorite_order_day,
    favorite_time_of_day,
    frequency_score,
    recency_score,
    volume_score,
    CASE
        WHEN frequency_score = 1 AND recency_score = 1  THEN 'Champion'
        WHEN frequency_score = 1 AND recency_score = 2  THEN 'Loyal Customer'
        WHEN frequency_score <= 2 AND recency_score <= 2 THEN 'Potential Loyalist'
        WHEN frequency_score >= 3 AND recency_score = 1 THEN 'New Customer'
        WHEN frequency_score >= 3 AND recency_score >= 3 THEN 'At Risk'
        ELSE                                                  'Occasional Shopper'
    END AS rfm_segment
FROM rfm_scores;

select * from mart_user_rfm limit 10;

#206,209
select count(*) from mart_user_rfm;


#this table will help analyze when people shop
select * from fct_orders limit 10;

CREATE TABLE mart_timing_analysis AS
SELECT
    order_day_name,
    order_dow,
    time_of_day_bucket,
    order_hour_of_day,
    COUNT(*)                        AS total_orders,
    ROUND(AVG(basket_size), 2)      AS avg_basket_size,
    ROUND(AVG(reorder_rate), 3)     AS avg_reorder_rate,
    SUM(is_first_order)             AS first_orders
FROM fct_orders
GROUP BY
    order_day_name,
    order_dow,
    time_of_day_bucket,
    order_hour_of_day;

select * from mart_timing_analysis limit 10;

#168
select count(*) from mart_timing_analysis

```


---

[← Back to Project Overview](/instacart-analytics/){: .btn .btn--info .btn--large}
[View Full Tableau Story](https://public.tableau.com/app/profile/kristin.mooney/viz/InstacartAnalyticsProject/UnderstandingInstacartShopperBehavior){: .btn .btn--primary .btn--large}

