# -Bike-Store-SQL-Analysis-Project
-This project is a SQL analysis of a retail bike store dataset ,designed to stimulate realworld business scenarios faced by data analysis .The goal of this project is not to write SQL queries but to undrstand and translate business questions into finding insights that supports decesions across departments 

-Using SQL  i have explored 25 key business Questions 

-This project highlight how SQL can be a powerful tools to transform raw data into actionable insights that drive business decesions.

## Objective
Analyze sales,customer and products

Answer real stakeholder questions

Build strong SQL problems-salving skills



-REVENUE CONCENTRATION
--The executive team believes that a small number of products generate most of the revenue.They want to identify which products contribute to the top 80% of total company revenue.

```
WITH product_revenue AS(
SELECT
p.product_id,
p.product_name,
SUM(oi.quantity * oi.list_price *(1 - oi.discount)) AS revenue
FROM order_items oi 
INNER JOIN products p
ON oi.product_id =p.product_id
GROUP BY p.product_id,p.product_name
),
revenue_calc AS(
SELECT
product_id,
product_name,
revenue,
SUM(revenue) OVER (ORDER BY revenue DESC) AS cummulative_revenue,
SUM(revenue) OVER () AS total_revenue
FROM product_revenue
)
SELECT
product_id,
product_name,
revenue,
cummulative_revenue,
(cummulative_revenue / total_revenue)*100 AS cummulative_percentage

FROM revenue_calc
WHERE (cummulative_revenue / total_revenue) <= 0.8
ORDER BY revenue DESC

```

STORE PERFORMANCE COMPARISONA
--Management wants to understand which store performs best. They want a comparison of total revenue, total orders, and average order value for each store.

```

SELECT
s.store_id,
s.store_name,
SUM(oi.quantity * oi.list_price *(1 - oi.discount)) AS total_revenue,
COUNT(DISTINCT o.order_id) AS total_orders,
SUM(oi.quantity * oi.list_price *(1 - oi.discount))/COUNT(DISTINCT o.order_id) AS avg_order_value
FROM stores s
INNER JOIN orders o
  ON s.store_id = o.store_id
INNER JOIN order_items oi
   ON o.order_id = oi.order_id 
GROUP BY s.store_id,s.store_name
ORDER BY total_revenue DESC

```

 MONTHLY FINANCE TREND
The finance department wants to see how revenue changes month-to-month 
so they can detect growth patterns or slow seasons.

```

WITH monthly_revenue AS (
SELECT
DATETRUNC(month,o.order_date) AS month,
SUM(oi.quantity * oi.list_price *(1 - oi.discount)) AS revenue
FROM orders o
INNER JOIN order_items oi
ON o.order_id = oi.order_id
GROUP BY DATETRUNC(month,o.order_date)
)
SELECT
month,
revenue,
LAG(revenue) OVER (ORDER BY month) AS previous_month_revenue,
revenue - LAG(revenue) OVER (ORDER BY month) AS revenue_change,

((revenue - LAG(revenue) OVER (ORDER BY month)) /LAG(revenue) OVER (ORDER BY month)) * 100 AS percent_change
FROM monthly_revenue
ORDER BY month

```

Top Customers Analysis

--Marketing wants to identify the top 10 customers who generate the highest revenue so they can target them for loyalty programs.

```

SELECT TOP 10
c.customer_id,
c.first_name,
c.last_name,
SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS total_revenue,
COUNT(DISTINCT o.order_id) AS total_orders
FROM customers c
INNER JOIN orders o
ON c.customer_id = o.customer_id
INNER JOIN order_items oi
ON o.order_id = oi.order_id
GROUP BY 
c.customer_id,
c.first_name,
c.last_name
ORDER BY total_revenue DESC

```

Customer Retention

--The customer success team wants to know how many customers make repeat purchases
versus those who only buy once.

```

WITH customer_orders AS(
SELECT 
customer_id,
COUNT(order_id) AS total_orders
FROM orders
GROUP BY customer_id
)

SELECT
CASE
	WHEN total_orders = 1 THEN 'One-time-customer'
	ELSE 'Repeat customer'
END AS customer_type,
COUNT(customer_id) AS number_of_customers
FROM customer_orders
GROUP BY 
	CASE
	WHEN total_orders = 1 THEN 'One-time-customer'
	ELSE 'Repeat customer'
END

```


Product Category Performance

--The product team wants to understand which product categories generate the highest revenue and unit sales.

```
SELECT 
    c.category_id,
    c.category_name,

  SUM(oi.quantity) AS total_units_sold,

  SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS total_revenue

FROM categories c

INNER JOIN products p
    ON c.category_id = p.category_id

INNER JOIN order_items oi
    ON p.product_id = oi.product_id

GROUP BY 
    c.category_id,
    c.category_name

ORDER BY total_revenue DESC

```

 Discount Impact
--Finance suspects that heavy discounting may be reducing profitability. They want to know which products receive the highest average discounts.

```

SELECT 
p.product_id,
p.product_name,
AVG(oi.discount) * 100 AS avg_discount_percentage,
COUNT(oi.order_id) AS number_of_times_discounted
FROM products p
INNER JOIN order_items oi
 ON p.product_id = oi.product_id
GROUP BY 
p.product_id,
p.product_name
ORDER BY avg_discount_percentage DESC

```

Best Performing Staff

--The regional manager wants to evaluate staff performance by identifying which employees generate the highest sales revenue.

```

 SELECT 
    s.staff_id,
    s.first_name,
    s.last_name,
    
  SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS total_sales_revenue,
 COUNT(DISTINCT o.order_id) AS total_orders_handled

FROM staffs s
INNER JOIN orders o
    ON s.staff_id = o.staff_id
INNER JOIN order_items oi
    ON o.order_id = oi.order_id
GROUP BY 
s.staff_id,
s.first_name,
s.last_name
ORDER BY total_sales_revenue DESC

```

Underperforming Products

--The merchandising team wants to identify products that have high stock levels but very low sales so they can consider promotions or discontinuation.

```
WITH sales AS (
SELECT 
 product_id,
 SUM(quantity) AS total_units_sold
 FROM order_items
 GROUP BY product_id
),
inventory AS (
 SELECT 
product_id,
SUM(quantity) AS total_stock
FROM stocks
GROUP BY product_id
)
SELECT 
p.product_id,
p.product_name,
i.total_stock,
COALESCE(s.total_units_sold, 0) AS total_units_sold
FROM products p
JOIN inventory i
    ON p.product_id = i.product_id
LEFT JOIN sales s
    ON p.product_id = s.product_id
WHERE i.total_stock > 0
AND COALESCE(s.total_units_sold, 0) < i.total_stock * 0.1
ORDER BY i.total_stock DESC

```

Store Inventory Risk

Operations wants to identify products that are close to running out of stock in each store.

```

SELECT 
s.store_id,
s.store_name,
p.product_id,
p.product_name,
 st.quantity AS stock_quantity
FROM dbo.stocks st
INNER JOIN stores s
    ON st.store_id = s.store_id
INNER JOIN products p
    ON st.product_id = p.product_id
WHERE st.quantity <= 5
ORDER BY 
s.store_name,
st.quantity ASC

```
 Seasonal Sales Patterns

--Management wants to know which months consistently generate the highest sales volume.
```
SELECT 
 MONTH (o.order_date)AS month,
SUM(oi.quantity) AS total_units_sold,
COUNT(DISTINCT o.order_id) total_orders
FROM orders o
INNER JOIN order_items oi
ON o.order_id = oi.order_id
GROUP BY MONTH (o.order_date)
ORDER BY total_units_sold DESC
```

Average Customer Spending

--Finance wants to calculate the average amount spent per customer across all orders.
```
SELECT
customer_id,
ROUND(AVG(total_customer_spending),2) AS avg_customer_spending
FROM(
	SELECT
	c.customer_id,
	SUM(oi.quantity*oi.list_price*(1-oi.discount)) total_customer_spending
	FROM customers c 
	INNER JOIN orders o
	ON c.customer_id = o.customer_id
	INNER JOIN order_items oi
	ON o.order_id = oi.order_id
	GROUP BY c.customer_id
 
)customer_spending
GROUP BY customer_id
```

 Geographic Sales Distribution

--The strategy team wants to understand which states generate the most revenue and which regions underperform.
```
SELECT
c.state,
SUM(oi.quantity*oi.list_price*(1-discount)) total_revenue,
COUNT(DISTINCT o.order_id) total_orders,
SUM(oi.quantity) total_units_sold
FROM customers c
INNER JOIN orders o
ON c.customer_id =o.customer_id
INNER JOIN order_items oi
ON o.order_id = oi.order_id
GROUP BY c.state
ORDER BY total_revenue

```

 Staff Productivity

--Operations wants to measure average order value handled by each staff member to evaluate sales efficiency.

```
SELECT
s.staff_id,
s.first_name + ' '+ s.last_name AS staff_name,
SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS total_revenue,
    
  SUM(oi.quantity * oi.list_price * (1 - oi.discount)) 
    / COUNT(DISTINCT o.order_id) AS average_order_value,

COUNT(DISTINCT o.order_id) AS total_orders
FROM staffs s
INNER JOIN orders o
ON s.staff_id =o.staff_id
INNER JOIN order_items oi
ON o.order_id = oi.order_id
GROUP BY 
    s.staff_id,
    s.first_name,
    s.last_name

ORDER BY average_order_value DESC

```
 Category Growth Trends
--The product team wants to analyze year-over-year revenue growth for each product category.

```
WITH category_yearly_revenue AS
(
    SELECT 
        c.category_id,
        c.category_name,
 YEAR(o.order_date) AS order_year,
 SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS total_revenue
FROM order_items oi
INNER JOIN products p 
ON oi.product_id = p.product_id
INNER JOIN categories c 
ON p.category_id = c.category_id
INNER JOIN orders o 
ON oi.order_id = o.order_id
GROUP BY 
        c.category_id,
        c.category_name,
        YEAR(o.order_date)
SELECT 
    category_name,
    order_year,
    total_revenue,
LAG(total_revenue) OVER (PARTITION BY category_name ORDER BY order_year) AS previous_year_revenue,
total_revenue - LAG(total_revenue) OVER (PARTITION BY category_name ORDER BY order_year) AS revenue_change,
CASE 
        WHEN LAG(total_revenue) OVER 
            (PARTITION BY category_name ORDER BY order_year) IS NULL 
        THEN NULL
        
  ELSE 
            (total_revenue - LAG(total_revenue) OVER 
                (PARTITION BY category_name ORDER BY order_year)) * 100.0
            / LAG(total_revenue) OVER 
                (PARTITION BY category_name ORDER BY order_year)
END AS yoy_growth_percentage
FROM category_yearly_revenue
ORDER BY category_name, order_year

```
Store Contribution to Total Revenue
--Executives want to know what percentage of total company revenue each store contributes.
```

WITH store_revenue AS
(
SELECT 
 s.store_id,
 s.store_name,
 SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS total_revenue
FROM stores s
INNER JOIN orders o
ON s.store_id = o.store_id
INNER JOIN order_items oi
ON o.order_id = oi.order_id
GROUP BY 
        s.store_id,
        s.store_name)
SELECT 
    store_id,
    store_name,
    total_revenue,

 SUM(total_revenue) OVER () AS company_total_revenue,
(total_revenue * 100.0) / SUM(total_revenue) OVER () AS revenue_percentage
FROM store_revenue
ORDER BY revenue_percentage DESC
```

Top Product per Store
--Store managers want to know which product generates the highest revenue in their store.

```
SELECT 
b.brand_id,
b.brand_name,
SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS total_revenue,
SUM(oi.quantity) AS total_units_sold,
COUNT(DISTINCT o.order_id) AS total_orders

FROM brands b
INNER JOIN products p
ON b.brand_id = p.brand_id
INNER JOIN order_items oi
ON p.product_id = oi.product_id
INNER JOIN orders o
ON oi.order_id = o.order_id
GROUP BY 
b.brand_id,
b.brand_name
ORDER BY total_revenue DESC

```

Customer Purchase Frequency Marketing wants to segment customers based on how frequently they place orders.

```

SELECT 
c.customer_id,
c.first_name + ' ' + c.last_name AS customer_name,
    
COUNT(o.order_id) AS total_orders,
CASE 
        WHEN COUNT(o.order_id) = 1 THEN 'One-time Buyer'
        WHEN COUNT(o.order_id) BETWEEN 2 AND 5 THEN 'Occasional Buyer'
        ELSE 'Frequent Buyer'
    END AS customer_segment
FROM customers c
LEFT JOIN orders o
ON c.customer_id = o.customer_id
GROUP BY 
c.customer_id,
c.first_name,
c.last_name
ORDER BY total_orders DESC
```

Revenue per Brand The product strategy team wants to analyze 
--which brands generate the highest revenue across all stores.

```

SELECT 
b.brand_name,
SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS total_revenue
FROM brands b
INNER JOIN products p
ON b.brand_id = p.brand_id
INNER JOIN order_items oi
ON p.product_id = oi.product_id
GROUP BY b.brand_name
ORDER BY total_revenue DESC

```

Order Fulfillment Speed Operations suspects delays in order processing and wants to analyze the average time between order date and shipped date.

```

SELECT 
store_id,
AVG(DATEDIFF(DAY, order_date, shipped_date)) AS avg_days
FROM orders
WHERE shipped_date IS NOT NULL
GROUP BY store_id
```
 High-Value Orders

--Finance wants to identify orders that exceed the average order value by at least 50%.
```
WITH order_values AS
(SELECT
o.order_id,
SUM(oi.quantity*oi.list_price *(1-oi.discount)) AS order_value
FROM orders o
INNER JOIN order_items oi
ON o.order_id = oi.order_id
GROUP BY o.order_id
),
avg_value AS 
(SELECT
AVG(order_value) AS avg_order_value
FROM order_values)
SELECT
ov.order_id,
ov.order_value,
av.avg_order_value
FROM order_values ov
CROSS JOIN avg_value av
 WHERE ov.order_value > = av.avg_order_value
 ORDER BY ov.order_value DESC

```


--Store Sales Consistency

Executives want to find stores that show consistent monthly growth in revenue for at least three consecutive months. 
```
WITH monthly_revenue AS
(
    SELECT 
        s.store_id,
        s.store_name,
        YEAR(o.order_date) AS order_year,
        MONTH(o.order_date) AS order_month,
        SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS revenue

FROM stores s
JOIN orders o
ON s.store_id = o.store_id
JOIN order_items oi
ON o.order_id = oi.order_id
GROUP BY 
        s.store_id,
        s.store_name,
        YEAR(o.order_date),
        MONTH(o.order_date)
),
revenue_with_lag AS
(
    SELECT 
        *,
        LAG(revenue) OVER (PARTITION BY store_id ORDER BY order_year, order_month
        ) AS prev_revenue
FROM monthly_revenue
),
growth_flag AS
(
    SELECT 
        *,
       CASE 
            WHEN revenue > prev_revenue THEN 1
            ELSE 0
        END AS is_growth
FROM revenue_with_lag
),
streaks AS
(
    SELECT 
        *,
        
   SUM(CASE WHEN is_growth = 0 THEN 1 ELSE 0 END)
       OVER (PARTITION BY store_id ORDER BY order_year, order_month) AS grp

  FROM growth_flag
)
SELECT 
    store_id,
    store_name,
    COUNT(*) AS consecutive_growth_months
FROM streaks
WHERE is_growth = 1
GROUP BY 
    store_id,
    store_name,
    grp
	HAVING COUNT(*) >= 3
ORDER BY consecutive_growth_months DESC

```


 Product Diversity per Store

--Management wants to know which store sells the widest variety of products.
```
SELECT 
    s.store_id,
    s.store_name,
    
  COUNT(DISTINCT oi.product_id) AS unique_products_sold

FROM stores s

JOIN orders o
    ON s.store_id = o.store_id

JOIN order_items oi
    ON o.order_id = oi.order_id

GROUP BY 
    s.store_id,
    s.store_name

ORDER BY unique_products_sold DESC

```

Customer Lifetime Value

--Marketing wants to estimate total revenue generated by each customer across all their purchases.
```
SELECT 
    c.customer_id,
    c.first_name + ' ' + c.last_name AS customer_name,
    
  SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS customer_lifetime_value,
    
  COUNT(DISTINCT o.order_id) AS total_orders,
    
  SUM(oi.quantity) AS total_units_purchased

FROM customers c

JOIN orders o
    ON c.customer_id = o.customer_id

JOIN order_items oi
    ON o.order_id = oi.order_id

GROUP BY 
    c.customer_id,
    c.first_name,
    c.last_name

ORDER BY customer_lifetime_value DESC

```

Slow Moving Inventory

--The inventory team wants to detect products that have remained in stock for a long time but generate very few sales.

```
WITH product_sales AS
(
    SELECT 
        oi.product_id,
        SUM(oi.quantity) AS total_units_sold
    FROM order_items oi
    GROUP BY oi.product_id
),

product_stock AS
(
    SELECT 
        p.product_id,
        p.product_name,
        SUM(s.quantity) AS total_stock
    FROM products p
    JOIN stocks s
        ON p.product_id = s.product_id
    GROUP BY 
        p.product_id,
        p.product_name
)

SELECT 
    ps.product_id,
    ps.product_name,
    ps.total_stock,
    
  ISNULL(sales.total_units_sold, 0) AS total_units_sold,
CASE 
        WHEN ISNULL(sales.total_units_sold, 0) = 0 THEN 'No Sales'
        WHEN ps.total_stock > sales.total_units_sold THEN 'Slow Moving'
        ELSE 'Normal'
    END AS inventory_status

FROM product_stock ps

LEFT JOIN product_sales sales
    ON ps.product_id = sales.product_id

WHERE 
    ps.total_stock > ISNULL(sales.total_units_sold, 0)

ORDER BY ps.total_stock DESC

```

## 📌 Recommendations

Based on the analysis conducted on the bike store dataset, the following strategic recommendations are proposed to improve business performance and decision-making:

1. Focus on High-Performing Products

A small number of products contribute a large percentage of total revenue.


Increase inventory levels for top-selling products
Prioritize these products in marketing and promotions
Bundle high-performing products with low-performing ones to boost sales

 2. Strengthen Customer Retention Strategies

A small segment of customers generates a significant portion of revenue.


Introduce loyalty and reward programs
Offer personalized promotions based on purchase behavior
Target one-time buyers with re-engagement campaigns

3. Optimize Store Performance

Performance varies across stores, with some consistently outperforming others.


Identify and replicate strategies used by top-performing stores
Provide training and support to underperforming stores
Allocate resources based on store-level performance

 4. Improve Inventory Management

Some products remain in stock for long periods with low sales.


Apply discounts or promotions to clear slow-moving inventory
Reduce procurement of low-demand products
Implement demand forecasting to optimize stock levels

5. Enhance Order Fulfillment Efficiency

Delays in order processing can negatively affect customer satisfaction

Streamline order processing and logistics workflows
Set internal benchmarks for fulfillment time
Continuously monitor and improve delivery performance

 6. Prioritize High-Performing Brands and Categories

Certain brands and categories contribute more to overall revenue.


Strengthen partnerships with top-performing brands
Increase visibility and promotion of high-performing categories
Reassess low-performing brands and adjust strategy accordingly
 7. Leverage Data for Strategic Decision-Making

Data analysis provides valuable insights for improving business outcomes.


Develop dashboards for real-time performance tracking
Monitor key metrics regularly
Promote a data-driven decision-making culture
