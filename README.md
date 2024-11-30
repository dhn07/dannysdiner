### Danny's Diner Case Study

![image](https://github.com/user-attachments/assets/bdd9ab9f-597b-4701-9a4e-8c4b554da910)

In this case study, I will be conducting some data analysis and answer the questions below using SQL.
This was originally sourced from the [8-week SQL challenge.](https://8weeksqlchallenge.com/case-study-1/) by Danny Ma.
Feel free to contact me on my [LinkedIn](https://www.linkedin.com/in/dhn07/) for any enquiries.

---

1. What is the total amount each customer spent at the restaurant?
2. How many days has each customer visited the restaurant?
3. What was the first item from the menu purchased by each customer?
4. What is the most purchased item on the menu and how many times was it purchased by all customers?
5. Which item was the most popular for each customer?
6. Which item was purchased first by the customer after they became a member?
7. Which item was purchased just before the customer became a member?
8. What is the total items and amount spent for each member before they became a member?
9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

---

##### Q1. What is the total amount each customer spent at the restaurant?

To get this we will get a total of the price of each menu item ordered by each customer.

```sql 
SELECT 
	s.customer_id,
	SUM(m.price) as total_sales
FROM sales as s
LEFT JOIN menu as m
	ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id ASC
```
##### Query Results

| customer_id | total_sales |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |

- Customer A's total spend was $76
- Customer B's total spend was $74
- Customer C's total spend was $36

---

##### Q2. How many days has each customer visited the restaurant?

Since each order has a date, we can count the number of unique dates a customer ordered from the restaurant.

```sql
SELECT 
   customer_id,
   COUNT(DISTINCT order_date) as days_visited
FROM sales
GROUP BY customer_id
ORDER BY customer_id ASC
```

##### Query Results

| customer_id | days_visited |
| ----------- | ------------ |
| A           | 4            |
| B           | 6            |
| C           | 2            |

- Customer A visited on 4 different days
- Customer B visited on 6 different days
- Customer C visited on 2 different days

---

##### Q3. What was the first item from the menu purchased by each customer?

In order to determine the first order for each customer we need to ```rank``` their transactions in by date. This will give us a number for each transaction per customer that we can use to isolate their first order.

```sql
SELECT 
   customer_id,
   product_name
FROM (
    SELECT 
    	customer_id,
        order_date,
      	product_name,
        DENSE_RANK() OVER (
          PARTITION BY s.customer_id 
          ORDER BY s.order_date) AS rank
    FROM sales as s
    LEFT JOIN menu as m
    	ON s.product_id = m.product_id
    ) as first_order    
WHERE rank = 1
GROUP BY customer_id, product_name
```
##### Query Results

| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

- Customer A's first order was curry and sushi
- Customer B's first order was curry
- Customer C's first order was ramen

---

##### Q4. What is the most purchased item on the menu and how many times was it purchased by all customers?

Using ```product_id``` we can count how many times a customer ordered an item to find the most purchased.

```sql
SELECT 
	m.product_name,
	COUNT(s.product_id) as total_orders
FROM sales as s
LEFT JOIN menu as m
	ON s.product_id = m.product_id
GROUP BY m.product_name, s.product_id
ORDER BY total_orders DESC
LIMIT 1
```
##### Query Results

| product_name | total_orders |
| ------------ | ------------ |
| ramen        | 8            |

The most ordered item was ramen with 8 orders.

---

##### Q5. Which item was the most popular for each customer?

This is similar to the question above but for each customer, we should be able use ```rank``` to get the highest order count for each customer. 

```sql
SELECT
   orders.customer_id,
   orders.product_name,
   order_count
FROM (
  SELECT 
      s.customer_id,
      m.product_name,
  	  COUNT(m.product_id) as order_count,
      DENSE_RANK() OVER (
    	PARTITION BY s.customer_id 
    	ORDER BY COUNT(s.customer_id) DESC
      ) AS rank
  FROM sales as s
  LEFT JOIN menu as m
      ON s.product_id = m.product_id
  GROUP BY s.customer_id, m.product_name, m.product_id
  ) as orders
WHERE rank = 1
GROUP BY orders.customer_id, orders.product_name, orders.rank, order_count
ORDER BY orders.customer_id, rank;

```
##### Query Results

| customer_id | product_name | order_count |
| ----------- | ------------ | ----------- |
| A           | ramen        | 3           |
| B           | curry        | 2           |
| B           | ramen        | 2           |
| B           | sushi        | 2           |
| C           | ramen        | 3           |

- Customer A and Customer C's most popular item was ramen
- Customer B couldn't decide and all of the items were equally popular

---

##### Q6. Which item was purchased first by the customer after they became a member?

To solve this we need the member's ```join date``` to filter out the purchases after this time. 
We can then add ranks to determine the first purchase on the filtered table.

```sql
SELECT 
  f.customer_id, f.order_date, f.product_name
FROM (
  SELECT 
    s.customer_id, s.order_date, s.product_id, u.product_name,
 	DENSE_RANK() OVER(
    PARTITION BY s.customer_id
    ORDER BY s.order_date ASC
      ) AS rank
  FROM sales as s
  LEFT JOIN menu as u
  	  ON s.product_id = u.product_id
  LEFT JOIN members as m
      ON s.customer_id = m.customer_id
  WHERE order_date > join_date
  ) as f
WHERE rank = 1
```
##### Query Results

| customer_id | order_date | product_name |
| ----------- | ---------- | ------------ |
| A           | 2021-01-10 | ramen        |
| B           | 2021-01-11 | sushi        |

- Customer A's first order was ramen on 10 January 2021
- Customer B's first order was sushi on 11 January 2021
- Customer C is not a member

---

##### Q7. Which item was purchased just before the customer became a member?

Similar to the last question, we can use the member's ```join date``` to filter out the purchases before they became a member.
We can then add ranks to determine the last purchase on the filtered table.

```sql
SELECT
	f.customer_id, f.order_date, f.product_name
FROM (
  SELECT 
    s.customer_id, s.order_date, s.product_id, u.product_name, m.join_date,
 	DENSE_RANK() OVER(
    PARTITION BY s.customer_id
    ORDER BY s.order_date DESC
      ) AS rank
  FROM sales as s
  LEFT JOIN menu as u
  	  ON s.product_id = u.product_id
  LEFT JOIN members as m
      ON s.customer_id = m.customer_id
  WHERE order_date < join_date
) as f
WHERE rank = 1
```
##### Query Results

| customer_id | order_date | product_name |
| ----------- | ---------- | ------------ |
| A           | 2021-01-01 | sushi        |
| A           | 2021-01-01 | curry        |
| B           | 2021-01-04 | sushi        |


- Customer A's last order before becoming a member was sushi and curry
- Customer B's last order before becoming a member was sushi
- Customer C is not a member

---

##### Q8. What is the total items and amount spent for each member before they became a member?

Continuing the theme of using ```join_date``` to filter the table and then using aggregation functions ```SUM``` and ```COUNT``` to get the total spend and items purchased.

```sql
SELECT
	f.customer_id, SUM(f.price) as total_spent, COUNT(f.order_date) as total_items
FROM (
  SELECT 
    s.customer_id, s.order_date, s.product_id, u.product_name, m.join_date, u.price,
 	DENSE_RANK() OVER(
    PARTITION BY s.customer_id
    ORDER BY s.order_date DESC
      ) AS rank
  FROM sales as s
  LEFT JOIN menu as u
  	  ON s.product_id = u.product_id
  LEFT JOIN members as m
      ON s.customer_id = m.customer_id
  WHERE order_date < join_date
) as f
GROUP BY f.customer_id
ORDER BY f.customer_id ASC;
```
##### Query Results

| customer_id | total_spent | total_items |
| ----------- | ----------- | ----------- |
| A           | 25          | 2           |
| B           | 40          | 3           |

- Customer A spent $25 in total and purchased 2 items before becoming a member
- Customer B spent $40 in total and purchased 3 items before becoming a member
---


##### Q9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

We can use the ```CASE``` function to convert the price into points based on which product was purchased. Then we can use a ```SUM``` to aggregate the resulting point totals for each customer.

```sql
SELECT 
	s.customer_id, 
    SUM(CASE
    	WHEN u.product_id = 1 THEN u.price * 20
    	ELSE u.price * 10 
        END
        ) AS points
FROM sales as s
LEFT JOIN menu as u
	ON s.product_id = u.product_id
LEFT JOIN members as m
	ON s.customer_id = m.customer_id
GROUP BY s.customer_id 
ORDER BY s.customer_id ASC
```
##### Query Results
| customer_id | points |
| ----------- | ------ |
| A           | 860    |
| B           | 940    |
| C           | 360    |

- Customer A has 860 points
- Customer B has 940 points
- Customer C has 360 points
---

##### Q10.  In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

This is a bit more complex, but using ```CASE``` similarly like the question above we can specify the conditions that cause the double points. 
There are two conditions, one being the product is sushi and other condition is the order date being within 1 week of their join date.
The table is also filtered by order date being before the end of January 2021 so we can get the points at that time.

```sql
SELECT 
	f.customer_id,
    SUM(
      CASE
        WHEN f.product_id = 1 THEN f.price * 20
        WHEN f.order_date BETWEEN f.join_date AND f.first_week THEN f.price * 20
        ELSE f.price * 10
	END) AS total_points
FROM (
  SELECT 
      s.customer_id, s.order_date, u.product_id, u.price, m.join_date, m.join_date +6 AS first_week
  FROM sales as s
  LEFT JOIN menu as u
      ON s.product_id = u.product_id
  LEFT JOIN members as m
      ON s.customer_id = m.customer_id
  WHERE s.order_date < '2021-01-31'
) as f
WHERE f.customer_id = 'A' OR f.customer_id = 'B'
```
##### Query Results

| customer_id | total_points |
| ----------- | ------------ |
| A           | 1370         |
| B           | 820          |

- Customer A has 1370 points by end of January
- Customer B has 820 points by end of January

---

### Conclusion

This was a great case study to learn many SQL techniques. It was challenging and rewarding at the same time. It had many logic problems that would appear in day to day work and database querying.
I would recommend others go through this or other similar case studies to learn SQL and other data analyis techniques.

--- 

#####  Links:
- [8-week SQL challenge.](https://8weeksqlchallenge.com/case-study-1/)
- [GitHub](https://github.com/danstands)
- [LinkedIn](https://www.linkedin.com/in/dhn07/)
