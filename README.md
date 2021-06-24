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
