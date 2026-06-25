---
layout: page
title: ""
subtitle: "How Data Modeling With SQL Answers Real Business Questions"
cover-img: 
thumbnail-img: 
share-img: 
tags: [Market Analysis, Data Analysis, SQL]
---

<h4>Relational Data Modeling and Business Analytics With PostgreSQL</h4>

<p align='justify'>
Imagine a company with thousands of customers, sales reps across multiple regions, and millions of dollars in orders. How do you figure out which region makes the most money, or which sales rep closes the biggest deals? You ask the database. I used PostgreSQL to query a real sales dataset and answer exactly those kinds of business questions — clearly and directly.
</p>

<p align='justify'>
The database has five tables including:
<ol>
<li>web_events</li>
<li>orders</li>
<li>accounts</li>
<li>sales_reps</li>
<li>region</li>
</ol>
</p>

<p align='justify'>
The Entity Relationship Diagram (ERD) for tables is as below:
</p>
<p align="center">
<img src="/assets/portfolio/ERD.png" width="700">
</p>

Through this section, I shared some of my efforts to answer analytics questions using PostgreSQL codes.  
<br>

**Analytical Question:** For the region with the largest sales, how many total orders were placed?

```
WITH t1 AS (
   SELECT r.name reg_name, SUM(o.total_amt_usd) total_usd    
   FROM sales_reps s  
   JOIN region r 
   ON r.id = s.region_id
   JOIN accounts a 
   ON s.id = a.sales_rep_id
   JOIN orders o 
   ON a.id = o.account_id 
   GROUP BY reg_name
   ORDER BY total_usd DESC
   LIMIT 1),

t2 AS (
   SELECT r.name reg_name, COUNT(o.total) total_order    
   FROM sales_reps s  
   JOIN region r 
   ON r.id = s.region_id
   JOIN accounts a 
   ON s.id = a.sales_rep_id
   JOIN orders o 
   ON a.id = o.account_id 
   GROUP BY reg_name)

SELECT t1.reg_name, t1.total_usd, t2.total_order
FROM t1
JOIN t2
ON t1.reg_name = t2.reg_name;
```

**Answer:** <br>
Region name: Northeast <br>
Total sales amount (usd): 7744405.36  <br> 
Total order: 2357
