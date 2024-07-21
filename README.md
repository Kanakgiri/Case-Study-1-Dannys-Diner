# Case Study #1 - Dannys Diner

## Problem Statement

Danny wants to use the data to answer a few simple questions about the customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

Danny has provided you with a sample of his overall customer data due to privacy issues - but he hopes that these examples are enough for you to write fully functioning SQL queries to help him answer his questions!

## About Data

Danny has shared with you 3 key datasets for this case study:

- Sales
- Menu
- Members

## Case Study Questions

1. What is the total amount each customer spent at the restaurant?
2. How many days has each customer visited the restaurant?
3. What was the first item from the menu purchased by each customer?
4. What is the most purchased item on the menu and how many times was it purchased by all customers?
5. Which item was the most popular for each customer?
6. Which item was purchased first by the customer after they became a member?
7. Which item was purchased just before the customer became a member?
8. What is the total items and amount spent for each member before they became a member?
9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?


```

CREATE SCHEMA dannys_diner;
SET search_path = dannys_diner;

CREATE TABLE sales (
  customer_id VARCHAR(1),
  order_date DATE,
  product_id INTEGER
);

INSERT INTO sales
  (customer_id, order_date, product_id)
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  product_id INTEGER,
  product_name VARCHAR(5),
  price INTEGER
);

INSERT INTO menu
  (product_id, product_name, price)
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  customer_id VARCHAR(1),
  join_date DATE
);

INSERT INTO members
  (customer_id, join_date)
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');
  
  
-- What is the total amount each customer spent at the restaurant?

select customer_id, sum(price)
from sales join menu
on sales.product_id = menu.product_id
group by customer_id
order by sum(price) desc;

-- How many days has each customer visited the restaurant?

select customer_id, count(distinct order_date) 
from sales join menu on sales.product_id =  menu.product_id
group by customer_id
order by count(distinct order_date) desc;

-- What was the first item from the menu purchased by each customer?

with cte as(
select customer_id, order_date, product_id,
		rank() over(partition by customer_id order by order_date) as rn
from sales)
select customer_id, product_name from cte s
join menu on s.product_id = menu.product_id
where rn =1;

-- What is the most purchased item on the menu and how many times 
-- was it purchased by all customers?

select product_name as most_pur, count(product_name) as times from sales
join menu on sales.product_id = menu.product_id
group by product_name
order by times desc limit 1;

-- Which item was the most popular for each customer?

with cte as (
select customer_id, product_name, count(product_name) as times,
		dense_rank() over(partition by customer_id order by count(product_name) desc) as rn
from sales
join menu on sales.product_id = menu.product_id
group by customer_id, product_name
)
select customer_id, product_name, times from cte
where rn = 1;

-- Which item was purchased first by the customer after they became a member?
with cte1 as(
with cte as(
select sales.customer_id, sales.product_id, sales.order_date,
		rank() over(partition by sales.customer_id order by sales.order_date) as rn
from sales join members on sales.customer_id = members.customer_id
where join_date <= order_date
)
select customer_id, product_id, order_date from cte
where rn=1
)
select cte1.customer_id, cte1.order_date, menu.product_name 
from cte1 join menu on cte1.product_id = menu.product_id
order by order_date;

-- Which item was purchased just before the customer became a member?

with cte as(
select sales.customer_id, sales.order_date, sales.product_id,
		rank() over(partition by sales.customer_id order by order_date desc) as rn
from sales join members on sales.customer_id = members.customer_id
where order_date < join_date
)
select cte.customer_id, cte.order_date, menu.product_name from cte
join menu on cte.product_id = menu.product_id
where rn =1
order by customer_id;

-- What is the total items and amount spent for each member before they became a member?

select sales.customer_id, count(sales.product_id) as total_items, sum(menu.price) as total_price
from sales join members on sales.customer_id = members.customer_id
		   join menu on sales.product_id = menu.product_id
           where join_date > order_date
           group by sales.customer_id;
           
-- If each $1 spent equates to 10 points and sushi has a 2x points 
-- multiplier - how many points would each customer have?

with cte as(
select sales.customer_id, menu.product_name, menu.price,
		case when menu.product_name = 'sushi' then menu.price * 10 * 2
			 else menu.price * 10 end as points
from sales join menu on sales.product_id = menu.product_id
)
select customer_id, sum(points) from cte
group by customer_id
order by sum(points) desc;

-- In the first week after a customer joins the program (including their join date) 
-- they earn 2x points on all items, not just sushi - how many points do 
-- customer A and B have at the end of January?

with cte as(
  select sales.customer_id,
			case when order_date-join_date < 7 then price * 10 * 2
				 when product_name = 'sushi' then price * 10 * 2
                 else price * 10 end as points
  from sales join members on sales.customer_id = members.customer_id
			 join menu on sales.product_id = menu.product_id
             where join_date < order_date
             and order_date < '2021-02-01'
)
select customer_id, sum(points) from cte
group by customer_id
order by sum(points) desc;
  
-- Recreating another Table
  
select sales.customer_id, sales.order_date, menu.product_name, menu.price,
	   case when join_date is null then 'N'
			when join_date > order_date then 'N'
			else 'Y' end as `member`
from sales left join members on sales.customer_id = members.customer_id
		   join menu on sales.product_id = menu.product_id;
  
-- Ranking

with cte as(
select sales.customer_id, sales.order_date, menu.product_name, menu.price,
	   case when join_date is null then 'N'
			when join_date > order_date then 'N'
			else 'Y' end as `member`
from sales left join members on sales.customer_id = members.customer_id
		   join menu on sales.product_id = menu.product_id
)
select *,
		case when `member` = 'N' then null
			 else rank() over(partition by customer_id, `member` order by order_date)
             end as ranking
from cte;

```
