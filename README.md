# E-commerce Analysis Using MySQL

## Table of Contents
 - [Project Overview](#project-overview)
 - [Data Analysis](#data-analysis)
 - [Results and Findings](#results-and-findings)
 - [Recommendations](#recommendations)
 - [Limitations](#limitations)
 - [References](#references)

### Project Overview
This project revolves around optimizing and analyzing a toy store's operations answering key business questions, ranging from inventory management to customer behavior insights. The task involves crafting SQL queries to address 25 specific questions, covering areas like stock levels (e.g., identifying zero stock or overstocked products), product performance (e.g., top-selling categories), loyal customer analysis (e.g., inactive customers), sales and revenue trends (e.g., month-over-month growth), customer acquisition and retention (e.g., new customers per month), and detailed customer segmentation (e.g., by gender, age, or spending). These queries help solve practical business problems, such as preventing stockouts, targeting high-value customers, and tailoring marketing strategies

### Data Sources 
The database consists of three main tables: dim_customers, which stores customer details like customer_id, first_name, last_name, email, country, join_date, gender, marital_status, and birth_date; fact_sales, which tracks sales data with sale_id, customer_id, product_id, order_date, quantity, and sales_amount; and dim_products, which holds product information including product_id, product_name, category, price, and stock. These dimensions tables (customers and products) are joined to the fact sales table using customer_id and product_id.

### Tools used for this project

- Excel : Data Cleaning
- MySQL : Data Analysis
- Power BI : Creating Reports

### Data Cleaning/Preparation
  In the initial data preparation phase, we performed the following task:
   1. Data loading and inspection to correct data errors.
   2. Handling missing values.
   3. Standardizing data formats.
   4. Removing duplicates.
   5. Validating data integrity.

### Exploratory Data Analysis (EDA)
 - Which products are out of stock or overstocked?
 - What is the sales trend overall?
 - Which product categories are top seller?
 - Which customers are loyal?
 - What are the peak sales period?

### Data Analysis
```sql 
CREATE DATABASE bens;

CREATE TABLE dim_customers (
    customer_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    gender VARCHAR(20),
    marital_status VARCHAR(25),
    email VARCHAR(100),
    country VARCHAR(50),
    join_date DATE,
    birth_date DATE
);

CREATE TABLE dim_products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(10,2),
    cost DECIMAL(10,2),
    stock INT
);

CREATE TABLE fact_sales (
    sale_id INT PRIMARY KEY,
    customer_id INT,
    product_id INT,
    order_date DATE,
    quantity INT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (product_id) REFERENCES product(product_id)
);

      QUERY OPTIMIZATION THROUGH INDEXES
                         
-- Customer_id is already indexed as PRIMARY KEY
CREATE INDEX idx_customers_country ON bens.dim_customers (country);
CREATE INDEX idx_customers_join_date ON bens.dim_customers (join_date);
CREATE INDEX idx_customers_birth_date ON bens.dim_customers (birth_date);
CREATE INDEX idx_customers_gender ON bens.dim_customers (gender);
CREATE INDEX idx_customers_marital_status ON bens.dim_customers (marital_status);

-- Product_id is already indexed as PRIMARY KEY
CREATE INDEX idx_product_category ON bens.dim_products (category);
CREATE INDEX idx_product_stock ON bens.dim_products (stock);

-- Sale_id is already indexed as PRIMARY KEY
CREATE INDEX idx_fact_sales_customer_id ON bens.fact_sales (customer_id);
CREATE INDEX idx_fact_sales_product_id ON bens.fact_sales (product_id);
CREATE INDEX idx_fact_sales_order_date ON bens.fact_sales (order_date);
CREATE INDEX idx_fact_sales_customer_date ON bens.fact_sales (customer_id, order_date);

    
A-STOCK INVENTORY MANAGEMENT: Focuses on managing and analyzing inventory levels to optimize stock and prevent shortages or overstocking.

-- 1. Which products have zero stock / are at risk of stockout /normal stock / are overstocked ?
-- Helps identify products needing restocking to avoid lost sales and optimize inventory levels.
-- Enables proactive restocking to prevent stockouts and maintain customer satisfaction.
-- Identifies excess inventory to reduce holding costs and free up storage space.


SELECT 
    product_id,
    product_name,
    category,
    stock,
    CASE
        WHEN stock = 0 THEN 'Out_Of_Stock'
        WHEN stock BETWEEN 0 AND 200 THEN 'At Risk'
        WHEN stock BETWEEN 200 AND 500 THEN 'Normal'
        WHEN stock > 500 THEN 'Overstocked'
        ELSE 'Ouffff'
    END AS Stock_Inventory_Segmentation
FROM
    bens.dim_products
; 
    
-- 2. What is the stock value of each category?
-- Provides insight into the financial value of inventory per category for better budgeting and investment decisions.


SELECT 
    category, ROUND(SUM(price * stock), 2) AS StockValue
FROM
    bens.dim_products 
GROUP BY category
ORDER BY StockValue DESC;

-- 3. Which products have the highest stock-to-sales ratio?
-- Highlights slow-moving products to adjust purchasing strategies and minimize overstock.
-- Identifies underperforming categories to improve marketing or phase out unprofitable items.


SELECT 
    p.product_id,
    p.category,
    p.product_name,
    p.stock,
    COALESCE(SUM(fs.quantity), 0) AS total_sold,
    ROUND(p.stock / NULLIF(SUM(fs.quantity), 0), 2) AS stock_sales_ratio
FROM
    bens.dim_products p
        LEFT JOIN
    bens.fact_sales fs ON p.product_id = fs.product_id
GROUP BY p.product_id , p.category, p.product_name , p.stock
HAVING total_sold > 0
ORDER BY stock_sales_ratio DESC
;

-- 4. Predict stock depletion based on average daily sales?
-- Allows planning for restocking to prevent shortages and ensure continuous product availability.


WITH Avg_Daily_Sales AS (
    SELECT 
        p.product_id,	
        p.product_name,
        p.stock,
        SUM(fs.quantity) / DATEDIFF(MAX(fs.order_date), MIN(fs.order_date)) AS Avg_Daily_Qty
    FROM 
        bens.fact_sales fs
        LEFT JOIN bens.dim_products p ON fs.product_id = p.product_id
    GROUP BY 
        p.product_id,
        p.product_name,
        p.stock
    HAVING 
        Avg_Daily_Qty > 0
)	
SELECT 
    product_id,
    product_name,
    stock,
    ROUND(stock / Avg_Daily_Qty, 0) AS Days_Until_Depletion
FROM 
    Avg_Daily_Sales
WHERE 
    stock > 0
ORDER BY 
    Days_Until_Depletion DESC;
    

 B- PRODUCT PERFORMANCE: Analyzes product sales, profitability, and market trends to guide product strategy.

-- 5. Which products are the top 5 sellers by quantity?
-- Highlights popular products to ensure adequate stock and promote high-demand items.

 
SELECT 
    p.product_id,
    p.product_name,
    SUM(fs.quantity) AS Total_Quantity
FROM
    bens.fact_sales fs
        LEFT JOIN
    bens.dim_products p ON fs.product_id = p.product_id
GROUP BY p.product_id , p.product_name
ORDER BY Total_Quantity DESC
LIMIT 5;

-- 6. What is the average order value by category?
-- Informs pricing and promotion strategies to boost profitability per category.
 

SELECT 
    p.category AS Product_Category,
    ROUND(AVG(p.price * fs.quantity), 2) AS Avg_Order_Value
FROM
    bens.fact_sales fs
        LEFT JOIN
    bens.dim_products p ON fs.product_id = p.product_id
GROUP BY Product_Category
ORDER BY Avg_Order_Value DESC;

-- 7. Which products contribute to 80% of total sales (Pareto analysis)?
-- Focuses efforts on the most impactful products to optimize sales and efficiency.


WITH Product_Sales AS (
    SELECT 
        p.product_id,
        p.product_name,
        SUM(p.price * fs.quantity) AS Sales_Amount,
        SUM(SUM(p.price * fs.quantity)) OVER () AS Total_Sales,
        SUM(SUM(p.price * fs.quantity)) OVER (ORDER BY SUM(p.price * fs.quantity) DESC) / 
            SUM(SUM(p.price * fs.quantity)) OVER () AS Cum_Share
    FROM 
        bens.fact_sales fs
        LEFT JOIN bens.dim_products p ON fs.product_id = p.product_id
    GROUP BY 
        p.product_id,
        p.product_name
)
SELECT 
    product_id,
    product_name,
    Sales_Amount
FROM 
    Product_Sales
WHERE 
    Cum_Share <= 0.8
ORDER BY 
    Sales_Amount DESC;

-- 8. What is the total sales, total cost, quantity sold, profit and profit margin by product category?
--  Helps assess category profitability to guide pricing and cost management decisions.


WITH Product_Sales AS (
    SELECT 
        p.category,
        SUM(p.price * fs.quantity) AS Sales_Amount,
        SUM(fs.quantity) AS Quantity_Sold,
        SUM(fs.quantity * p.cost) AS Cost_Amount
    FROM 
        bens.fact_sales fs
        LEFT JOIN bens.dim_products p ON fs.product_id = p.product_id
    GROUP BY 
       p.category
)
SELECT 
    category,
    Sales_Amount,
    Quantity_Sold,
    Cost_Amount,
    ROUND((Sales_Amount - Cost_Amount),2) AS Profit,
    ROUND((Sales_Amount - Cost_Amount) / Sales_Amount, 2) AS Profit_Margin
FROM 
    Product_Sales;
    
-- 9. Which products have the highest profit margin?
--  Prioritizes high-margin products to maximize profit and resource allocation.


WITH Product_Sales AS (
    SELECT 
        p.product_id,
        p.product_name,
        SUM(p.price * fs.quantity) AS Sales_Amount,
        SUM(fs.quantity * p.cost) AS Cost_Amount
    FROM 
        bens.fact_sales fs
        LEFT JOIN bens.dim_products p ON fs.product_id = p.product_id
    GROUP BY 
       p.product_id,
       p.product_name
)
SELECT 
    product_id,
	product_name,
    Sales_Amount,
    Cost_Amount,
    ROUND((Sales_Amount - Cost_Amount),2) AS Profit,
    ROUND((Sales_Amount - Cost_Amount) / Sales_Amount, 2) AS Profit_Margin
FROM 
    Product_Sales
ORDER BY 6 DESC;


      C- LOYAL CUSTOMER ANALYSIS : Focuses on identifying and analyzing loyal or high-value customers.

-- 10. Which customers have not made any purchases/ inactive customers?
--  Targets inactive customers with re-engagement campaigns to boost sales.


SELECT 
    *
FROM
    bens.dim_customers
WHERE
    customer_id NOT IN (SELECT 
            customer_id
        FROM
            bens.fact_sales);
            
		
-- 11. Which customers ordered the same product multiple times? (repeated purchases)
--  Identifies loyal customers for targeted retention strategies and personalized offers.


SELECT 
    c.customer_id,
    CONCAT(c.last_name, ' ', c.first_name) AS FullName,
    COUNT(DISTINCT CASE 
        WHEN fs.product_id = ANY (
            SELECT product_id 
            FROM bens.fact_sales fs2 
            WHERE fs2.customer_id = c.customer_id 
            GROUP BY product_id 
            HAVING COUNT(*) > 1
        ) THEN fs.product_id 
        ELSE NULL 
    END) AS Number_Of_Repeated_Products,
    SUM(fs.quantity * p.price) AS Total_Sales
FROM 
    bens.dim_customers c
    JOIN bens.fact_sales fs ON c.customer_id = fs.customer_id
    JOIN bens.dim_products p ON fs.product_id = p.product_id
GROUP BY 
    c.customer_id
HAVING 
    Number_Of_Repeated_Products > 1 
ORDER BY 
    Total_Sales DESC;

-- 12. Which customers are in the top 10% by total spending?
-- Focuses on top spenders to enhance customer retention and increase lifetime value.


WITH CustomerSpending AS (
    SELECT 
        c.customer_id,
        CONCAT(c.last_name, ' ', c.first_name) AS FullName,
        ROUND(SUM(fs.quantity * p.price), 2) AS Total_Spent,
        NTILE(10) OVER (ORDER BY ROUND(SUM(fs.quantity * p.price), 2) DESC) AS Spending_Decile
    FROM 
        bens.dim_customers c
        JOIN bens.fact_sales fs ON c.customer_id = fs.customer_id
        JOIN bens.dim_products p ON fs.product_id = p.product_id
    GROUP BY 
        c.customer_id,
        c.first_name,
        c.last_name
)
SELECT 
    customer_id,
    FullName,
    Total_Spent
FROM 
    CustomerSpending
WHERE 
    Spending_Decile = 1
ORDER BY 
    Total_Spent DESC;
    
-- 13. What is the average time between purchases for each customer? (purchases frequency)
-- Measures customer loyalty and informs re-engagement timing to maintain sales momentum.


WITH PurchaseIntervals AS (
    SELECT 
        customer_id,
        order_date,
        DATEDIFF(order_date, LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date)) AS Days_Between
    FROM 
        bens.fact_sales
    WHERE 
        order_date IS NOT NULL
)
SELECT 
    c.customer_id,
    CONCAT(c.last_name, ' ', c.first_name) AS FullName,
    ROUND(AVG(pi.Days_Between), 2) AS Avg_Days_Between_Purchases
FROM 
    bens.dim_customers c
    LEFT JOIN PurchaseIntervals pi ON c.customer_id = pi.customer_id
GROUP BY 
    c.customer_id,
    c.first_name,
    c.last_name
HAVING 
    COUNT(pi.Days_Between) > 0
ORDER BY 
    Avg_Days_Between_Purchases ASC;
    
--14.  What is the average sales amount per customer?   


SELECT c.customer_id, c.first_name, c.last_name, ROUND(AVG(fs.sales_amount), 2) AS avg_sales
FROM bens.dim_customers c
JOIN bens.fact_sales fs ON c.customer_id = fs.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name;

--15. Which countries have customers with no purchases in the last 90 days?
-- Identifies lapsed customers for reactivation campaigns to recover lost sales.


SELECT DISTINCT c.country
FROM customers c
LEFT JOIN fact_sales fs ON c.customer_id = fs.customer_id
WHERE fs.sale_id IS NULL OR fs.order_date < DATE_SUB(CURDATE(), INTERVAL 90 DAY);


          D- REVENUE ANALYSIS : Examines overall sales performance, trends, and growth.

-- 16. What is the total sales amount by country?
-- Reveals geographic sales performance to tailor marketing and expansion strategies.


SELECT 
    c.country, SUM(fs.quantity * p.price) AS Total_Sales
FROM
    bens.dim_customers c
        JOIN
    bens.fact_sales fs ON fs.customer_id = c.customer_id
        JOIN
    bens.dim_products p ON fs.product_id = p.product_id
GROUP BY c.country
ORDER BY 2 DESC;

-- 17. What is the total sales by month and the month-over-month sales growth rate?
--  Monitors sales growth to identify trends and inform strategic business adjustments.


WITH MonthlySales AS (
    SELECT 
        DATE_FORMAT(fs.order_date, '%Y-%m') AS Sales_Month,
        ROUND(SUM(fs.quantity * p.price), 2) AS Monthly_Sales
    FROM 
        bens.fact_sales fs
        JOIN bens.dim_products p ON fs.product_id = p.product_id
    WHERE 
        fs.order_date IS NOT NULL
    GROUP BY 
        DATE_FORMAT(fs.order_date, '%Y-%m')
)
SELECT 
    Sales_Month,
    Monthly_Sales,
    ROUND(((Monthly_Sales - LAG(Monthly_Sales) OVER (ORDER BY Sales_Month)) / 
           LAG(Monthly_Sales) OVER (ORDER BY Sales_Month) * 100), 2) AS Growth_Rate_Percent
FROM 
    MonthlySales
ORDER BY 
    Sales_Month;


  E- CUSTOMER ACQUISITION, SEGMENTATION AND RENTENTION RATE : Focuses on attracting new customers and retaining existing ones.
    
-- 18. Which countries have the highest customer acquisition rate?
--  Identifies high-growth markets for targeted marketing and expansion efforts.


SELECT 
    c.country,
    COUNT(*) AS New_Customers,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM bens.dim_customers), 2) AS Acquisition_Rate_Percent
FROM 
    bens.dim_customers c
GROUP BY 
    c.country
ORDER BY 
    New_Customers DESC;

-- 19. How are customers segmented by total spending?
--  Enables tailored marketing strategies for different spending segments to maximize revenue.


SELECT 
    c.customer_id,
    CASE 
        WHEN SUM(fs.quantity * p.price) > 50000 THEN 'High Spenders'
        WHEN SUM(fs.quantity * p.price) BETWEEN 10000 AND 50000 THEN 'Medium Spenders'
        ELSE 'Low Spenders'
    END AS Spending_Segment,
    ROUND(SUM(fs.quantity * p.price), 2) AS Total_Spent
FROM 
    bens.dim_customers c
    JOIN bens.fact_sales fs ON c.customer_id = fs.customer_id
    JOIN bens.dim_products p ON fs.product_id = p.product_id
GROUP BY 
    c.customer_id
ORDER BY 
    c.customer_id,
    Total_Spent DESC;

-- 20. What is the distribution of marital status among top-spending customers?
--  Informs personalized offers for top spenders based on marital status insights.


WITH TopSpenders AS (
    SELECT 
        c.customer_id,
        c.marital_status,
        SUM(fs.quantity * p.price) AS Total_Spent,
        NTILE(4) OVER (ORDER BY SUM(fs.quantity * p.price) DESC) AS Spending_Quartile
    FROM 
        bens.dim_customers c
        JOIN bens.fact_sales fs ON c.customer_id = fs.customer_id
        JOIN bens.dim_products p ON fs.product_id = p.product_id
    GROUP BY 
        c.customer_id,
        c.marital_status
)
SELECT 
    marital_status,
    COUNT(*) AS Top_Spender_Count,
    ROUND(AVG(Total_Spent), 2) AS Avg_Spent
FROM 
    TopSpenders
WHERE 
    Spending_Quartile = 1
GROUP BY 
    marital_status
ORDER BY 
    Top_Spender_Count DESC;

-- 21. Which gender and marital status combinations have the highest order frequency?
--  Identifies high-frequency buyer demographics for targeted retention strategies.


SELECT 
    c.gender,
    c.marital_status,
    p.category,
    COUNT(*) AS Order_Count,
    COUNT(DISTINCT c.customer_id) AS Customer_Count
FROM 
    bens.dim_customers c
    JOIN bens.fact_sales fs ON c.customer_id = fs.customer_id
    JOIN bens.dim_products p ON fs.product_id = p.product_id
GROUP BY 
    c.gender,
    c.marital_status,
    p.category
ORDER BY 
    Order_Count DESC
LIMIT 
    5;
    
-- 22 . How do age groups differ in their purchasing behavior by category?
-- Helps customize inventory and promotions to match age-based buying preferences.


WITH AgeSegment AS (
    SELECT 
        c.customer_id,
        CASE 
            WHEN TIMESTAMPDIFF(YEAR, c.birth_date, CURDATE()) < 30 THEN 'Under 30'
            WHEN TIMESTAMPDIFF(YEAR, c.birth_date, CURDATE()) BETWEEN 30 AND 50 THEN '30-50'
            ELSE 'Over 50'
        END AS Age_Group
    FROM 
        bens.dim_customers c
)
SELECT 
    asg.Age_Group,
    p.category,
    COUNT(*) AS Purchase_Count,
    ROUND(AVG(fs.quantity * p.price), 2) AS Avg_Purchase
FROM 
    AgeSegment asg
    JOIN bens.fact_sales fs ON ASG.customer_id = fs.customer_id
    JOIN bens.dim_products p ON fs.product_id = p.product_id
GROUP BY 
    asg.Age_Group,
    p.category
ORDER BY 
    asg.Age_Group,
    Purchase_Count DESC;

 
-- 23. What is the churn rate of customers who joined this year?
--  Measures new customer retention to improve onboarding and reduce turnover.

```



### Results and Findings



### Recommendations




### Limitations




### References















