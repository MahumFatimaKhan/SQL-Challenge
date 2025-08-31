# 🍕Case Study #2: Pizza Runner
![Image](/assets/Case_Study_2.png)

**Link:** [8 Week SQL Challenge - Case Study #1](https://8weeksqlchallenge.com/case-study-2/)

**Entity Relationship Diagram:** 
![Image1](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_2_ERD.png)

Before writing SQL queries, I explored the customer_orders and runner_orders tables to understand their structure. I noticed missing values in order status and timestamp columns, along with inconsistencies like text stored instead of numeric or boolean data. To ensure reliable analysis, I cleaned the data by standardizing types and handling nulls, creating a solid base for further querying.
To clean null values:
```sql
CREATE TEMP TABLE customer_orders_temp AS
SELECT
order_id,
customer_id,
pizza_id,
CASE
WHEN exclusions LIKE 'null' or exclusions LIKE '' THEN null
ELSE exclusions
END AS exclusions,
CASE
WHEN extras LIKE 'null' or extras LIKE '' THEN null
ELSE extras
END AS extras,
order_time
FROM pizza_runner.customer_orders;
```

```sql
CREATE TEMP TABLE runner_orders_temp AS
SELECT
order_id,
runner_id,
CASE
WHEN pickup_time ='' OR pickup_time = 'null' THEN NULL
ELSE pickup_time
END AS pickup_time,
CASE
WHEN distance =''  OR distance = 'null' THEN NULL
WHEN distance ILIKE '%km' THEN TRIM('km' FROM distance)
ELSE distance
END AS distance,
CASE
WHEN duration =''  OR duration = 'null' THEN NULL
WHEN duration ILIKE '%mins' THEN TRIM('mins' FROM duration)
WHEN duration ILIKE '%minutes' THEN TRIM('minutes' FROM duration)
WHEN duration ILIKE '%minute' THEN TRIM('minute' FROM duration)
ELSE duration
END AS duration,
CASE
WHEN cancellation =''  OR cancellation = 'null' THEN NULL
ELSE cancellation
END AS cancellation
FROM pizza_runner.runner_orders;
```


To view the data types before fixing them:
```sql
SELECT
column_name,
data_type
FROM
information_schema.columns
WHERE
table_name = 'runner_orders_temp';
```
![Image2](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_2_Cleaning.png)


Changing the data types of pickup time, distance, duration, cancellation




```sql
ALTER TABLE runner_orders_temp
ALTER COLUMN pickup_time
TYPE TIMESTAMP
USING NULLIF(pickup_time, '')::TIMESTAMP,
ALTER COLUMN distance TYPE FLOAT USING distance::FLOAT,
ALTER COLUMN duration TYPE INT USING duration::INT;
```
![Image3](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_2_Cleaning2.png)


## A. Pizza Metrics
### How many pizzas were ordered?
```sql
SELECT COUNT(ORDER_ID) AS TOTAL_PIZZA_ORDERS FROM customer_orders_temp
```
![Image4](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_2_A1.png)

### 2. How many unique customer orders were made?




```sql
SELECT COUNT(DISTINCT ORDER_ID) AS CUSTOMER_ORDERS FROM customer_orders_temp
```


### 3. How many successful orders were delivered by each runner?




```sql
SELECT COUNT(ORDER_ID) AS ORDERS,RUNNER_ID FROM runner_orders_temp
WHERE CANCELLATION IS NULL
GROUP BY 2
```


### 4. How many of each type of pizza was delivered?
```sql
SELECT pizza_name,COUNT(c.pizza_id) AS pizzas FROM runner_orders_temp r
INNER JOIN customer_orders_temp c on c.order_id = r.order_id
inner join pizza_names p on p.pizza_id =c.pizza_id
WHERE CANCELLATION IS NULL
GROUP BY 1
```
### 5. How many Vegetarian and Meatlovers were ordered by each customer
```sql
SELECT pizza_name,customer_id,COUNT(c.pizza_id) AS pizzas FROM runner_orders_temp r
INNER JOIN customer_orders_temp c on c.order_id = r.order_id
inner join pizza_names p on p.pizza_id =c.pizza_id
WHERE CANCELLATION IS NULL
GROUP BY 1,2
```








### 6. What was the maximum number of pizzas delivered in a single order?


```sql
SELECT c.order_id,count(pizza_id) AS pizzas FROM runner_orders_temp r
INNER JOIN customer_orders_temp c on c.order_id = r.order_id
WHERE CANCELLATION IS NULL
GROUP BY 1
order by pizzas desc limit 1
```






### 7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?


```sql
select customer_id,
sum(case when exclusions is null and extras is null then 1
else 0 end) as no_change,
sum(case when exclusions is not null or extras is not null then 1
else 0 end ) as min_one_change
from runner_orders_temp r
INNER JOIN customer_orders_temp c on c.order_id = r.order_id
WHERE CANCELLATION IS NULL
group by 1
order by 1
```




### 8.How many pizzas were delivered that had both exclusions and extras?
```sql
select count(distinct c.order_id) as changes_in_both
from runner_orders_temp r
INNER JOIN customer_orders_temp c on c.order_id = r.order_id
WHERE CANCELLATION IS null
and exclusions is not null
and extras is not null
order by 1
```




### 9. What was the total volume of pizzas ordered for each hour of the day?
```sql
SELECT EXTRACT(HOUR FROM ORDER_TIME) AS HOUR,COUNT(ORDER_ID) AS ORDERS FROM customer_orders_temp
GROUP BY 1
ORDER BY 1
```




### 10. What was the volume of orders for each day of the week?


```sql
SELECT TO_CHAR(order_time, 'Day') AS DAY,COUNT(ORDER_ID) AS ORDERS FROM customer_orders_temp
GROUP BY 1
ORDER BY 1
```






## B. Runner and Customer Experience
### 1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)




```sql
SELECT
FLOOR((registration_date - DATE '2021-01-01') / 7) + 1 AS registration_week,
COUNT(runner_id) AS runner_signup
FROM runners
WHERE registration_date >= DATE '2021-01-01'
GROUP BY 1
ORDER BY 1;
```
How it works:


(registration_date - DATE '2021-01-01') → gets number of days since Jan 1


/ 7 → converts days to weeks


FLOOR(...) → rounds down to full weeks


+' 1 → makes Jan 1 fall in week 1






### 2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?




```sql
WITH runner_time AS (
SELECT
r.runner_id,
EXTRACT(EPOCH FROM (r.pickup_time - c.order_time)) / 60 AS minutes_diff
FROM
runner_orders_temp r
JOIN
customer_orders_temp c ON r.order_id = c.order_id
)
SELECT
runner_id,
ROUND(AVG(minutes_diff)::numeric, 2) AS avg_time
FROM runner_time
GROUP BY 1;
```


### 3. Is there any relationship between the number of pizzas and how long the order takes to prepare?


```sql
WITH runner_time AS (
SELECT
count(c.order_id) as pizza_orders,
EXTRACT(EPOCH FROM (r.pickup_time - c.order_time)) / 60 AS minutes_diff
FROM
runner_orders_temp r
JOIN
customer_orders_temp c ON r.order_id = c.order_id
group by pickup_time,order_time
)
SELECT
pizza_orders,
ROUND(AVG(minutes_diff)::numeric, 2) AS avg_time
FROM runner_time
GROUP BY 1
order by 1;
```




### 4. What was the average distance travelled for each customer?


```sql
select customer_id, ROUND(AVG(distance)::numeric, 2) AS avg_distance
from customer_orders_temp c left join
runner_orders_temp r on  c.order_id = r.order_id
group by 1
order by 1
```










### 5. What was the difference between the longest and shortest delivery times for all orders?
```sql
select max(duration)-min(duration) as duration_diff
from
runner_orders_temp
```




### 6. What was the average speed for each runner for each delivery and do you notice any trend for these values?


```sql
select runner_id,order_id,(distance/duration*60) as speed
from
runner_orders_temp WHERE CANCELLATION IS NULL
order by 1
```


### 7. What is the successful delivery percentage for each runner?


```sql
SELECT
runner_id,
ROUND(
100.0 * SUM(
CASE
WHEN distance IS NOT NULL AND pickup_time IS NOT NULL AND cancellation IS NULL THEN 1
ELSE 0
END
) / COUNT(*),
0
) AS success_perc
FROM runner_orders_temp
GROUP BY 1
order by 1;
```


##  C. Ingredient Optimisation


### 1. What are the standard ingredients for each pizza?


```sql
with toppings as (SELECT
pr.pizza_id,pizza_name,
REGEXP_SPLIT_TO_TABLE(toppings, '[,\s]+')::INTEGER AS topping_id
FROM pizza_recipes pr
inner join pizza_names pn on pr.pizza_id=pn.pizza_id
)
select pizza_name, topping_name
from toppings t
inner join pizza_toppings pt on t.topping_id =pt.topping_id
```






### 2. What was the most commonly added extra?


```sql
with extras as (
select order_id,regexp_split_to_table(extras,'[,\s]+')::integer as topping_id from customer_orders_temp
where extras is not null)
select topping_name, count(order_id) as count_of_toppings from extras e
left join pizza_toppings t on e.topping_id=t.topping_id
group by 1
```


### 3. What was the most common exclusion?


```sql
with exclusions as (
select order_id,regexp_split_to_table(exclusions,'[,\s]+')::integer as topping_id from customer_orders_temp
where exclusions is not null)
select topping_name, count(order_id) as count_of_toppings from exclusions e
left join pizza_toppings t on e.topping_id=t.topping_id
group by 1
```


### 4. Generate an order item for each record in the customers_orders table in the format of one of the following:
Meat Lovers
Meat Lovers - Exclude Beef
Meat Lovers - Extra Bacon
Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers


```sql
WITH excluded_toppings AS (
SELECT
co.order_id,
pn.pizza_name,
(regexp_split_to_table(co.exclusions, '[,\s]+'))::INT AS topping_id,
'exclude' AS type
FROM customer_orders_temp co
JOIN pizza_names pn ON co.pizza_id = pn.pizza_id
WHERE co.exclusions IS NOT NULL
),
no_toppings AS (
SELECT
co.order_id,
pn.pizza_name,
0 AS topping_id,
'no_topping' AS type
FROM customer_orders_temp co
JOIN pizza_names pn ON co.pizza_id = pn.pizza_id
WHERE co.exclusions IS NULL and co.extras is null
),
extra_toppings AS (
SELECT
co.order_id,
pn.pizza_name,
(regexp_split_to_table(co.extras, '[,\s]+'))::INT AS topping_id,
'extra' AS type
FROM customer_orders_temp co
JOIN pizza_names pn ON co.pizza_id = pn.pizza_id
WHERE co.extras IS NOT NULL
),
combined AS (
SELECT * FROM excluded_toppings
UNION ALL
SELECT * FROM extra_toppings
union all
select * from no_toppings
),
details as (
SELECT
c.order_id,
c.pizza_name,
pt.topping_name,
c.type
FROM combined c
left JOIN pizza_toppings pt ON c.topping_id = pt.topping_id
ORDER BY c.order_id DESC),
grouped as (
SELECT
d.order_id,
d.pizza_name,
STRING_AGG(CASE WHEN d.type = 'exclude' THEN d.topping_name END, ', ') AS excluded,
STRING_AGG(CASE WHEN d.type = 'extra' THEN d.topping_name END, ', ') AS extras
FROM details d
GROUP BY order_id, pizza_name
)
SELECT
order_id,
CASE
WHEN excluded IS NULL AND extras IS NULL THEN pizza_name
WHEN excluded IS NOT NULL AND extras IS NULL THEN pizza_name || ' - Exclude ' || excluded
WHEN excluded IS NULL AND extras IS NOT NULL THEN pizza_name || ' - Extra ' || extras
ELSE pizza_name || ' - Exclude ' || excluded || ' - Extra ' || extras
END AS order_item
FROM grouped
ORDER BY order_id;
```












## D. Pricing and Ratings
### 1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
```sql
WITH valid_deliveries AS (
SELECT order_id
FROM runner_orders_temp
WHERE cancellation IS NULL
),
delivered_pizzas AS (
SELECT
c.order_id,
c.pizza_id,
p.pizza_name,
CASE
WHEN p.pizza_name = 'Meatlovers' THEN 12
WHEN p.pizza_name = 'Vegetarian' THEN 10
END AS price
FROM customer_orders_temp c
JOIN valid_deliveries v ON c.order_id = v.order_id
JOIN pizza_names p ON c.pizza_id = p.pizza_id
)
SELECT
SUM(price) AS total_revenue
FROM delivered_pizzas;
```








### 2. What if there was an additional $1 charge for any pizza extras?
Add cheese is $1 extra
```sql
WITH valid_deliveries AS (
SELECT order_id
FROM runner_orders_temp
WHERE cancellation IS NULL
),
delivered_pizzas AS (
SELECT
c.order_id,
c.pizza_id,
p.pizza_name,
CASE
WHEN p.pizza_name = 'Meatlovers' THEN 12
WHEN p.pizza_name = 'Vegetarian' THEN 10
END AS base_price,
c.extras
FROM customer_orders_temp c
JOIN valid_deliveries v ON c.order_id = v.order_id
JOIN pizza_names p ON c.pizza_id = p.pizza_id
),
final_revenue AS (
SELECT
order_id,
pizza_name,
base_price,
extras,
-- Count number of extras: count commas and add 1 (if not null)
CASE
WHEN extras IS NULL THEN 0
ELSE array_length(string_to_array(extras, ','), 1)
END AS extras_count,
base_price +
CASE
WHEN extras IS NULL THEN 0
ELSE array_length(string_to_array(extras, ','), 1)
END AS total_price
FROM delivered_pizzas
)
SELECT
SUM(total_price) AS total_revenue
FROM final_revenue;
```




### 3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.




```sql
DROP TABLE IF EXISTS runner_ratings;
CREATE TABLE runner_ratings (
rating_id SERIAL PRIMARY KEY,
order_id INTEGER NOT NULL,
runner_id INTEGER NOT NULL,
rating INTEGER CHECK (rating >= 1 AND rating <= 5),
rating_comment TEXT,
rating_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```




### 4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
customer_id
order_id
runner_id
rating
order_time
pickup_time
Time between order and pickup
Delivery duration
Average speed
Total number of pizzas
```sql
WITH valid_deliveries AS (
SELECT *
FROM runner_orders_temp
WHERE cancellation IS NULL
),
order_info AS (
SELECT
co.order_id,
co.customer_id,
co.order_time,
COUNT(*) AS total_pizzas
FROM customer_orders_temp co
JOIN valid_deliveries ro ON co.order_id = ro.order_id
GROUP BY co.order_id, co.customer_id, co.order_time
),
delivery_data AS (
SELECT
ro.order_id,
ro.runner_id,
CAST(ro.pickup_time AS TIMESTAMP) AS pickup_time,
CAST(o.order_time AS TIMESTAMP) AS order_time,
o.customer_id,
o.total_pizzas,
-- Time between order and pickup
CAST(ro.pickup_time AS TIMESTAMP) - CAST(o.order_time AS TIMESTAMP) AS time_to_pickup,
CAST(ro.duration AS INTEGER) AS delivery_duration_mins,
CAST(ro.distance AS FLOAT) AS distance_km,
ROUND((CAST(ro.distance AS FLOAT) / (CAST(ro.duration AS FLOAT) / 60))::numeric, 2) AS avg_speed_kmph
FROM valid_deliveries ro
JOIN order_info o ON ro.order_id = o.order_id
),
final_output AS (
SELECT
dd.customer_id,
dd.order_id,
dd.runner_id,
rr.rating,
dd.order_time,
dd.pickup_time,
dd.time_to_pickup,
dd.delivery_duration_mins,
dd.avg_speed_kmph,
dd.total_pizzas
FROM delivery_data dd
LEFT JOIN runner_ratings rr
ON dd.order_id = rr.order_id
AND dd.runner_id = rr.runner_id
)
SELECT * FROM final_output
ORDER BY order_id;
```




### 5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?
```sql
WITH valid_deliveries AS (
SELECT *
FROM runner_orders_temp
WHERE cancellation IS NULL
),
pizza_prices AS (
SELECT
co.order_id,
pn.pizza_name,
CASE
WHEN pn.pizza_name = 'Meatlovers' THEN 12
WHEN pn.pizza_name = 'Vegetarian' THEN 10
END AS price
FROM customer_orders_temp co
JOIN valid_deliveries ro ON co.order_id = ro.order_id
JOIN pizza_names pn ON co.pizza_id = pn.pizza_id
),
revenue AS (
SELECT
SUM(price) AS total_revenue
FROM pizza_prices
),
runner_payments AS (
SELECT
SUM(CAST(distance AS FLOAT) * 0.30) AS total_runner_cost
FROM valid_deliveries
),
final_result AS (
SELECT
r.total_revenue,
rp.total_runner_cost,
r.total_revenue - rp.total_runner_cost AS profit_leftover
FROM revenue r, runner_payments rp
)
SELECT * FROM final_result;
```







