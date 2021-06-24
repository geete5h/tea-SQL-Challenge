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
##### Method 1:
In the **Purchasing.SupplierTransactions** table, we can see we have SupplierInvoiceNumber and Transaction Amount. Once an invoice is issued to WWI, WWI pays back the amount after a few days, but by then, new invoices are issued by the supplier. So, we do not know which Invoices have been exactly paid off. I split the running total of Transaction Amount into *invoice column*  which is always incremented and *payment column* from which amount is always subtracted. Then keep moving down along the transactions until we come across the first running payment total equal to or greater than the invoice running total (when the invoice was issued). This way we get to the point till where WWI has made enough payments to cover an invoice. Then we get the count of invoices which ae paid and the count of inovices which are not paid, the amount and divide the amount by the count of paid invoices to get the average invoice amount.
```
WITH inv AS (
     SELECT SupplierInvoiceNumber, TransactionDate, SupplierID,
            SUM(TransactionAmount) OVER (
                PARTITION BY SupplierID
                ORDER BY TransactionDate, SupplierTransactionID
                ROWS UNBOUNDED PRECEDING) AS NetInvoicedAmount
     FROM Purchasing.SupplierTransactions
     WHERE TransactionAmount > 0),

paid AS (
     SELECT SupplierTransactionID, TransactionDate, SupplierID, -TransactionAmount AS Amount,
            -SUM(TransactionAmount) OVER (
                PARTITION BY SupplierID
                ORDER BY TransactionDate, SupplierTransactionID
                ROWS UNBOUNDED PRECEDING) AS NetPaidAmount
     FROM Purchasing.SupplierTransactions
     WHERE TransactionAmount < 0),

-- Getting the Days taken to make an invoice payment
DTP AS (SELECT inv.SupplierID,inv.SupplierInvoiceNumber,
        inv.TransactionDate AS InvoiceDate,
        paid.TransactionDate AS PaymentDate,
        DATEDIFF(day, inv.TransactionDate, paid.TransactionDate) AS DaysToPayment
FROM inv
INNER JOIN paid ON inv.SupplierID=paid.SupplierID
WHERE inv.NetInvoicedAmount<=paid.NetPaidAmount AND
      inv.NetInvoicedAmount>paid.NetPaidAmount-paid.Amount),

 -- List of Unpaid Invoices
NotPaid AS (SELECT Purchasing.SupplierTransactions.TransactionDate,Purchasing.SupplierTransactions.SupplierInvoiceNumber, Purchasing.SupplierTransactions.SupplierID 
FROM Purchasing.SupplierTransactions
WHERE Purchasing.SupplierTransactions.SupplierInvoiceNumber NOT IN (SELECT DTP.SupplierInvoiceNumber FROM DTP)),

-- List of Paid Invoices
PaidInv AS (SELECT Purchasing.SupplierTransactions.TransactionDate,Purchasing.SupplierTransactions.SupplierInvoiceNumber, Purchasing.SupplierTransactions.SupplierID 
FROM Purchasing.SupplierTransactions
WHERE Purchasing.SupplierTransactions.SupplierInvoiceNumber IN (SELECT DTP.SupplierInvoiceNumber FROM DTP)),

-- Average Invoice Amount
AvgInv AS (SELECT inv.SupplierID,
ROUND(MAX(inv.NetInvoicedAmount)/COUNT(inv.SupplierID),2) AS [AvgInvAmt]
FROM inv
GROUP BY inv.SupplierID),

-- Count of Invoices which are paid
CntPaid AS (SELECT PaidInv.SupplierID, COUNT(PaidInv.SupplierID) as [CountPaidInvoices]
FROM PaidInv
GROUP BY PaidInv.SupplierID),

-- Count of Invoices which are not paid
CntNotPaid AS (SELECT NotPaid.SupplierID, COUNT(NotPaid.SupplierID) as [CountNotPaidInvoices]
FROM NotPaid
GROUP BY NotPaid.SupplierID)

-- Final Query
SELECT Purchasing.Suppliers.SupplierName AS [Supplier Name],
CntPaid.CountPaidInvoices AS [Count of Invoices Paid],
ISNULL(CntNotPaid.CountNotPaidInvoices,0) as [Count of Invoices Not Paid],
AvgInv.AvgInvAmt AS [Average Amount Per Invoice]
FROM Purchasing.Suppliers
JOIN CntPaid ON CntPaid.SupplierID = Purchasing.Suppliers.SupplierID
FULL JOIN CntNotPaid ON CntNotPaid.SupplierID = Purchasing.Suppliers.SupplierID
JOIN AvgInv ON AvgInv.SupplierID = Purchasing.Suppliers.SupplierID
```

![image](https://user-images.githubusercontent.com/33748024/123334640-0299fc00-d509-11eb-84ef-077a8d01daa9.png)


##### Method 2:
Based on the Outstanding Balance Column, we can get the Suppliers, who have not been paid.
But this does not directly answer which invoices have been paid, so I'd prefer to go by  Method 1, if we want to know which invoices have been actually paid off.
In this method, if the Outstanding Balance is 0.0, then the Supplier has been paid, but if the Outstanding Balance is greater than 0.0, then the Supplier needs to be paid.

```
SELECT Purchasing.Suppliers.SupplierName AS [Supplier Name],
CntPaid.[Paid Invoices] AS [Count of Invoices Paid],
ISNULL(CntNotPaid.[Unpaid Invoices],0) as [Count of Invoices Not Paid],
AvgInv.[Average Invoice Amount] AS [Average Amount Per Invoice]
FROM Purchasing.Suppliers
JOIN (SELECT SupplierID, COUNT(SupplierID) AS [Paid Invoices]
FROM Purchasing.SupplierTransactions
WHERE OutstandingBalance = 0.0
GROUP BY SupplierID) CntPaid ON CntPaid.SupplierID = Purchasing.Suppliers.SupplierID
FULL JOIN (SELECT SupplierID, COUNT(SupplierID) AS [Unpaid Invoices]
FROM Purchasing.SupplierTransactions
WHERE OutstandingBalance > 0.0
GROUP BY SupplierID) CntNotPaid ON CntNotPaid.SupplierID = Purchasing.Suppliers.SupplierID
JOIN(SELECT SupplierID, (SUM(TransactionAmount)/COUNT(SupplierID)) AS [Average Invoice Amount]
FROM Purchasing.SupplierTransactions
WHERE SupplierInvoiceNumber IS NOT NULL
GROUP BY SupplierID) AvgInv ON AvgInv.SupplierID = Purchasing.Suppliers.SupplierID
```

![image](https://user-images.githubusercontent.com/33748024/123335509-2e69b180-d50a-11eb-9770-02fcc338bf78.png)



### Challenge 4
Using "unit price" and "recommended retail price", which item in the warehouse has the lowest gross profit amount? Which item has the highest? What is the median gross profit across all items in the warehouse?
#### Solution:
