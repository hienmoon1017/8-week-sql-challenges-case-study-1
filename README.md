# 8 Week SQL Challenges by Danny | Case Study #1 - Danny's Diner
### ERD for Database
![image](https://github.com/user-attachments/assets/85c19d63-88a9-46d2-a6ab-6b09d9a66885)

## Result of Questions
### Q1. What is the total amount each customer spent at the restaurant?
```sql
SELECT distinct s.customer_id
,sum(m.price) as "total spent"
FROM danny_dinner.sales s
JOIN danny_dinner.menu m on s.product_id = m.product_id
GROUP BY 1
ORDER BY 1
;
```
_Result:_

![image](https://github.com/user-attachments/assets/932a87a5-582e-4f80-b2cd-226eaeb9f13a)

### Q2. How many days has each customer visited the restaurant?
```sql
SELECT customer_id
,count(order_date) as "total visited day"
FROM danny_dinner.sales
GROUP BY customer_id
ORDER BY 1
;
```
_Result:_

![image](https://github.com/user-attachments/assets/ad2ee758-b64a-4528-b096-01b73b72b2f2)

### Q3. What was the first item from the menu purchased by each customer?
```sql
-- show product_name
WITH first_order_date as
(
	SELECT customer_id, min(order_date) as first_date
	FROM danny_dinner.sales
	GROUP BY 1
),
	first_purchase as
	(
		SELECT s.customer_id, s.order_date, s.product_id
		FROM danny_dinner.sales s
		JOIN first_order_date fod on s.customer_id = fod.customer_id
			and s.order_date = fod.first_date
	)
SELECT distinct fp.customer_id, fp.order_date, m.product_name
FROM first_purchase fp
JOIN danny_dinner.menu m on fp.product_id = m.product_id
ORDER BY 1
;
-- show product_id
WITH first_order_date as
(
	SELECT customer_id, min(order_date) as first_date
	FROM danny_dinner.sales
	GROUP BY 1	
)
SELECT distinct s.customer_id, fod.first_date, s.product_id
FROM danny_dinner.sales s
JOIN first_order_date fod on fod.customer_id = s.customer_id
	and s.order_date = fod.first_date 
ORDER BY 1
;
```
_Result:_

![image](https://github.com/user-attachments/assets/ff6d262c-1892-44b8-98ba-137b2a944ba1)

### Q4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT product_id, count(customer_id) as purchased_time
FROM
	(
		SELECT distinct customer_id, product_id, order_date
		FROM danny_dinner.sales
	)
GROUP BY 1
ORDER BY 2 DESC
LIMIT 1
;
```
_Result:_

![image](https://github.com/user-attachments/assets/42b678dd-85b0-47b6-9296-01334b78d7cc)
### Q5. Which item was the most popular for each customer?
```sql
WITH most_popular_item as
(
	SELECT customer_id
	,product_id
	,count(*) as order_count
	,rank() over (partition by customer_id order by count(*) DESC) as ranking
	FROM 
	(
		SELECT distinct customer_id, product_id, order_date
		FROM danny_dinner.sales
	)	
	GROUP BY 1,2	
)
SELECT customer_id
		,product_id
		,order_count
		,ranking
FROM most_popular_item
WHERE ranking = 1
ORDER BY 1,2
;
```
_Result:_

![image](https://github.com/user-attachments/assets/17b8d1c2-4edd-4ab7-aef6-17edf0c6b96f)

### Q6. Which item was purchased first by the customer after they became a member?
```sql
WITH first_purchase_after_member as
(
	SELECT s.customer_id
	,s.product_id
	,s.order_date
	,m.join_date
	FROM 
	(
		SELECT distinct customer_id, product_id, order_date
		FROM danny_dinner.sales 
	) as s
	JOIN danny_dinner.members m on s.customer_id = m.customer_id
	WHERE s.order_date >= m.join_date	
),
ranked_item as
(
	SELECT customer_id
	,product_id	
	,order_date
	,join_date
	,rank() over (partition by customer_id order by order_date ASC) as ranking
	FROM first_purchase_after_member
)
SELECT ri.customer_id
,ri.product_id
,m.product_name
,ri.order_date
,ri.join_date
,ri.ranking
FROM ranked_item ri
JOIN danny_dinner.menu m on ri.product_id = m.product_id
WHERE ranking=1
ORDER BY 1
;
```
_Result:_

![image](https://github.com/user-attachments/assets/e89f9640-d35e-406e-ab3d-de9494adafea)

### Q7. Which item was purchased just before the customer became a member?
```sql
WITH purchased_item_before_member as
(
	SELECT s_dis.customer_id, s_dis.product_id, s_dis.order_date, m.join_date
	FROM 
	(
		SELECT distinct customer_id, product_id, order_date
		FROM danny_dinner.sales
	) as s_dis
	LEFT JOIN danny_dinner.members m on m.customer_id = s_dis.customer_id
	WHERE s_dis.order_date < m.join_date or m.customer_id isnull
)
SELECT pibm.customer_id, m.product_name, pibm.order_date, pibm.join_date
FROM purchased_item_before_member as pibm
JOIN danny_dinner.menu m on m.product_id = pibm.product_id
ORDER BY 1,3
;
```
_Result:_

![image](https://github.com/user-attachments/assets/b823a095-33a2-4dcf-9adc-6e6bf638076b)

### Q8. What is the total items and amount spent for each member before they became a member?
```sql
WITH purchased_item_before_member as
(
	SELECT s_dis.customer_id, s_dis.product_id, s_dis.order_date, m.join_date
	FROM
	(
		SELECT distinct customer_id, product_id, order_date
		FROM danny_dinner.sales
	) as s_dis
	LEFT JOIN danny_dinner.members m on m.customer_id = s_dis.customer_id
	WHERE s_dis.order_date < m.join_date or m.customer_id is null
)
SELECT pibm.customer_id
,count(*) as total_items
,sum(m.price) as amount_spent
FROM purchased_item_before_member pibm
JOIN danny_dinner.menu m on m.product_id = pibm.product_id
GROUP BY 1
ORDER BY 1
;
```
_Result:_

![image](https://github.com/user-attachments/assets/e203d4f9-f595-4d1e-923f-711ef9ac0481)

### Q9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
-- show only total_points per customer
WITH points as
(
	SELECT s_dis.customer_id
	,s_dis.product_id
	,case when s_dis.product_id = 1 then m.price * 20 -- product_id of sushi is 1
	  	  else m.price * 10
	  	  end as points
	FROM
	(
		SELECT distinct customer_id, product_id, order_date
		FROM danny_dinner.sales
	) as s_dis
	JOIN danny_dinner.menu m on m.product_id = s_dis.product_id
)
SELECT customer_id
,sum(points) as total_points
FROM points
GROUP BY 1
ORDER BY 1
;

-- show total_points per product and customer
WITH points as
(
	SELECT s_dis.customer_id
	,s_dis.product_id
	,case when s_dis.product_id = 1 then m.price * 20 -- product_id of sushi is 1
	  	  else m.price * 10
	  	  end as points
	FROM
	(
		SELECT distinct customer_id, product_id, order_date
		FROM danny_dinner.sales
	) as s_dis
	JOIN danny_dinner.menu m on m.product_id = s_dis.product_id
)
SELECT customer_id
,case when product_id = 1 then 'Sushi'
	  else 'Other'
	  end as item
,sum(points) as total_points
FROM points
GROUP BY 1,2 
ORDER BY 1,2 DESC
;
```
_Result:_

![image](https://github.com/user-attachments/assets/18c97196-7c36-478c-9057-5d7b0dc0c8ce)

### Q10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
WITH points_January as
(
	SELECT s_dis.customer_id
	,s_dis.order_date
	,m.join_date
	,count(*) as item_count
	,case when s_dis.order_date between m.join_date and m.join_date + interval '6 day' then count(*) * 2
		else count(*)
		end as points_jan
	FROM
	(
		SELECT distinct customer_id, product_id, order_date
		FROM danny_dinner.sales
	) as s_dis
	JOIN danny_dinner.members m on m.customer_id = s_dis.customer_id	
	WHERE s_dis.order_date >= m.join_date and extract(month from s_dis.order_date) = 01	
	GROUP BY 1,2,3
)
SELECT customer_id
, case when order_date between join_date and join_date + interval '6 day' then 'first_week'
	   else 'other_of_jan'
	   end as week
,sum(item_count) as total_item
,sum(points_jan) as total_points
FROM points_January
GROUP BY 1,2
ORDER BY 1
;
```
_Result:_

![image](https://github.com/user-attachments/assets/1e3b1407-c70d-4e25-a998-450695c74658)

### Q11. Create new table includes customer_id, order_date, product_name, price, member (Yes, No), and ranking_member with null values for non-member purchases
```sql
WITH ranking as
(
	SELECT s_dis.customer_id
	,s_dis.product_id
	,s_dis.order_date
	,case when s_dis.order_date >= m.join_date or s_dis.customer_id isnull then 'Yes'
		  else 'No'
		  end as member	
	FROM
	(
		SELECT distinct customer_id, product_id, order_date
		FROM danny_dinner.sales
	) as s_dis
	LEFT JOIN danny_dinner.members m on m.customer_id = s_dis.customer_id
)
SELECT r.customer_id
,r.order_date
,m.product_name
,m.price
,r.member
,case when r.member = 'Yes' then rank() over (partition by r.customer_id, r.member order by r.order_date ASC) 
	  else null	  		
	  end as ranking_member
FROM ranking r
JOIN danny_dinner.menu m on m.product_id = r.product_id
ORDER BY 1,2
;
```
_Result:_

![image](https://github.com/user-attachments/assets/a5b9dfa3-653a-4eb1-8e3b-77bfba1818eb)

Thank you for stopping by, and I'm pleased to connect with you, my new friend!

Please do not forget to FOLLOW and star ⭐ the repository if you find it valuable.

I wish you a day filled with happiness and energy!

Warm regards,

Hien Moon
