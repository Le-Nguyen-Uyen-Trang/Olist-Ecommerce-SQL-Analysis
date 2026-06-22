# Olist-Ecommerce-SQL-Analysis
Olist E-Commerce (Brazil) data analysis project using PostgreSQL. Solving key business problems regarding Revenue, Customer Analysis (RFM), Logistics performance, and Product Quality.
#  Olist E-Commerce Data Analysis Using SQL

## 1. Introduction
This project utilizes **PostgreSQL (pgAdmin)** to analyze real-world e-commerce data from Olist (Brazil). The primary objective is to transform raw data into actionable business insights covering sales optimization, product quality, seller performance, and logistics management.

### 2. Data Schema & Table Meanings

####  Customers & Sellers (Entities)
* **`Olist_customers`**: Manages buyer profiles. Crucial for distinguishing between transaction-based IDs (`customer_id`) and unique real-world individual IDs (`customer_unique_id`) to perform retention (RFM) analysis.
* **`Olist_sellers`**: Contains seller identifiers (`seller_id`) and location details to track and evaluate merchant performance.

####  Products & Categories (Merchandise)
* **`Olist_products`**: Stores physical specifications of items (weight, dimensions, photo counts) and category names in Portuguese.
* **`Olist_category_name_translation`**: A lookup dictionary mapping Portuguese category names to English for global business reporting.

####  Orders, Logistics & Payments (Transactions)
* **`Olist_orders`**: The core backbone managing the order lifecycle and critical milestones (purchase, approval, shipping, and delivery dates) to calculate cancellation and delay rates.
* **`Olist_order_items`**: Records individual items within each shopping cart, containing item sequence (`item_id`), base price (`price`), and shipping cost (`freight_value`). This is the primary table for revenue computation.
* **`Olist_order_payments`**: Tracks cash flow, recording payment methods (credit card, boleto voucher), installments, and the actual transactional values collected.

####  Reviews & Geolocation (Advanced Operations)
* **`Olist_order_reviews`**: Measures customer satisfaction by capturing review scores (1-5 stars) and textual feedback for quality control.
* **`Olist_geolocation`**: Stores GPS coordinates (latitude and longitude) linked to zip codes for geospatial heatmaps and delivery distance computations.


## 3. Business Questions & SQL Queries

###  Topic 1: Sales & Seller Optimization

#### Query 1: Top 10 revenue-generating products and their categories
```sql
SELECT 
  Olist_order_items.product_id,
	Product_category_name,
  SUM(price) AS total_revenue  
FROM Olist_order_items JOIN Olist_products ON Olist_order_items.product_id = Olist_products.product_id 
GROUP BY Olist_order_items.product_id, Product_category_name
ORDER BY total_revenue DESC
LIMIT 10;
```
* ** Business Insight:**  Finds top-selling products to focus on running ads and stocking up inventory in main warehouses. Conversely, it can also find slow-moving products to give discounts or stop selling them.

---

**#### Query 2: Top 10 sellers by total revenue**
```sql
SELECT 
    seller_id,
    SUM(price) AS total_revenue
FROM Olist_order_items 
GROUP BY seller_id
ORDER BY total_revenue DESC 
LIMIT 10;
```
* ** Business Insight:**  Finds top sellers with high revenue to give them special rewards. Conversely, it can also find low-performing sellers who need retraining or warnings to improve their sales. 

---

###  Topic 2: Logistics & Geography

#### Query 3: What is the overall late delivery rate?
```sql
SELECT 
    100.0 * SUM(CASE WHEN order_estimated_delivery_date < order_delivered_customer_date THEN 1 ELSE 0 END) 
    / COUNT(order_id) AS late_rate
FROM Olist_orders
WHERE order_delivered_customer_date IS NOT NULL;
```
* ** Business Insight:**  Logistics Quality Control: Measures delivery punctuality to safeguard customer experience. Elevated delay rates signal a critical need to re-evaluate 3PL partnerships and optimize fulfillment workflows. (Calculated on completed deliveries).

#### Query 4: Which city generates the highest revenue?
```sql
SELECT customer_city, SUM(price) AS total_revenue
FROM Olist_order_items 
JOIN Olist_orders ON Olist_orders.order_id = Olist_order_items.order_id
JOIN Olist_customers ON Olist_orders.customer_id = Olist_customers.customer_id
GROUP BY customer_city
ORDER BY total_revenue DESC
LIMIT 1
```
* ** Business Insight:** Pinpoints core regional hubs to strategically drive local infrastructure investments. Identifying high-volume cities triggers proactive expansion of fulfillment centers, directly minimizing transit times and secondary logistics overheads.
---

###  Topic 3: Customer Behavior & Product Quality

#### Query 5: Who is the highest-value customer (Highest monetary spender)?
```sql
SELECT customer_unique_id, SUM(price)
FROM Olist_orders 
JOIN Olist_order_items ON Olist_orders.order_id = Olist_order_items.order_id
JOIN Olist_customers ON Olist_customers.customer_id= Olist_orders.customer_id
GROUP BY customer_unique_id
ORDER BY  SUM(price) DESC
LIMIT 1
```
* ** Business Insight:** Identifies top-spending customers ("Whales") based on total revenue generated. This allows marketing teams to target them with exclusive loyalty rewards to keep them buying.

#### Query 6: Top 10 worst-reviewed products (High order volume but lowest average scores)
```sql
SELECT product_id, AVG(review_score)
FROM Olist_order_reviews AS reviews
JOIN Olist_order_items AS items
ON reviews.order_id = items.order_id
GROUP BY product_id
HAVING COUNT(reviews.order_id) >10
ORDER BY AVG(review_score) ASC 
LIMIT 10
```
* ** Business Insight:** Flags frequently-ordered products (volume > 10 reviews) that have actual quality issues based on poor ratings. This helps the platform remove bad items or penalize sellers to reduce customer complaints.

###  Topic 4: Time-Series Trends

#### Query 7: Monthly revenue trend (Using `DATE_TRUNC`)
```sql
---
Option 1: Using DATE_TRUNC 
SELECT 
    DATE_TRUNC('month', order_estimated_delivery_date) AS order_month, 
    SUM(price) AS total_revenue
FROM Olist_order_items 
JOIN Olist_orders ON Olist_order_items.order_id = Olist_orders.order_id
GROUP BY DATE_TRUNC('month', order_estimated_delivery_date)
ORDER BY order_month ASC;

Option 2: Using TO_CHAR 
SELECT 
    TO_CHAR(ord.order_estimated_delivery_date, 'YYYY-MM') AS order_month, 
    SUM(item.price) AS total_revenue 
FROM Olist_order_items AS item 
JOIN Olist_orders AS ord ON item.order_id = ord.order_id 
GROUP BY TO_CHAR(ord.order_estimated_delivery_date, 'YYYY-MM')
ORDER BY order_month ASC;

Option 3: Using EXTRACT 
SELECT 
    EXTRACT(YEAR FROM ord.order_estimated_delivery_date) AS order_year,
    EXTRACT(MONTH FROM ord.order_estimated_delivery_date) AS order_month,
    SUM(item.price) AS total_revenue
FROM Olist_order_items AS item
JOIN Olist_orders AS ord ON item.order_id = ord.order_id
GROUP BY 
    EXTRACT(YEAR FROM ord.order_estimated_delivery_date),
    EXTRACT(MONTH FROM ord.order_estimated_delivery_date)
ORDER BY order_year ASC, order_month ASC;

* ** Business Insight:** Business Insight: Tracks monthly revenue growth and identifies seasonal sales patterns. This helps the business plan inventory, manage cash flow, and prepare marketing budgets for peak months.

```
## 4. Conclusion
Through this SQL framework, raw e-commerce records are successfully translated into concrete data-driven recommendations: optimizing product listings, positioning fulfillment networks, filtering out quality defects, and isolating high-tier consumer cohorts.
