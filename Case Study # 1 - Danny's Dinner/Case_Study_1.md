
# 🍜 Case Study #1: Danny's Diner
![Image](/assets/Case_Study_1.png)

**Link:** [8 Week SQL Challenge - Case Study #1](https://8weeksqlchallenge.com/case-study-1/)

**Entity Relationship Diagram:** 
![Image1](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_ERD.png)


---

### 1. What is the total amount each customer spent at the restaurant?
```sql
SELECT
    customer_id, SUM(price) AS total_amount
FROM dannys_diner.sales s 
LEFT JOIN dannys_diner.menu m ON s.product_id = m.product_id
GROUP BY customer_id
ORDER BY total_amount DESC
LIMIT 5;
```
![Image2](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_Output1.png)
---

### 2. How many days has each customer visited the restaurant?
```sql
SELECT
    customer_id, COUNT(DISTINCT order_date) AS visits
FROM dannys_diner.sales s 
GROUP BY customer_id
ORDER BY visits DESC
LIMIT 5;
```
![Image3](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_Output2.png)
---

### 3. What was the first item from the menu purchased by each customer?
```sql
WITH orders AS (
    SELECT s.customer_id, order_date, product_name,
        DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY order_date) AS rank
    FROM dannys_diner.sales s 
    INNER JOIN dannys_diner.menu m ON s.product_id = m.product_id
)
SELECT customer_id, product_name 
FROM orders
WHERE rank = 1
GROUP BY customer_id, product_name;
```
![Image4](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_Output3.png)
---

### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT COUNT(s.product_id) AS sales, product_name
FROM dannys_diner.sales s 
LEFT JOIN dannys_diner.menu m ON s.product_id = m.product_id
GROUP BY product_name
ORDER BY sales DESC 
LIMIT 1;
```
![Image5](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_Output4.png)
---

### 5. Which item was the most popular for each customer?
```sql
WITH most_purchased AS (
    SELECT COUNT(s.product_id) AS sales, product_name, customer_id,
        DENSE_RANK() OVER (
            PARTITION BY customer_id 
            ORDER BY COUNT(s.product_id) DESC
        ) AS rank
    FROM dannys_diner.sales s 
    LEFT JOIN dannys_diner.menu m ON s.product_id = m.product_id
    GROUP BY product_name, customer_id
    ORDER BY sales DESC
)
SELECT customer_id, product_name 
FROM most_purchased 
WHERE rank = 1;
```
![Image6](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_Output5.png)
---

### 6. Which item was purchased first by the customer after they became a member?
```sql
WITH first_order AS (
    SELECT m.customer_id, m.join_date, s.order_date, mn.product_name,
        DENSE_RANK() OVER (PARTITION BY m.customer_id ORDER BY order_date) AS rank
    FROM dannys_diner.members m 
    LEFT JOIN dannys_diner.sales s ON m.customer_id = s.customer_id
    LEFT JOIN dannys_diner.menu mn ON s.product_id = mn.product_id 
    WHERE order_date > join_date
)
SELECT customer_id, product_name 
FROM first_order 
WHERE rank = 1;
```
![Image7](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_Output6.png)
---

### 7. Which item was purchased just before the customer became a member?
```sql
WITH last_purchased AS (
    SELECT m.customer_id, m.join_date, s.order_date, mn.product_name,
        ROW_NUMBER() OVER (PARTITION BY m.customer_id ORDER BY order_date DESC) AS rank
    FROM dannys_diner.members m 
    LEFT JOIN dannys_diner.sales s ON m.customer_id = s.customer_id
    LEFT JOIN dannys_diner.menu mn ON s.product_id = mn.product_id 
    WHERE join_date > order_date
)
SELECT customer_id, product_name 
FROM last_purchased 
WHERE rank = 1;
```
![Image8](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_Output7.png)
---

### 8. What is the total items and amount spent for each member before they became a member?
```sql
WITH summary AS (
    SELECT m.customer_id, COUNT(s.product_id) AS products, SUM(price)
    FROM dannys_diner.members m 
    LEFT JOIN dannys_diner.sales s ON m.customer_id = s.customer_id
    LEFT JOIN dannys_diner.menu mn ON s.product_id = mn.product_id 
    WHERE join_date > order_date
    GROUP BY m.customer_id
)
SELECT * 
FROM summary;
```
![Image9](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_Output8.png)
---

### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
WITH points AS (
    SELECT m.product_name, m.product_id, m.price,
        CASE 
            WHEN m.product_id = 1 THEN price * 20 
            ELSE price * 10 
        END AS points
    FROM dannys_diner.menu m
)
SELECT customer_id, SUM(points) AS points 
FROM dannys_diner.sales s
INNER JOIN points p ON p.product_id = s.product_id
GROUP BY customer_id;
```
![Image10](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_Output9.png)
---

### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
WITH valid_date AS (
    SELECT 
        customer_id, 
        join_date, 
        join_date + 6 AS valid_date, 
        (DATE_TRUNC('month', '2021-01-31'::DATE) + INTERVAL '1 month' - INTERVAL '1 day')::date AS last_date
    FROM dannys_diner.members
), 
points_customer AS (
    SELECT s.customer_id,
        CASE 
            WHEN order_date BETWEEN join_date AND valid_date THEN price * 20
            WHEN s.product_id = 1 THEN price * 20
            ELSE price * 10 
        END AS points 
    FROM dannys_diner.sales s
    INNER JOIN dannys_diner.menu m ON s.product_id = m.product_id
    INNER JOIN valid_date p ON p.customer_id = s.customer_id
    WHERE join_date <= order_date
      AND order_date <= last_date
)
SELECT customer_id, SUM(points) AS points 
FROM points_customer 
GROUP BY customer_id;
```
![Image11](https://github.com/MahumFatimaKhan/SQL-Challenge/blob/main/assets/Case_Study_1_Output10.png)
