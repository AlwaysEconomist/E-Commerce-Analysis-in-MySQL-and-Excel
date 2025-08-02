# Data-Analysis-Project-Using-MySQL
Product and Sales Performance , Customer Behavior Analysis &amp; Inventory Management Project Using MySQL


CREATE DATABASE bens;
USE bens;

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
    stock BIGINT
);

CREATE TABLE fact_sales (
    sale_id INT PRIMARY KEY,
    customer_id INT,
    product_id INT,
    order_date DATE,
    quantity INT
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

    
        -- A- STOCK INVENTORY MANAGEMENT : Focuses on managing and analyzing inventory levels to optimize stock and prevent shortages or overstocking.

-- 1. Which products have zero stock / are at risk of stockout /normal stock / are overstocked ?

SELECT 
    product_id,
    product_name,
    category,
    stock,
    CASE
        WHEN stock = 0 THEN 'Out_Of_Stock'
        WHEN stock BETWEEN 0 AND 100 THEN 'At Risk'
        WHEN stock BETWEEN 100 AND 400 THEN 'Normal'
        WHEN stock > 0 THEN 'Overstocked'
        ELSE 'Ouffff'
    END AS Stock_Inventory_Segmentation
FROM
    bens.dim_products
; 
    
-- 2. What is the stock value of each category?

SELECT 
    category, ROUND(SUM(price * stock), 2) AS StockValue
FROM
    bens.dim_products 
GROUP BY category
ORDER BY StockValue DESC;

-- 3. Which products have the highest stock-to-sales ratio?

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
    


            
                 -- B- PRODUCT PERFORMANCE: Analyzes product sales, profitability, and market trends to guide product strategy.


-- 5. Which products are the top 5 sellers by quantity?

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


 
      -- C- LOYAL CUSTOMER ANALYSIS : Focuses on identifying and analyzing loyal or high-value customers.

-- 10. Which customers have not made any purchases/ inactive customers?

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
    


	        -- D- REVENUE ANALYSIS : Examines overall sales performance, trends, and growth.


-- 14. What is the total sales amount by country?

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

-- 15. What is the total sales by month and the month-over-month sales growth rate?

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


		 **-- E- CUSTOMER ACQUISITION, SEGMENTATION AND RENTENTION RATE : Focuses on attracting new customers and retaining existing ones.**

    
-- 16. Which countries have the highest customer acquisition rate?

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

-- 17. How are customers segmented by total spending?

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

-- 18. What is the distribution of marital status among top-spending customers?

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

-- 19. Which gender and marital status combinations have the highest order frequency?

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
    
-- 20 . How do age groups differ in their purchasing behavior by category?

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

 

























