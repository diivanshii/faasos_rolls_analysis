# Faasos Roll Management System Analysis

![faasos](https://github.com/user-attachments/assets/fe670a14-7633-4f01-9bac-ab80ec90cc14)

## Overview
This project analyzes and optimizes the operations of a food delivery system that manages orders for rolls (veg and non-veg), drivers, and customers. It involves metrics, insights into customer and driver experiences, ingredient optimization, and order-related data.

## Objectives
1. Analyze roll order metrics.
2. Evaluate driver and customer experiences.
3. Optimize ingredient usage.
4. Examine pricing and ratings.

## Database Schema

### Tables and Data
#### 1. `driver`
```sql
CREATE TABLE driver(driver_id INTEGER, reg_date DATE);
INSERT INTO driver(driver_id, reg_date) VALUES 
    (1, '2021-01-01'),
    (2, '2021-01-03'),
    (3, '2021-01-08'),
    (4, '2021-01-15');
```

#### 2. `ingredients`
```sql
CREATE TABLE ingredients(ingredients_id INTEGER, ingredients_name VARCHAR(60));
INSERT INTO ingredients(ingredients_id, ingredients_name) VALUES 
    (1, 'BBQ Chicken'),
    (2, 'Chilli Sauce'),
    (3, 'Chicken'),
    (4, 'Cheese'),
    (5, 'Kebab'),
    (6, 'Mushrooms'),
    (7, 'Onions'),
    (8, 'Egg'),
    (9, 'Peppers'),
    (10, 'Schezwan Sauce'),
    (11, 'Tomatoes'),
    (12, 'Tomato Sauce');
```

#### 3. `rolls`
```sql
CREATE TABLE rolls(roll_id INTEGER, roll_name VARCHAR(30));
INSERT INTO rolls(roll_id, roll_name) VALUES 
    (1, 'Non Veg Roll'),
    (2, 'Veg Roll');
```

#### 4. `rolls_recipes`
```sql
CREATE TABLE rolls_recipes(roll_id INTEGER, ingredients VARCHAR(24));
INSERT INTO rolls_recipes(roll_id, ingredients) VALUES 
    (1, '1,2,3,4,5,6,8,10'),
    (2, '4,6,7,9,11,12');
```

#### 5. `driver_order`
```sql
CREATE TABLE driver_order(order_id INTEGER, driver_id INTEGER, pickup_time DATETIME, distance VARCHAR(7), duration VARCHAR(10), cancellation VARCHAR(23));
INSERT INTO driver_order(order_id, driver_id, pickup_time, distance, duration, cancellation) VALUES 
    (1, 1, '2021-01-01 18:15:34', '20km', '32 minutes', ''),
    (2, 1, '2021-01-01 19:10:54', '20km', '27 minutes', ''),
    (3, 1, '2021-01-03 00:12:37', '13.4km', '20 mins', 'NaN'),
    (6, 3, NULL, NULL, NULL, 'Cancellation');
```

#### 6. `customer_orders`
```sql
CREATE TABLE customer_orders(order_id INTEGER, customer_id INTEGER, roll_id INTEGER, not_include_items VARCHAR(4), extra_items_included VARCHAR(4), order_date DATETIME);
INSERT INTO customer_orders(order_id, customer_id, roll_id, not_include_items, extra_items_included, order_date) VALUES 
    (1, 101, 1, '', '', '2021-01-01 18:05:02'),
    (2, 101, 1, '', '', '2021-01-01 19:00:52'),
    (3, 102, 1, '', 'NaN', '2021-01-02 23:51:23');
```

## Business Problems and Solutions

### Problem 1: Identifying Roll Popularity Trends
**Objective:** Understand which types of rolls (veg or non-veg) are most popular among customers.

**Solution:** Analyze the order data to find the most frequently ordered roll types.
```sql
SELECT r.roll_name, COUNT(co.roll_id) AS order_count
FROM customer_orders co
JOIN rolls r ON co.roll_id = r.roll_id
GROUP BY r.roll_name
ORDER BY order_count DESC;
```

### Problem 2: Reducing Order Cancellations
**Objective:** Identify drivers with high cancellation rates to improve delivery reliability.

**Solution:** Calculate the percentage of cancellations for each driver and suggest targeted interventions.
```sql
SELECT driver_id, 
       COUNT(order_id) AS total_orders,
       SUM(CASE WHEN cancellation IS NOT NULL AND cancellation != '' THEN 1 ELSE 0 END) AS cancellations,
       (SUM(CASE WHEN cancellation IS NOT NULL AND cancellation != '' THEN 1 ELSE 0 END) * 100.0 / COUNT(order_id)) AS cancellation_rate
FROM driver_order
GROUP BY driver_id
ORDER BY cancellation_rate DESC;
```

### Problem 3: Forecasting Future Ingredient Demand
**Objective:** Predict future ingredient requirements using time-series analysis of past usage trends.

**Solution:** Generate projections based on monthly ingredient usage data.
```sql
WITH MonthlyIngredientUsage AS (
    SELECT 
        i.ingredients_name, 
        DATEPART(YEAR, co.order_date) AS order_year, 
        DATEPART(MONTH, co.order_date) AS order_month,
        COUNT(*) AS usage_count
    FROM customer_orders co
    JOIN rolls_recipes rr ON co.roll_id = rr.roll_id
    JOIN ingredients i ON ',' || rr.ingredients || ',' LIKE '%,' || CAST(i.ingredients_id AS VARCHAR) || ',%'
    WHERE co.order_id IN (
        SELECT order_id FROM driver_order WHERE cancellation IS NULL OR cancellation = ''
    )
    GROUP BY i.ingredients_name, DATEPART(YEAR, co.order_date), DATEPART(MONTH, co.order_date)
)
SELECT ingredients_name, order_year, order_month, usage_count, 
       AVG(usage_count) OVER (PARTITION BY ingredients_name ORDER BY order_year, order_month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS projected_demand
FROM MonthlyIngredientUsage
ORDER BY ingredients_name, order_year, order_month;
```

### Problem 4: Ingredient Demand Prediction
**Objective:** Forecast ingredient demand based on past orders for efficient inventory management.

**Solution:** Calculate the frequency of each ingredient in completed orders.
```sql
SELECT i.ingredients_name, COUNT(*) AS usage_count
FROM customer_orders co
JOIN rolls_recipes rr ON co.roll_id = rr.roll_id
JOIN ingredients i ON ',' || rr.ingredients || ',' LIKE '%,' || CAST(i.ingredients_id AS VARCHAR) || ',%'
WHERE co.order_id IN (
    SELECT order_id FROM driver_order WHERE cancellation IS NULL OR cancellation = ''
)
GROUP BY i.ingredients_name
ORDER BY usage_count DESC;
```

### Problem 5: Peak Order Times
**Objective:** Identify peak times for orders to optimize staffing and resource allocation.

**Solution:** Analyze the distribution of orders by time of day.
```sql
SELECT DATEPART(HOUR, order_date) AS order_hour, COUNT(order_id) AS order_count
FROM customer_orders
GROUP BY DATEPART(HOUR, order_date)
ORDER BY order_hour;
```

### Problem 6: Identifying Drivers with Exceptional Performance
**Objective:** Recognize drivers who consistently deliver faster than average across comparable distances.

**Solution:** Benchmark driver performance against average delivery times.
```sql
WITH DistanceBenchmarks AS (
    SELECT distance, AVG(CAST(duration AS FLOAT)) AS avg_duration
    FROM driver_order
    WHERE cancellation IS NULL OR cancellation = ''
    GROUP BY distance
)
SELECT do.driver_id, do.distance, AVG(CAST(do.duration AS FLOAT)) AS avg_driver_duration, 
       db.avg_duration AS benchmark_duration, 
       (db.avg_duration - AVG(CAST(do.duration AS FLOAT))) AS performance_gap
FROM driver_order do
JOIN DistanceBenchmarks db ON do.distance = db.distance
WHERE cancellation IS NULL OR cancellation = ''
GROUP BY do.driver_id, do.distance, db.avg_duration
HAVING AVG(CAST(do.duration AS FLOAT)) < db.avg_duration
ORDER BY performance_gap DESC;
```

### Problem 7: Monitoring Customer Order Frequency Changes
**Objective:** Identify customers whose ordering patterns have significantly changed over time.

**Solution:** Analyze changes in monthly order frequency for each customer.
```sql
WITH MonthlyOrders AS (
    SELECT customer_id, 
           DATEPART(YEAR, order_date) AS order_year, 
           DATEPART(MONTH, order_date) AS order_month, 
           COUNT(order_id) AS order_count
    FROM customer_orders
    GROUP BY customer_id, DATEPART(YEAR, order_date), DATEPART(MONTH, order_date)
),
OrderChanges AS (
    SELECT customer_id, order_year, order_month, order_count,
           LAG(order_count) OVER (PARTITION BY customer_id ORDER BY order_year, order_month) AS prev_order_count
    FROM MonthlyOrders
)
SELECT customer_id, order_year, order_month, prev_order_count, order_count, 
       (order_count - prev_order_count) AS order_change
FROM OrderChanges
WHERE prev_order_count IS NOT NULL
ORDER BY ABS(order_change) DESC;
```


### Problem 8: Revenue Analysis by Roll Type
**Objective:** Calculate total revenue generated by each roll type to identify high-performing items.

**Solution:** Sum the revenues grouped by roll type.
```sql
SELECT r.roll_name, SUM(price) AS total_revenue
FROM customer_orders co
JOIN rolls r ON co.roll_id = r.roll_id
JOIN pricing p ON co.roll_id = p.roll_id
GROUP BY r.roll_name
ORDER BY total_revenue DESC;
```

### Problem 9: Average Order Size
**Objective:** Determine the average number of rolls per order to gauge customer purchasing behavior.

**Solution:** Calculate the average rolls ordered per order.
```sql
SELECT AVG(order_size) AS avg_order_size
FROM (
    SELECT order_id, COUNT(roll_id) AS order_size
    FROM customer_orders
    GROUP BY order_id
) AS subquery;
```

### Problem 10: Driver Workload Distribution
**Objective:** Evaluate how workload is distributed among drivers to improve resource allocation.

**Solution:** Count the number of orders handled by each driver.
```sql
SELECT driver_id, COUNT(order_id) AS orders_handled
FROM driver_order
WHERE cancellation IS NULL OR cancellation = ''
GROUP BY driver_id
ORDER BY orders_handled DESC;
```

## Outcomes
The project results provided insights into roll order patterns, driver performance, customer preferences, and ingredient optimization opportunities. Key findings include:
1. High demand for certain roll types.
2. Drivers' performance metrics for completed orders.
3. Patterns in order customization requests by customers.
4. Ingredient demand trends for inventory planning.
5. Peak order times for operational efficiency.
6. Identification of loyal customers.
7. Delivery efficiency insights by distance.
8. Revenue contributions by roll type.
9. Average order size trends.
10. Workload distribution among drivers.
