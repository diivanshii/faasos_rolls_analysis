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
