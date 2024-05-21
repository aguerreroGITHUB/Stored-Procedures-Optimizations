# Optimizing SQL Stored Procedures
Optimizing SQL stored procedures is essential to improve database performance and ensure application efficiency. Here are some practical tips for achieving the most optimized stored procedures possible.

## 1. Memory Usage
Efficient use of memory is critical. Depending on data volume and query usage, different approaches should be considered during the development of the procedures: including views, CTEs (Common Table Expressions) or temporary tables.

- Views: Ideal for queries reused in multiple stored procedures with small to medium data volumes.
- CTEs: Best for temporary and less complex queries which will only exist in one procedure.
- Temporary Tables (#Table): The usual right choice for high data volumes, allowing the server to operate at maximum speed without running out of memory. (Once the operations with views and ctes reach the maximum allocated memory of the server, the speed will drastically decrease until the end of the query).
```SQL
/*Original Example*/
SELECT *
FROM dbo.View1 v1
  INNER JOIN dbo.View2 v2
    ON v1.PK = v2.PK
  
```
```SQL
/*Optimized Version*/
SELECT *
INTO #V1
FROM dbo.View1

SELECT *
INTO #V2
FROM dbo.View2

SELECT *
FROM #V1
  INNER JOIN #V2
    ON #V1.PK = #V2.PK

DROP TABLE #V1;
DROP TABLE #V2;
```
This would be a big optimization if the views read from tables with millions of rows. The server would have to have a lot of allocated memory in order not to be saturated with the original operation.

## 2. Views that Read from Views
The general advice is to avoid creating views that read from other views. If necessary, use the query of the base view and modify it to create the new view. Allowing this to happen is only recommended when the human readability of the code scales and performance is still good.
```SQL
/*Original Example*/
CREATE VIEW BaseView AS
  SELECT Table1.ID, MAX(Table1.Field2) Field2
  FROM Table1
    INNER JOIN Table2
      ON Table1.ID = Table2.ID
  GROUP BY Table1.ID ;

CREATE VIEW SecondView AS
  SELECT BaseView.*
  FROM BaseView
      LEFT JOIN Table3
        ON BaseView.ID = Table3.ID
  WHERE Table3.ID is null


```
```SQL
/*Optimized Version*/
CREATE VIEW BaseView AS
  SELECT Table1.ID, MAX(Table1.Field2)
  FROM Table1
    INNER JOIN Table2
      ON Table1.ID = Table2.ID
  GROUP BY Table1.ID ;

CREATE VIEW SecondView AS
  SELECT Table1.ID, MAX(Table1.Field2) Field2
  FROM Table1
    INNER JOIN Table2
      ON Table1.ID = Table2.ID
    LEFT JOIN Table3
      ON Table1.ID = Table3.ID
  WHERE Table3.ID is null
  GROUP BY Table1.ID, Table3.ID;
```
## 3. Clauses (IF EXISTS) in Select
To avoid duplicate joins and handle aggregations, use OUTER APPLY with a minimum aggregation query of the table and see if the crossover is null.
```SQL
/*Original Example*/
SELECT 
    e.EmployeeID,
    e.FirstName,
    e.LastName,
    CASE 
        WHEN EXISTS (
            SELECT 1
            FROM Sales s
            WHERE s.EmployeeID = e.EmployeeID
        ) THEN 'Yes'
        ELSE 'No'
    END AS HasSales,
    (SELECT COUNT(*)
     FROM Sales s
     WHERE s.EmployeeID = e.EmployeeID) AS SalesCount
FROM 
    Employees e;
```
```SQL
/*Optimized Version*/
SELECT 
    e.EmployeeID,
    e.FirstName,
    e.LastName,
    ISNULL(s.HasSales, 'No') AS HasSales,
    ISNULL(s.SalesCount, 0) AS SalesCount
FROM 
    Employees e
OUTER APPLY (
    SELECT 
        'Yes' AS HasSales,
        COUNT(*) AS SalesCount
    FROM Sales s
    WHERE s.EmployeeID = e.EmployeeID
    GROUP BY s.EmployeeID
) s;
```
## 4. Order of Joins
The joins are executed from top to bottom. First, place the INNER JOIN (the more rows you filter, the higher up) and then the LEFT JOIN.
```SQL
/*Original Example*/
SELECT 
    o.OrderID,
    c.CustomerName,
    od.ProductID,
    p.ProductName,
    s.SupplierName,
    sh.ShipmentDate
FROM Orders o
  LEFT JOIN Suppliers s
    ON  p.SupplierID = s.SupplierID
  INNER JOIN OrderDetails od
    ON  o.OrderID = od.OrderID
  INNER JOIN Customers c
    ON  o.CustomerID = c.CustomerID
    AND c.Region = 'North America'
  INNER JOIN Products p
    ON  od.ProductID = p.ProductID
  LEFT JOIN Shipments sh
    ON  o.OrderID = sh.OrderID;

```
```SQL
/*Optimized Version*/
SELECT 
    o.OrderID,
    c.CustomerName,
    od.ProductID,
    p.ProductName,
    s.SupplierName,
    sh.ShipmentDate
FROM Orders o
  INNER JOIN Customers c
    ON  o.CustomerID = c.CustomerID
    AND c.Region = 'North America'
  INNER JOIN OrderDetails od
    ON  o.OrderID = od.OrderID
  INNER JOIN Products p
    ON  od.ProductID = p.ProductID
  LEFT JOIN Suppliers s
    ON  p.SupplierID = s.SupplierID
  LEFT JOIN Shipments sh
    ON  o.OrderID = sh.OrderID;
```
See how in this example, the INNER JOIN to Customers will reduce the amount of rows in the execution, so next joins will be faster.

## 5. Tables without PK or Indexes
If a table has no PK or indexes, it will be difficult to make efficient queries. Make sure that all tables have a PK defined and, if they are not going to have one, design a functional index.

Try this by yourself by removing the PKs of some sample databases like Adventureworks, times will increase so hard.

## 6. Avoid Select *
Avoid using SELECT * in your queries. Specifying fields manually allows the SQL engine to better optimize queries and reduces unnecessary resource consumption. As a general practice, create views and procedures by specifying the necessary fields.

_______

Implementing these tips can make a significant difference in the performance of your stored procedures. Adjusting the approach according to data volume and query usage is key to effective optimization.
