# tea-SQL-Challenge

### Challenge 1
The leadership team has asked us to graph total monthly sales over time. Write a query that returns the data we need to complete this request.
#### Solution:

```
SELECT
CONVERT(varchar(7), (Sales.Invoices.InvoiceDate), 126) AS [Invoice Year-Month],
SUM(Sales.InvoiceLines.ExtendedPrice) AS [Revenue]
FROM Sales.Invoices
INNER JOIN Sales.InvoiceLines ON Sales.InvoiceLines.InvoiceID = Sales.Invoices.InvoiceID
GROUP BY CONVERT(varchar(7), (Sales.Invoices.InvoiceDate), 126)
ORDER BY [Invoice Year-Month];
```
Running the above query yeilds the following result.

![image](https://user-images.githubusercontent.com/33748024/123332225-d8930a80-d505-11eb-9aad-6458d87acb8a.png)

![image](https://user-images.githubusercontent.com/33748024/123332444-1f810000-d506-11eb-946c-5d53d13c5e0b.png)


### Challenge 2
What is the fastest growing customer category in Q1 2016 (compared to same quarter sales in the previous year)? What is the growth rate?
#### Solution:
As the Financial Year starts in November for WWI, I calculated the Revenue from November 1 - January 31 for both Q1 2015 and Q2 2016 grouping by Customer Category Name using the query:
```
SELECT A.CustomerCategoryName AS [Customer Category Name], 
A.Revenue AS [Q1 2015 Revenue], 
B.Revenue AS [Q1 2016 Revenue],
((B.Revenue-A.Revenue)/A.Revenue)*100 AS [Growth Rate]
FROM (SELECT 
Sales.CustomerCategories.CustomerCategoryName AS CustomerCategoryName,
SUM(Sales.InvoiceLines.ExtendedPrice) AS Revenue
FROM Sales.Invoices 
INNER JOIN Sales.InvoiceLines ON Sales.InvoiceLines.InvoiceID = Sales.Invoices.InvoiceID
INNER JOIN Sales.Customers ON Sales.Customers.CustomerID = Sales.Invoices.CustomerID
INNER JOIN Sales.CustomerCategories ON Sales.CustomerCategories.CustomerCategoryID = Sales.Customers.CustomerCategoryID
WHERE (Sales.Invoices.InvoiceDate BETWEEN '2014-11-01' and '2015-01-31') 
GROUP BY Sales.CustomerCategories.CustomerCategoryName) A
 JOIN
 (SELECT 
Sales.CustomerCategories.CustomerCategoryName AS CustomerCategoryName,
SUM(Sales.InvoiceLines.ExtendedPrice) AS Revenue
FROM Sales.Invoices 
INNER JOIN Sales.InvoiceLines ON Sales.InvoiceLines.InvoiceID = Sales.Invoices.InvoiceID
INNER JOIN Sales.Customers ON Sales.Customers.CustomerID = Sales.Invoices.CustomerID
INNER JOIN Sales.CustomerCategories ON Sales.CustomerCategories.CustomerCategoryID = Sales.Customers.CustomerCategoryID
WHERE (Sales.Invoices.InvoiceDate BETWEEN '2015-11-01' and '2016-01-31') 
GROUP BY Sales.CustomerCategories.CustomerCategoryName) B
ON A.CustomerCategoryName = B.CustomerCategoryName
```
The query returns the following result:

![image](https://user-images.githubusercontent.com/33748024/123333198-0b89ce00-d507-11eb-99cb-f65e3b8bf8db.png)

It can be seen that the fastest growing category is the **Gift Store** with a Growth rate of **4.8%**


### Challenge 3
Write a query to return the list of suppliers that WWI has purchased from, along with # of invoices paid, # of invoices still outstanding, and average invoice amount.
#### Solution:
