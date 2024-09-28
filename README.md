# 8 Week SQL Challenges by Danny | Case Study #1 - Danny's Diner
### ERD for Database
![image](https://github.com/user-attachments/assets/fd3e1711-dddc-407c-aa7a-893a56459b14)
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
![image](https://github.com/user-attachments/assets/932a87a5-582e-4f80-b2cd-226eaeb9f13a)

