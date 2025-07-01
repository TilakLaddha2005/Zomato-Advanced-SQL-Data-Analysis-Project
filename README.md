# Zomato-Advanced-SQL-Data-Analysis-Project

A complete SQL-based analytics project on a Zomato-style food delivery platform with database schema, mock data, and 20+ real-world SQL queries for business insights and customer behavior analysis.

---

##  Project Structure

This project uses mock data across five key entities:

| Table       | Description                                 |
|-------------|---------------------------------------------|
| `customers`   | Customer info (ID, name, contact, location) |
| `restaurants` | Restaurant data (cuisine, rating, etc.)     |
| `riders`      | Rider details (vehicle type, contact)       |
| `delivery`    | Delivery assignments, linked to riders      |
| `orders`      | Order facts with customer, payment, status  |

---

![ERD diagram](https://github.com/TilakLaddha2005/Zomato-Advanced-SQL-Data-Analysis-Project/blob/main/ERD%20diagram.png)

---
## Data Handling & NULL Cleanup

```sql
-- Insert dummy deliverers with missing rider_id

INSERT INTO delivery (deliverer_id) VALUES ('DEL4150'), ('DEL4151'), ('DEL4152');

-- Count nulls
SELECT COUNT(*) FROM delivery WHERE rider_id IS NULL;

-- Delete rows with missing FK
DELETE FROM delivery WHERE rider_id IS NULL;

```
---

## Business Queries & Analysis

###  Customer Behavior

```sql
-- Ordered cuisines by customer (last 1 year)
select customer_id 
FROM customers
WHERE name = 'David Ramirez';

SELECT COUNT(order_id) 
FROM orders
WHERE customer_id = 'CUST1000';  -- 2 orders by David Ramirez
SELECT res.cuisine
FROM orders o
JOIN restaurants res ON o.restaurant_id = res.restaurant_id
WHERE o.customer_id = 'CUST1000'
AND o.order_date >= CURRENT_DATE - INTERVAL '1 YEAR';

-- Top 3 customers by lifetime orders
WITH ranked_customers AS (
  SELECT cus.customer_id, cus.name, COUNT(o.order_id) AS total_orders,
  DENSE_RANK() OVER(ORDER BY COUNT(o.order_id) DESC) AS top_rank
  FROM orders o
  JOIN customers cus ON o.customer_id = cus.customer_id
  GROUP BY cus.customer_id, cus.name
)
SELECT * FROM ranked_customers WHERE top_rank <= 3;

-- Top 5 customers by spend
WITH rank_customers_spend AS (
  SELECT cus.customer_id, cus.name, SUM(o.order_amount) as total_spend,
  DENSE_RANK() OVER(ORDER BY SUM(o.order_amount) DESC) AS ranked
  FROM customers cus
  JOIN orders o ON cus.customer_id = o.customer_id
  GROUP BY cus.customer_id, cus.name
)
SELECT * FROM rank_customers_spend WHERE ranked <= 5;

-- Cancellation rate per customer
SELECT
cus.customer_id,
cus.name,
COUNT(orders.order_id) as total_orders,
SUM(CASE WHEN orders.order_status = 'Cancelled' then 1 else 0 END)as cancelled_orders,
SUM(CASE WHEN orders.order_status = 'Cancelled' then 1 else 0 END)* 100 / COUNT(orders.order_id) AS cancellation_rate
FROM customers cus
JOIN orders
ON orders.customer_id = cus.customer_id
GROUP BY
cus.customer_id,
cus.name
ORDER BY cancellation_rate DESC;

-- Find customers who haven’t ordered in the last 90 days or not at all
WITH last_orders AS(
	SELECT 
	o.customer_id,
	MAX(o.order_date) as last_order
	FROM orders o
	GROUP BY
	o.customer_id)

SELECT 
cus.customer_id,
cus.name,
lo.last_order
FROM customers cus
JOIN last_orders lo
ON lo.customer_id = cus.customer_id
WHERE lo.last_order IS NULL OR
lo.last_order < CURRENT_DATE - INTERVAL '90 Day' ;

-- Identify high-value but infrequent customers
SELECT 
cus.customer_id,
cus.name AS name_,
COUNT(o.order_id) AS total_orders,
SUM(o.order_amount) AS spending
FROM customers cus
JOIN orders o
ON cus.customer_id = o.customer_id
GROUP BY
cus.customer_id,
cus.name
HAVING COUNT(o.order_id) <= 3 AND SUM(o.order_amount) > 3000
ORDER BY
SUM(o.order_amount) DESC LIMIT 5;

-- Customer Lifetime Value (CLV)
SELECT cus.customer_id, cus.name, COUNT(o.order_id) AS total_orders,
       SUM(o.order_amount) AS total_revenue, AVG(o.order_amount) AS avg_revenue
FROM customers cus
JOIN orders o ON o.customer_id = cus.customer_id
WHERE o.order_status = 'Delivered'
GROUP BY cus.customer_id, cus.name;

-- Repeat vs One-Time Customers
SELECT customer_type, COUNT(*) as num_customers
FROM (
  SELECT customer_id,
  CASE WHEN total_order = 1 THEN 'one-time' ELSE 'repeat' END AS customer_type
  FROM (
    SELECT o.customer_id, COUNT(order_id) AS total_order
    FROM orders o
    WHERE o.order_status = 'Delivered'
    GROUP BY o.customer_id
  ) AS t
) AS final
GROUP BY customer_type;

```
---
### Sales & Market Insights

```sql
-- Monthly revenue trend (last 12 months)
SELECT EXTRACT(MONTH FROM o.order_date) AS month,
       SUM(o.order_amount) AS total_revenue
FROM orders o
WHERE order_status = 'Delivered'
GROUP BY month
ORDER BY month;

-- Average order amount by payment method
SELECT payment_method, AVG(order_amount) AS avg_amount
FROM orders
GROUP BY payment_method;

-- Most frequently used payment methods
SELECT payment_method, COUNT(*) AS total_usage
FROM orders
GROUP BY payment_method
ORDER BY total_usage DESC;

-- Locations with highest number of customers
SELECT location, COUNT(customer_id) AS total_customers
FROM customers
GROUP BY location
ORDER BY total_customers DESC LIMIT 5;

-- Most cancelled orders per restaurant
SELECT res.name, COUNT(o.order_id) AS Total_cancelled_orders
FROM restaurants res
JOIN orders o ON o.restaurant_id = res.restaurant_id
WHERE o.order_status = 'Cancelled'
GROUP BY res.name
ORDER BY Total_cancelled_orders DESC LIMIT 5;

-- Average rating per cuisine type
SELECT 
res.cuisine AS cuisine,
AVG(res.rating) AS avg_rating
FROM restaurants res
GROUP BY
res.cuisine
ORDER BY
AVG(res.rating) DESC;

-- Top 5 restaurants by number of delivered orders
SELECT 
res.name AS res_name,
COUNT(o.order_id) AS total_orders
FROM restaurants res
JOIN orders o
on res.restaurant_id = o.restaurant_id
WHERE 
o.order_status = 'Delivered'
GROUP BY
res.name
ORDER BY 
COUNT(o.order_id) DESC LIMIT 5;

-- Which 5 location have the highest number of customers?
SELECT 
cus.location AS location,
COUNT(cus.customer_id) AS total_customers
FROM customers cus
GROUP BY 
cus.location 
ORDER BY
COUNT(cus.customer_id) DESC LIMIT 5;

-- Most cancelled orders per restaurant.
SELECT
res.name AS name,
COUNT(o.order_id) AS Total_cancelled_orders
FROM restaurants res
JOIN orders o
ON o.restaurant_id = res.restaurant_id
WHERE o.order_status = 'Cancelled'
GROUP BY 
res.name
ORDER BY
COUNT(o.order_id) DESC LIMIT 5;

```
---
### Delivery Operations

```sql
-- Most active riders by number of deliveries
SELECT r.rider_id, r.name, COUNT(o.order_id) AS deliveries
FROM riders r
JOIN delivery d ON r.rider_id = d.rider_id
JOIN orders o ON d.deliverer_id = o.deliverer_id
WHERE o.order_status = 'Delivered'
GROUP BY r.rider_id, r.name
ORDER BY deliveries DESC LIMIT 5;

-- Idle riders (low utilization)
WITH rider_activity AS (
  SELECT r.rider_id, r.name, COUNT(o.order_id) AS deliveries
  FROM riders r
  JOIN delivery d ON r.rider_id = d.rider_id
  JOIN orders o ON d.deliverer_id = o.deliverer_id
  WHERE o.order_status = 'Delivered'
  GROUP BY r.rider_id, r.name
)
SELECT * FROM rider_activity WHERE deliveries < 5;

-- Average delivery value by restaurant location
SELECT res.location, AVG(o.order_amount) AS avg_value
FROM restaurants res
JOIN orders o ON o.restaurant_id = res.restaurant_id
WHERE o.order_status = 'Delivered'
GROUP BY res.location
ORDER BY avg_value DESC;

```
---
### Segmentation & Classification

```sql
-- RFM Segmentation View
CREATE VIEW RFM_segmentation AS 
WITH rfm_base AS (
  SELECT cus.customer_id, cus.name,
         MAX(o.order_date) AS last_ordered,
         CURRENT_DATE - MAX(o.order_date) AS recency_days,
         COUNT(o.order_id) AS total_orders,
         SUM(o.order_amount) AS total_revenue
  FROM customers cus
  JOIN orders o ON o.customer_id = cus.customer_id
  WHERE o.order_status = 'Delivered'
  GROUP BY cus.customer_id, cus.name
),
rfm_scored AS (
  SELECT *,
    CASE WHEN recency_days <= 15 THEN 5
         WHEN recency_days <= 30 THEN 4
         WHEN recency_days <= 45 THEN 3
         WHEN recency_days <= 60 THEN 2
         ELSE 1 END AS r_score,
    CASE WHEN total_orders >= 15 THEN 5
         WHEN total_orders >= 10 THEN 4
         WHEN total_orders >= 5  THEN 3
         WHEN total_orders >= 3 THEN 2
         ELSE 1 END AS f_score,
    CASE WHEN total_revenue >= 3000 THEN 5
         WHEN total_revenue >= 2000 THEN 4
         WHEN total_revenue >= 1000 THEN 3
         WHEN total_revenue >= 500 THEN 2
         ELSE 1 END AS m_score
  FROM rfm_base
),
rfm_segmented AS (
  SELECT *, (r_score + f_score + m_score) AS rfm_total,
    CASE 
      WHEN r_score = 5 AND f_score >= 4 AND m_score >= 4 THEN 'Champion'
      WHEN r_score >= 4 AND f_score >= 3 THEN 'Loyal'
      WHEN r_score <= 2 AND f_score <= 2 THEN 'At Risk'
      ELSE 'Others' END AS segment
  FROM rfm_scored
)
SELECT * FROM rfm_segmented;

-- Customer classification view
CREATE VIEW customer_classification AS
SELECT cus.customer_id, cus.name,
       SUM(o.order_amount) AS total_revenue,
       CASE WHEN SUM(o.order_amount) > 4000 THEN 'Gold'
            WHEN SUM(o.order_amount) > 2000 THEN 'Silver'
            ELSE 'Bronze' END AS customer_classification
FROM customers cus
JOIN orders o ON o.customer_id = cus.customer_id
WHERE o.order_status = 'Delivered'
GROUP BY cus.customer_id, cus.name;

```
---
### KPIs and Insights

#### Monthly revenue by city

#### Average order value per payment method

#### Cancellation rate by cuisine

#### Customer segmentation (Champion, Loyal, At Risk)

#### Idle rider identification

#### Repeat vs One-time customers
---

### Advanced Features

#### RFM_segmentation view for customer scoring

#### customer_classification view for Gold/Silver/Bronze tiers
---

### Author
#### Tilak Laddha
#### Email: tilakladdhaofficial2005@gmail.com
#### LinkedIn: https://github.com/TilakLaddha2005/Int_sql_project

---
