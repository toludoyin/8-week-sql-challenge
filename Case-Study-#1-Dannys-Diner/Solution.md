## Solutions
### 1. What is the total amount each customer spent at the restaurant?

```sql
select
distinct s.customer_id,
sum(m.price) as total_amount
from dannys_diner.sales s
join dannys_diner.menu m using(product_id)
group by 1
order by 1
```

| customer_id | total_amount |
| ----------- | ------------ |
| A           | 76           |
| B           | 74           |
| C           | 36           |

---
### 2. How many days has each customer visited the restaurant?
```sql
select
distinct s.customer_id,
count(distinct s.order_date) as num_of_visit_days
from dannys_diner.sales s
join dannys_diner.menu m using(product_id)
group by 1
```
| customer_id | num_of_visit_days |
| ----------- | ----------------- |
| A           | 4                 |
| B           | 6                 |
| C           | 2                 |

---
### 3. What was the first item from the menu purchased by each customer?
```sql
select order_date,
customer_id,
product_name
    from (
    select
    order_date::date,
    customer_id,
    product_name,
    row_number() over(partition by customer_id order by order_date::date) as row_num
    from dannys_diner.sales s
    join dannys_diner.menu m using(product_id)
    order by order_date::date, customer_id
) as first_order
where row_num = 1
```

| order_date               | customer_id | product_name |
| ------------------------ | ----------- | ------------ |
| 2021-01-01T00:00:00.000Z | A           | curry        |
| 2021-01-01T00:00:00.000Z | B           | curry        |
| 2021-01-01T00:00:00.000Z | C           | ramen        |

---
### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
select
m.product_name,
sum(m.price) as total_price,
count(*) as num_of_times_purchase
from dannys_diner.sales s
join dannys_diner.menu m using(product_id)
group by 1
order by 2 desc
limit 1
```

| product_name | total_price | num_of_times_purchase |
| ------------ | ----------- | --------------------- |
| ramen        | 96          | 8                     |

---
### 5. Which item was the most popular for each customer?
```sql
select customer_id,
    product_name,
    num_of_times_product_ordered
    from (
    select
    customer_id,
    product_name,
    count(product_name) as num_of_times_product_ordered,
    row_number() over(partition by customer_id order by count(product_name) desc) as row_num
    from dannys_diner.sales s
    join dannys_diner.menu m using(product_id)
    group by 1,2
    order by 1
) as popular
where row_num = 1
```
| customer_id | product_name | num_of_times_product_ordered |
| ----------- | ------------ | ---------------------------- |
| A           | ramen        | 3                            |
| B           | ramen        | 2                            |
| C           | ramen        | 3                            |

---
### 6. Which item was purchased first by the customer after they became a member?
```sql
select customer_id,
    product_name,
    num_of_times_product_ordered
    from (
    select
    customer_id,
    product_name,
    join_date,
    s.order_date,
    count(product_name) as num_of_times_product_ordered,
    row_number() over(partition by customer_id order by order_date) as row_num
    from dannys_diner.sales s
    join dannys_diner.menu m using(product_id)
    left join  dannys_diner.members me using(customer_id)
    where me.join_date < s.order_date
    group by 1, 2, 3, 4
    order by 1
) as popular
where row_num = 1
```

| customer_id | product_name | num_of_times_product_ordered |
| ----------- | ------------ | ---------------------------- |
| A           | ramen        | 1                            |
| B           | sushi        | 1                            |

---
### 7. Which item was purchased just before the customer became a member?
```sql
select * from (
    select
    customer_id,
    product_name,join_date, s.order_date,
    count(product_name) as num_of_times_product_ordered,
    row_number() over(partition by customer_id order by order_date desc) as row_num
    from dannys_diner.sales s
    join dannys_diner.menu m using(product_id)
    left join  dannys_diner.members me using(customer_id)
    where me.join_date > s.order_date
    group by 1,2,3,4
    order by 1
) as popular
where row_num = 1
```

| customer_id | product_name | join_date                | order_date               | num_of_times_product_ordered |
| ----------- | ------------ | ------------------------ | ------------------------ | ---------------------------- |
| A           | sushi        | 2021-01-07T00:00:00.000Z | 2021-01-01T00:00:00.000Z | 1                            |
| B           | sushi        | 2021-01-09T00:00:00.000Z | 2021-01-04T00:00:00.000Z | 1                            |

---
### 8. What is the total items and amount spent for each member before they became a member?
```sql
select
customer_id,
count(product_id) as num_of_items,
sum(m.price) as total_amount
from dannys_diner.sales s
join dannys_diner.menu m using(product_id)
left join  dannys_diner.members me using(customer_id)
where me.join_date > s.order_date
group by 1
order by 1
```
| customer_id | num_of_items | total_amount |
| ----------- | ------------ | ------------ |
| A           | 2            | 25           |
| B           | 3            | 40           |

---
### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
with multiplier as (
    select
    s.customer_id,
    m.product_name,
    m.price,
    case when product_id = 1 then m.price*20 else m.price*10
    end as points
    from dannys_diner.menu m
    join dannys_diner.sales s using(product_id)
)
select
customer_id,
sum(points) as total_price
from multiplier
group by 1
order by 2 desc
```
| customer_id | total_price |
| ----------- | ----------- |
| B           | 940         |
| A           | 860         |
| C           | 360         |


---
### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
with earnings as (
    select
    customer_id,
    s.product_id,
    s.order_date,
    m.join_date
    from dannys_diner.sales s
    join dannys_diner.members m using(customer_id)
    where m.join_date <= s.order_date
)
select
customer_id,
sum(total_point) as total_point
from (
    select
    customer_id,
    product_id,
    sum(case when order_date < join_date + interval '7 day' then price * 20 else price*10 end) as total_point
    from earnings ea
    join dannys_diner.menu me using(product_id)
    where extract(month from order_date) = 01
    group by 1,2
) as stop
group by 1
```

| customer_id | total_point |
| ----------- | ----------- |
| A           | 1020        |
| B           | 320         |

---
## Bonus Questions
### Joining All The Things
```sql
-- The members column  with value N stands for NO and Y as YES.
select customer_id,
order_date,
product_name,
price,
case when join_date <= order_date then 'Y'else 'N' end as member
from dannys_diner.sales s
join dannys_diner.menu m using(product_id)
left join  dannys_diner.members me using(customer_id)
order by 1,2
```

| customer_id | order_date               | product_name | price | member |
| ----------- | ------------------------ | ------------ | ----- | ------ |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |

---
### Rank All The Things
```sql
with users_details as (
    select *,
    case when join_date <= order_date then 'Y' else 'N' end as member
    from dannys_diner.sales s
    join dannys_diner.menu m using(product_id)
    left join  dannys_diner.members me using(customer_id)
)
select customer_id,
order_date,
product_name,
price,
member,
case when member = 'Y' then (rank() over(partition by customer_id,member
                       order by order_date)) else NULL end as ranking
from users_details
```

| customer_id | order_date               | product_name | price | member | ranking |
| ----------- | ------------------------ | ------------ | ----- | ------ | ------- |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |         |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |         |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      | 1       |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      | 2       |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |         |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |         |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |         |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      | 1       |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      | 2       |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |         |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |         |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |         |

---