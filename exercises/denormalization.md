## Normalization and Denormalization

When we study relational data modeling and schema design, we learn to normalize our data.  That is, we learn to create a model of our objects and the relationships between them in such a way that any one piece of data is stored exactly once in the database.  In a completely normalized database, any single piece of data is represented in one column in one record of one table.  That logical "cell" (row and column in a table) represents the single source of truth for that data, one place to look up or update its value.

### Benefits of Normalization

Normalization is considered a best practice in relational database schema design because normalized data organization carries a host of benefits:

**Logical Data Representation**: A normalized schema often maps naturally to the business requirements, where each table represents a logical grouping of data (such as Customers, Orders, or Products). This makes the data model easier to understand, document, and extend as the business grows.

**Eliminate Redundancy**: By organizing data in a normalized way, you avoid duplicate information, which reduces the risk of inconsistencies. For instance, storing customer details in a single Customers table rather than repeating them in each Orders row ensures that if a customer’s information changes, only one update is needed.

**Simpler Updates**: In normalized tables, changes to data only need to be made in one place, simplifying updates. For example, an update to a product’s description in a normalized ```Products``` table will be reflected wherever that product is referenced through foreign keys, avoiding errors that could occur from updating redundant data in multiple places.

**Reduce storage requirements**: By eliminating redundant data, normalized databases require less storage. This makes them more efficient and less costly to scale, particularly important as data grows.

**Ensure Consistency**: In a normalized database, updates, deletions, and insertions can be more precisely managed, reducing errors. Consistency is guaranteed through constraints like primary keys, foreign keys, and unique constraints, which ensure that each piece of data adheres to its intended relationships.

**Adaptable Schema**: As the data model grows, normalized structures make it easier to add new entities, attributes, relationships, and constraints. This modularity allows you to scale the schema without risking data inconsistency or complexity.  The modular structure allows you to modify tables with minimal impact on other parts of the database.

**Targeted Indexing**: With unique data in each table, indexing becomes more effective. Indexes can be created on individual tables without being complicated by data duplication, speeding up query response times for searches, updates, and deletes.

**Reduced Anomalies**: Normalization reduces the three main types of anomalies: insertion, update, and deletion anomalies. For example, normalization prevents insertion anomalies where you’d need to create a “dummy” value just to satisfy database constraints, or update anomalies where you’d need to update data in multiple places to keep it consistent.

### When to Denormalize

While perfect normalization is an academic ideal with real-world benefits, it can also have real-world negative effects on an applications performance.  The primary cost of normalization is the need to JOIN multiple tables in a SQL query to get the data we want.

As an example, consider an e-commerce platform with three tables in a normalized relational database schema:

* **Products** table ontains detailed information about each product.
* **Orders** table contains information specific to each order.
* **OrderItems** table references both the Orders and Products tables. Each line item in an order will have its own row in OrderItems, specifying the product and quantity.

```
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY,
    CustomerID INT,
    OrderDate DATE
);

CREATE TABLE OrderItems (
    OrderItemID INT PRIMARY KEY,                  -- Unique identifier for each line item
    OrderID INT REFERENCES Orders(OrderID),       -- Foreign key reference to Orders
    ProductID INT REFERENCES Products(ProductID), -- Foreign key reference to Products
    Quantity INT
);

CREATE TABLE Products (
    ProductID INT PRIMARY KEY,
    ProductName VARCHAR(100),
    ProductPrice DECIMAL(10,2)
);
```

### Exercise

Given the schema above, write a query that will return all the information necessary to print a complete order form or invoice.

What additional work does your application have to do to create a detailed invoice?

### Explanation

Here's one such query.  It returns a row for every line item in the specified order, with columns containing data for from all 3 tables:

```
SELECT 
    o.OrderID,
    o.CustomerID,
    o.OrderDate,
    oi.OrderItemID,
    oi.ProductID,
    p.ProductName,
    p.ProductPrice,
    oi.Quantity,
    (oi.Quantity * p.ProductPrice) AS ItemTotal
FROM 
    Orders o
JOIN 
    OrderItems oi ON o.OrderID = oi.OrderID
JOIN 
    Products p ON oi.ProductID = p.ProductID
WHERE 
    o.OrderID = $1;
```

Every line item of the order is represented, and the query does the math to calculate a line item total.  It would take an additional query or sub-query of the same data to produce an order total, so that's work that we might leave to the application.  

Note that this exercise is not about using indexes to narrow the scope of search; it's about managing joins.  So we're going to ignore the fact that we have no CustomerID or OrderDate index on Orders, and no OrderID index in OrderItems.  Our query will do a full table scan on both the ```Orders``` table and the ```OrderItems``` table in order to emphasize the cost of the joins.

### Exercise

In a high-traffic e-commerce application where orders are frequently viewed, querying with joins across these two tables can become inefficient as the dataset grows. Every time an order is displayed, the database has to fetch the product details by joining tables, which can be slow for large datasets.  To optimize read performance, you can denormalize the data by including copies of often-referenced Product fields in the OrderItems record.  

For this exercise, modify the schema and the query to significantly reduce the time and effort to produce the Order summary above.

### Explanation

Since the Order summary will necessarily include the name and price of the product in each line item, we want to duplicate the ```ProductName``` and ```ProductPrice``` from the ```Products``` table in the directly into the ```OrderItems``` table. This avoids the need to join with the ```Products``` table when fetching Order details.  
```
-- Denormalized columns, values to be copied from the Products record with the referenced ProductID
ALTER TABLE OrderItems ADD COLUMN ProductName VARCHAR(100);
ALTER TABLE OrderItems ADD COLUMN ProductPrice DECIMAL(10,2);
```

In this denormalized design, the ProductName and ProductPrice are copied into the Orders table, so when fetching orders, there's no need to join with the Products table.
```
SELECT 
    o.CustomerID, o.OrderID,
    oi.ProductName, oi.ProductPrice,
    SUM(oi.Quantity * oi.ProductPrice) AS TotalSpent
FROM 
    Orders o
JOIN 
    OrderItems oi ON o.OrderID = oi.OrderID
WHERE 
    o.CustomerID = 12345  
    AND YEAR(o.OrderDate) = 2023
GROUP BY 
    o.CustomerID;
```

Note that this change also addresses another problem with this data design.  What happens if the description or price of a Product record is changed?  Answer: all past orders reference the now-updated product record, meaning that the historical record of what was purchased, and for how much, can be affected by later updates to our Products listing.


### Exercise

What if we wanted simply to be able to total the value of orders from a particular customer in a particular date range?  

Write a query that returns the total amount spent by a customer in the year 2023.

In the test database, how long does it take to execute the query for customer ```12345```?  How many rows does the query have to read?

### Explanation

Here's a query that satisfies the exercise above.
```
SELECT 
    o.CustomerID, SUM(oi.Quantity * oi.ProductPrice) AS TotalSpent
FROM 
    Orders o
JOIN 
    OrderItems oi ON o.OrderID = oi.OrderID
WHERE 
    o.CustomerID = 12345  
    AND YEAR(o.OrderDate) = 2023
GROUP BY 
    o.CustomerID;
```

The query is faster than the first query in this section, but it still reads every record from ```Orders``` looking for orders that match our criteria, and every record in the ```OrderItems``` table to find the line items and calculate the line item prices.

### Exercise

Even with the denormalization we applied earlier, we have to sum all the line items for all the orders belonging to the customer.  That **still** requires a 2-way join, because we need the ```Order``` table to filter by customer and date, and we need the ```OrderItems``` to calculate the line item values to be summed.

Apply a similar denormalization by adding an ```OrderTotal``` column to the ```Orders``` table and write the new query to calculate the total amount spent by a given customer in a given year.  How much faster is that query?  How much less data must it read to produce the result?

### Conclusion

Benefits of Denormalization:
* **Faster reads**: You avoid joins, leading to faster query performance, especially for frequently accessed data (like product details for orders).
* **Simplified queries**: Queries become simpler since you don’t need to join tables for product details.

Trade-offs:
* **Data redundancy**: The ProductName and ProductPrice are now stored redundantly in the Orders table, meaning if a product’s details change, you need to update all corresponding rows in the Orders table.
* **Increased storage**: Since you're storing product information multiple times, this increases storage requirements.
* **Complexity in updates**: If the ProductPrice changes, you may need to update it in both the Products table and any rows in the OrderItems table that hasn't been finalized or fulfilled.

When Is This a Good Idea?
* **Read-heavy systems**: If the system is read-heavy and users frequently view orders, denormalization can improve performance by reducing the number of joins.
* **Data that's not updated often**: Denormalization works well when the denormalized data (like ProductName and ProductPrice) doesn’t change often. If these values change frequently, updating redundant data across the database can become a challenge.

[Next Section](aost.md)

