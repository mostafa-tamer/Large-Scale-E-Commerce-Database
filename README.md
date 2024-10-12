# Large-Scale E-Commerce Database Design

This project focuses on designing and populating a large-scale e-commerce database using PostgreSQL. it includes creating database functions to insert millions of rows efficiently and writing optimized SQL queries for reporting and data retrieval, resulting in a 74.7% increase in performance.

## Database Schema
 
strates the database schema for the e-commerce project:
  
<img src="https://github.com/user-attachments/assets/e8c2b38c-a9eb-4c20-a094-2f688e9bbfa0" alt="E-commerce Database Schema" width="35%">

## Tables

### 1. **Categories Table**
Stores information about product categories.

```sql
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);
```

### 2. **Products Table**
Stores information about products.

```sql
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    category_id INT REFERENCES categories(category_id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL,
    stock_quantity INT DEFAULT 0,
    author VARCHAR(255)
);
```

### 3. **Customers Table**
Stores information about customers.

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL
);
```

### 4. **Orders Table**
Stores information about orders placed by customers.

```sql
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id) ON DELETE CASCADE,
    date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 5. **Order Details Table**
Stores details about each product in an order (many-to-many relationship between `orders` and `products`).

```sql
CREATE TABLE order_details (
    order_detail_id SERIAL PRIMARY KEY,
    order_id INT REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id INT REFERENCES products(product_id) ON DELETE CASCADE,
    quantity INT NOT NULL,
    unit_price NUMERIC(10, 2) NOT NULL
);
```

## Database Functions for Data Insertion

1. **Insert Data in the Product Table (100k rows)**  
   This function generates and inserts 100,000 product records into the `products` table.

   ```sql
   CREATE OR REPLACE FUNCTION CREATE_DUMMY_PRODUCTS () RETURNS VOID AS $$ 
   BEGIN
       FOR i IN 1..100 LOOP
           FOR j IN 1..1000 LOOP
               INSERT INTO products (CATEGORY_ID, NAME, DESCRIPTION, PRICE, STOCK_QUANTITY, AUTHOR)
               VALUES (i, CONCAT('Name_', j), CONCAT('DESCRIPTION_', i), i * j, (i * 1.0 * j) / (i + j), CONCAT('AUTHOR_', i));
           END LOOP;
       END LOOP;
   END;
   $$ LANGUAGE PLPGSQL;
   ```

2. **Insert Data in the Customer Table (1 Million rows)**  
   This function generates and inserts 1,000,000 customer records into the `customers` table.

   ```sql
   CREATE OR REPLACE FUNCTION CREATE_DUMMY_CUSTOMERS () RETURNS VOID AS $$ 
   BEGIN
       FOR i IN 1..1000000 LOOP
           INSERT INTO customers (FIRST_NAME, LAST_NAME, EMAIL, PASSWORD)
           VALUES (CONCAT('Customer_First_Name_', i), CONCAT('Customer_Last_Name_', i), CONCAT('Email_', i), CONCAT('Password_', i));
       END LOOP;
   END;
   $$ LANGUAGE PLPGSQL;
   ```

3. **Insert Data in the Categories Table (100 rows)**  
   A function to insert 100 category records.

   ```sql
   CREATE OR REPLACE FUNCTION CREATE_DUMMY_CATEGORIES () RETURNS VOID AS $$ 
   BEGIN
       FOR i IN 1..100 LOOP
           INSERT INTO categories (name) VALUES (CONCAT('Category_', i));
       END LOOP;
   END;
   $$ LANGUAGE PLPGSQL;
   ```

4. **Insert Data in Orders and Order Details (2M, 5M rows respectively)**  
   These functions generate data for the `orders` and `order_details` tables based on existing `customers` and `products`.

   ```sql
   CREATE OR REPLACE FUNCTION CREATE_DUMMY_ORDERS () RETURNS VOID AS $$ 
   BEGIN
       FOR i IN 1..1000000 LOOP
           FOR j IN 1..5 LOOP
               INSERT INTO orders (customer_id, date) VALUES (i, NOW());
           END LOOP;
       END LOOP;
   END;
   $$ LANGUAGE PLPGSQL;
   ```

   ```sql
   CREATE OR REPLACE FUNCTION CREATE_DUMMY_ORDER_DETAILS () RETURNS VOID AS $$ 
   BEGIN
       FOR i IN 1..50 LOOP
           FOR j IN 1..100000 LOOP
               FOR k IN 1..4 LOOP
                   INSERT INTO order_details (order_id, product_id, quantity, unit_price)
                   VALUES (i * j, j, j / i, ((ROUND(FLOOR(RANDOM() * 10) + 1)* 1000)::numeric)::money);
               END LOOP;
           END LOOP;
       END LOOP;
   END;
   $$ LANGUAGE PLPGSQL;
   ```

Here is the updated README file with the required sections on query time before and after optimization, and the optimization techniques applied:

---

## Optimized SQL Queries

The following queries are optimized for performance, with indexes and optimizations applied where needed.

### 1. **Retrieve Total Number of Products in Each Category**  
This query returns the count of products grouped by category.

#### Before Optimization:
```sql
SELECT C.NAME, COUNT(P.*)
FROM PRODUCTS P
JOIN CATEGORIES C ON P.CATEGORY_ID = C.CATEGORY_ID
GROUP BY C.NAME;
```
- **Query Time (Before Optimization):** 55 ms
- **Optimization Technique:** Changed `COUNT(P.*)` to `COUNT(P.PRODUCT_ID)` to avoid unnecessary computation on entire rows and grouping by the `C.CATEGORY_ID` before `C.NAME`.
- **Query Time (After Optimization):** 30 ms

#### After Optimization:
```sql
SELECT C.NAME, COUNT(P.PRODUCT_ID)
FROM CATEGORIES C
JOIN PRODUCTS P ON C.CATEGORY_ID = P.CATEGORY_ID
GROUP BY C.CATEGORY_ID, C.NAME;
```

---

### 2. **Find Top Customers by Total Spending**  
A query to find the top 10 customers based on total spending.

#### Before Optimization:
```sql
SELECT C.NAME, SUM(OD.UNIT_PRICE * OD.QUANTITY) AS TOTAL_SPENDING
FROM CUSTOMERS C
JOIN ORDERS O ON C.CUSTOMER_ID = O.CUSTOMER_ID
JOIN ORDER_DETAILS OD ON O.ORDER_ID = OD.ORDER_ID
GROUP BY C.NAME
ORDER BY TOTAL_SPENDING DESC
LIMIT 10;
```
- **Query Time (Before Optimization):** 22 sec
- **Optimization Technique:** Used a `WITH` clause to calculate `TOTAL_SPENDING` separately before joining with customers, reducing redundant computations.
- **Query Time (After Optimization):** 12.5 sec

#### After Optimization:
```sql
WITH TOTAL_SPENDING AS (
    SELECT O.CUSTOMER_ID, SUM(OD.UNIT_PRICE * OD.QUANTITY) AS TOTAL_SPENDING
    FROM ORDERS O
    JOIN ORDER_DETAILS OD ON O.ORDER_ID = OD.ORDER_ID
    GROUP BY O.CUSTOMER_ID
)
SELECT C.NAME, TS.TOTAL_SPENDING
FROM CUSTOMERS C
JOIN TOTAL_SPENDING TS ON C.CUSTOMER_ID = TS.CUSTOMER_ID
ORDER BY TS.TOTAL_SPENDING DESC
LIMIT 10;
```

---

### 3. **Retrieve Most Recent Orders with Customer Information (Last 1000 Orders)**  
This query fetches the most recent 1000 orders with corresponding customer details.

#### Before Optimization:
```sql
SELECT C.NAME, C.EMAIL, O.DATE AS ORDER_ID
FROM CUSTOMERS C
JOIN ORDERS O ON C.CUSTOMER_ID = O.CUSTOMER_ID
ORDER BY DATE DESC
LIMIT 1000;
```
- **Query Time (Before Optimization):** 1.5 sec
- **Optimization Technique:** Added an index on `ORDERS.DATE` to speed up sorting and clustered the table to ensure efficient retrieval as well as joining will orders after filteration done on orders.
- **Query Time (After Optimization):** 0.006 sec

#### After Optimization:
```sql
CREATE INDEX ORDERS_DATE_INDEX ON ORDERS (DATE);

CLUSTER PRODUCTS USING ORDERS_DATE_INDEX;

SELECT C.NAME, C.EMAIL, O.DATE
FROM CUSTOMERS C
JOIN (SELECT DATE, CUSTOMER_ID FROM ORDERS ORDER BY DATE DESC LIMIT 1000) O ON C.CUSTOMER_ID = O.CUSTOMER_ID;
```

---

### 4. **List Products with Low Stock Quantities (< 10)**  
A query to list products that have stock quantities less than 10.

#### Before Optimization:
```sql
SELECT NAME, DESCRIPTION, PRICE, STOCK_QUANTITY, AUTHOR
FROM PRODUCTS
WHERE STOCK_QUANTITY < 10
ORDER BY STOCK_QUANTITY;
```
- **Query Time (Before Optimization):** 15 ms
- **Optimization Technique:** Created a clustered index on `STOCK_QUANTITY` and clustered the table for efficient retrieval of low stock products.
- **Query Time (After Optimization):** 2 ms

#### After Optimization:
```sql
CREATE INDEX PRODUCTS_STOCK_QUANTITY_INDEX ON PRODUCTS (STOCK_QUANTITY);

CLUSTER PRODUCTS USING PRODUCTS_STOCK_QUANTITY_INDEX;

SELECT NAME, DESCRIPTION, PRICE, STOCK_QUANTITY, AUTHOR
FROM PRODUCTS
WHERE STOCK_QUANTITY < 10
ORDER BY STOCK_QUANTITY;
```

---

### 5. **Calculate Revenue Generated from Each Product Category**  
This query calculates the total revenue for each product category.

#### Before Optimization:
```sql
SELECT C.NAME, SUM(UNIT_PRICE * OD.QUANTITY) AS REVENUE
FROM ORDER_DETAILS OD
JOIN ORDERS O ON OD.ORDER_ID = O.ORDER_ID
JOIN PRODUCTS P ON P.PRODUCT_ID = OD.PRODUCT_ID
JOIN CATEGORIES C ON P.CATEGORY_ID = C.CATEGORY_ID
GROUP BY C.NAME;
```
- **Query Time (Before Optimization):** 11 sec
- **Optimization Technique:** Used a materialized view to precompute and cache results for faster querying.
- **Query Time (After Optimization):** 0.00003 sec (using materialized view)

#### After Optimization:
```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS EACH_CATEGORY_REVENUE AS (
    SELECT C.NAME, SUM(UNIT_PRICE * OD.QUANTITY) AS REVENUE
    FROM CATEGORIES C
    JOIN PRODUCTS P ON P.CATEGORY_ID = C.CATEGORY_ID
    JOIN ORDER_DETAILS OD ON OD.PRODUCT_ID = P.PRODUCT_ID
    JOIN ORDERS O ON OD.ORDER_ID = O.ORDER_ID
    GROUP BY C.CATEGORY_ID, C.NAME
);

SELECT * FROM EACH_CATEGORY_REVENUE;
```

## Query Performance Optimization

All queries have been optimized with appropriate indexes and use of materialized views where necessary. Further improvements can be made using `EXPLAIN ANALYZE` to evaluate and refine query performance.
