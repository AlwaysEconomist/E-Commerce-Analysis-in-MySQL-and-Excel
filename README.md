# E-commerce Analysis Using MySQL

## Table of Contents
 - [Project Overview](#project-overview)
 - [Data Analysis](#data-analysis)
 - [Results and Findings](#results-and-findings)
 - [Recommendations](#recommendations)
 - [References](#references)

### Project Overview
This project revolves around optimizing and analyzing a toy store's operations answering key business questions, ranging from inventory management to customer behavior insights. The task involves crafting SQL queries to address some specific questions, covering areas like stock levels (e.g., identifying zero stock or overstocked products), product performance (e.g., top-selling categories), loyal customer analysis (e.g., inactive customers), sales and revenue trends (e.g., month-over-month growth), customer acquisition and retention (e.g., new customers per month), and detailed customer segmentation (e.g., by gender, age, or spending). These queries help solve practical business problems, such as preventing stockouts, targeting high-value customers, and tailoring marketing strategies.


<img width="769" height="451" alt="image" src="https://github.com/user-attachments/assets/fdf87edc-39d4-4f2c-914b-4898ab80f870" />







### Data Sources 
The database consists of three main tables: dim_customers, which stores customer details, fact_sales which tracks sales data and dim_products, which holds product information. These dimensions tables (customers and products) are joined to the fact sales table using customer_id and product_id.

### Tools used for this project

- Power Query in Excel : Data Cleaning .
- MySQL : Data Analysis.
- Excel : Visualization with simple Bar, Column, Line and Pie charts.

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

-- Database Structure

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
    product_name VARCHAR(50),
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

      -- QUERY OPTIMIZATION THROUGH INDEXES
                         
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

    
-- A. STOCK INVENTORY MANAGEMENT: Focuses on managing and analyzing inventory levels to optimize stock and prevent shortages or overstocking.

-- 1. Which products have zero stock / are at risk of stockout /normal stock / are overstocked ?
-- Helps identify products needing restocking to prevent stockouts and optimize inventory levels, while reducing excess inventory to lower holding costs and free up storage space.

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

-- 3. Predict stock depletion based on average daily sales?
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
    

 -- B. PRODUCT PERFORMANCE: Analyzes product sales, profitability, and market trends to guide product strategy.

-- 4. Which products are the top 5 sellers by quantity?
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

-- 5. What is the average order value by category?
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

-- 6. Which products contribute to 80% of total sales (Pareto analysis)?
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

-- 7. What is the total sales, total cost, quantity sold, profit and profit margin by product category?
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
    
-- 8. Which products have the highest profit margin?
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


      -- C. LOYAL CUSTOMER ANALYSIS : Focuses on identifying and analyzing loyal or high-value customers.

-- 9. Which customers have not made any purchases/ inactive customers?
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
            

-- 10. Which customers are in the top 10% by total spending?
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
    
-- 11. What is the average time between purchases for each customer? (purchases frequency)
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
    
--12.  What is the average sales amount per customer?   

SELECT 
    c.customer_id, c.first_name, c.last_name, ROUND(AVG(fs.sales_amount), 2) AS avg_sales
FROM bens.dim_customers c
     JOIN bens.fact_sales fs ON c.customer_id = fs.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name;


   -- D. REVENUE ANALYSIS : Examines overall sales performance, trends, and growth.

-- 13. What is the total sales amount by country?
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

-- 14. What is the total sales by month and the month-over-month sales growth rate?
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


  -- E. CUSTOMER ACQUISITION, SEGMENTATION AND RENTENTION RATE : Focuses on attracting new customers and retaining existing ones.
    
-- 15. Which countries have the highest customer acquisition rate?
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

-- 16. How are customers segmented by total spending?
--  Enables tailored marketing strategies for different spending segments to maximize revenue.

SELECT 
    c.customer_id,
    CASE 
        WHEN SUM(fs.quantity * p.price) > 10000 THEN 'High Spenders'
        WHEN SUM(fs.quantity * p.price) BETWEEN 5000 AND 10000 THEN 'Medium Spenders'
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

-- 17. What is the distribution of marital status among top-spending customers?
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

-- 18. Which gender and marital status combinations have the highest order frequency?
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
    
-- 19 . How do age groups differ in their purchasing behavior by category?
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

-- 20. What is the churn rate of customers who joined this year?  
-- Measures new customer retention to improve onboarding and reduce turnover.

WITH ActiveCustomers AS (
    SELECT 
        c.customer_id,
        MIN(fs.order_date) AS First_Purchase
    FROM 
        bens.dim_customers c
        JOIN bens.fact_sales fs ON c.customer_id = fs.customer_id
    WHERE 
        c.join_date >= '2025-01-01' AND c.join_date <= CURDATE()
    GROUP BY 
        c.customer_id
)
SELECT 
    COUNT(*) AS Total_New_Customers,
    COUNT(CASE 
        WHEN NOT EXISTS (
            SELECT 1 
            FROM bens.fact_sales fs2 
            WHERE fs2.customer_id = ac.customer_id 
            AND fs2.order_date > DATE_ADD(ac.First_Purchase, INTERVAL 1 MONTH)
        ) THEN 1 
        ELSE NULL 
    END) AS Churned_Customers,
    ROUND((COUNT(CASE WHEN NOT EXISTS (
        SELECT 1 
        FROM bens.fact_sales fs2 
        WHERE fs2.customer_id = ac.customer_id 
        AND fs2.order_date > DATE_ADD(ac.First_Purchase, INTERVAL 1 MONTH)
    ) THEN 1 ELSE NULL END) * 100.0 / COUNT(*)), 2) AS Churn_Rate_Percent
FROM 
    ActiveCustomers ac;


-- 21. Which customers are segmented by recency of purchase (last 30, 60, 90 days)?
-- Helps you send reminders to customers who haven’t purchased in a while!

SELECT 
    c.customer_id,
    CONCAT(c.last_name, ' ', c.first_name) AS FullName,
    CASE 
        WHEN MAX(fs.order_date) >= DATE_SUB(CURDATE(), INTERVAL 30 DAY) THEN 'Active (0-30 days)'
        WHEN MAX(fs.order_date) >= DATE_SUB(CURDATE(), INTERVAL 60 DAY) THEN 'Recent (31-60 days)'
        WHEN MAX(fs.order_date) >= DATE_SUB(CURDATE(), INTERVAL 90 DAY) THEN 'Lapsed (61-90 days)'
        ELSE 'Inactive (>90 days)'
    END AS Recency_Segment
FROM 
    bens.dim_customers c
    LEFT JOIN bens.fact_sales fs ON c.customer_id = fs.customer_id
GROUP BY 
    c.customer_id,
    c.first_name,
    c.last_name
ORDER BY 
    FIELD(Recency_Segment, 'Active (0-30 days)', 'Recent (31-60 days)', 'Lapsed (61-90 days)', 'Inactive (>90 days)');
```


### Results and Findings
 - The inventory reveals that 40% of products are overstocked, 28% maintain normal stock levels, 30% are at risk of stockout, and a mere 2% are out of stock, emphasizing the need for strategic adjustments to optimize stock distribution and prevent potential sales losses.
   
<img width="832" height="401" alt="image" src="https://github.com/user-attachments/assets/6c7e2fb8-d235-40ee-90bf-cc81cd3cf080" />


   
 - Based on the average daily sales, electronics and books are projected to be out of stock within 90 days, necessitating urgent restocking efforts for these high-demand categories. In contrast, other product categories are expected to last up to five months, allowing for more flexible inventory planning and management.
   
<img width="750" height="450" alt="image" src="https://github.com/user-attachments/assets/47bdef59-e3c0-4cae-a27b-121f9d10650e" />


   
 - 80% of the store's revenue is driven by just 26 products, accounting for 52% of the total product lineup (50 products),
 with 6 standout performers (Move Plus, Bring Pro, Her Lite, Thank Lite, Voice Pro, and Compare Plus) each generating over $1 million.
 - The profit margin across each product category ranges from a solid 29% to 33%, reflecting a healthy and consistent profitability that strengthens the store's financial performance and supports future growth initiatives.
 - The categories of Home Appliances, Sports, and Electronics emerge as top performers, each generating revenue exceeding $5 million, underscoring their  significant contribution to the store's overall financial success and highlighting key areas for strategic focus and investment.
 - The top 10 products boasting an impressive profit margin of approximately 50% are predominantly from the home appliances and sports categories,
 highlighting these segments as key drivers of high profitability and potential areas for targeted expansion.
 - An impressive 86% of customers in the database have made at least one purchase, reflecting an outstanding conversion rate that showcases the store's ability to effectively turn visitors into loyal buyers, but 80% of them are inactive that means they do not make any purchases for more than 90 days.
 - The total customer acquisition rate across countries remains stable at approximately 20%, indicating a consistent and reliable growth in new customers that supports the store's ongoing expansion and market penetration efforts.
 - An overwhelming 98% of customers spend between $0 and $10,000 in the store, highlighting a broad base of low-to-mid-range spenders that underscores the need for targeted strategies to encourage higher spending among this dominant segment.
 - The primary customer base consists of adults aged 30 and older, indicating a focus on a mature demographic.
 - The analysis shows monthly sales fluctuating between $970K and $1.4M, with notable peaks around March 2024 and January 2025. The growth rate percentage varies significantly, reaching up to 25% during peak sales periods and dropping to -15% during declines.

<img width="1273" height="641" alt="image" src="https://github.com/user-attachments/assets/45aaadea-a527-4e83-995a-faf0d6641124" />



### Recommendations
 - Prioritize restocking electronics and books within the next 90 days to avoid stockouts, while optimizing overstocked inventory (40%) by redistributing excess stock to high-demand categories or offering promotions to clear surplus.
 - Focus marketing and inventory investment on the top-performing categories (Home Appliances, Sports, Electronics) and the 6 standout products (Move Plus, Bring Pro, Her Lite, Thank Lite, Voice Pro, Compare Plus), which drive 80% of revenue, to maximize profitability and sales.
 - Develop targeted reactivation campaigns for the 80% of inactive customers (out of the 86% who have purchased) to re-engage them, leveraging their past purchase data to encourage repeat buys and increase overall sales.
 - Introduce strategies to upsell or cross-sell to the 98% of customers spending $0-$10,000, such as bundling high-margin products from Home Appliances and Sports (50% profit margin) to boost average order value.
 - Tailor marketing and product offerings to the mature demographic (80% aged 30+), aligning seasonal promotions with peak sales periods (e.g., March and January) to capitalize on growth rates up to 25% and mitigate declines.
 - The healthy profit margin range of 29%-33% across all categories, combined with a 50% margin on top 10 products, indicates an opportunity to explore premium product lines to enhance overall profitability.
 - The consistent 20% customer acquisition rate across countries suggests a stable market expansion potential, which could be leveraged by introducing region-specific promotions to further boost growth.

  
### References

 - "SQL for Data Scientists: A Beginner's Guide for Building Datasets for Analysis" by Renee M. P. Teate
 - "The Data Warehouse Toolkit: The Definitive Guide to Dimensional Modeling" by Ralph Kimball and Margy Ross
 - "SQL Performance Explained" by Markus Winand













