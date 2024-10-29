## Normalization and Denormalization

When we're taught relational data modeling and schema design, we're taught to normalize our data.  That is, we learn to create a model of our objects and the relationships between them in such a way that any one piece of data is stored exactly once in the database.  In a completely normalized database, any single piece of data is represented in one column in one record of one table.  That logical "cell" (row and column in a table) represents the single source of truth for that data, one place to look up or update its value.

### Benefits of Normalization

Normalization is considered a best practice in relational database schema design because normalized data organization carries a host of benefits:

**Logical Data Representation**: A normalized schema often maps naturally to the business requirements, where each table represents a logical grouping of data (such as Customers, Orders, or Products). This makes the data model easier to understand, document, and extend as the business grows.

**Eliminate Redundancy**: By organizing data in a normalized way, you avoid duplicate information, which reduces the risk of inconsistencies. For instance, storing customer details in a single Customers table rather than repeating them in each Orders row ensures that if a customer’s information changes, only one update is needed.

**Simpler Updates**: In normalized tables, changes to data only need to be made in one place, simplifying updates. For example, an update to a product’s description in a normalized Products table will be reflected wherever that product is referenced through foreign keys, avoiding errors that could occur from updating redundant data in multiple places.

**Reduce storage requirements**: By eliminating redundant data, normalized databases require less storage. This makes them more efficient and less costly to scale, particularly important as data grows.

**Ensure Consistency**: In a normalized database, updates, deletions, and insertions can be more precisely managed, reducing errors. Consistency is guaranteed through constraints like primary keys, foreign keys, and unique constraints, which ensure that each piece of data adheres to its intended relationships.

**Adaptable Schema**: As the data model grows, normalized structures make it easier to add new entities, attributes, relationships, and constraints. This modularity allows you to scale the schema without risking data inconsistency or complexity.  The modular structure allows you to modify tables with minimal impact on other parts of the database.

**Targeted Indexing**: With unique data in each table, indexing becomes more effective. Indexes can be created on individual tables without being complicated by data duplication, speeding up query response times for searches, updates, and deletes.

**Reduced Anomalies**: Normalization reduces the three main types of anomalies: insertion, update, and deletion anomalies. For example, normalization prevents insertion anomalies where you’d need to create a “dummy” value just to satisfy database constraints, or update anomalies where you’d need to update data in multiple places to keep it consistent.

### When to Denormalize

While perfect normalization is an academic ideal with real-world benefits, it can also have real-world negative effects on an applications performance.  The primary cost of normalization is the need to JOIN multiple tables in a SQL query to get the data we want.

As an example, consider an e-commerce platform with three tables in a normalized relational database schema:

Products Table: Contains detailed information about each product.
Orders Table: Contains information specific to each order.
OrderItems Table: reference both the Orders and Products tables. Each line item in an order will have its own row in OrderItems, specifying the product and quantity.

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

### Exercise

What if we wanted simply to be able to total the value of orders from a particular customer in a particular date range?  We'd have to sum all the line items for all the orders belonging to a customer.  But we'd **still** have to use a 3-way join, because we need the ```Order``` table to filter by customer and date, and we need the ```OrderItems``` and ```Product``` entries to calculate the line item values to be summed.

Write a query that returns the total amount spent by a customer in the year 2023.

In the test database, how long does it take to execute the query for customer ```12345```?  How many rows does the query have to read?

### Hint

Here's a query that satisfies the exercise above.
```
SELECT 
    o.CustomerID,
    SUM(oi.Quantity * p.ProductPrice) AS TotalSpent
FROM 
    Orders o
JOIN 
    OrderItems oi ON o.OrderID = oi.OrderID
JOIN 
    Products p ON oi.ProductID = p.ProductID
WHERE 
    o.CustomerID = 12345  
    AND YEAR(o.OrderDate) = 2023
GROUP BY 
    o.CustomerID;
```

### Exercise

Given the schema and queries we 've 

### Exercise


This case illustrates a circumstance where we might want to denormalize the database to improve query performance.  Denormalizing here consists of duplicating data, or storing 

duplicate How could you hange the model to improve the performance of this query?



### Conclusion

denormalization is good for...

[Next Section](aost.md)

