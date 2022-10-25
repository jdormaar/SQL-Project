# What are your risk areas? Identify and describe them.
## risks:
EXAMPLES OF POTENTIAL CLEANING RISKS:
1. RISK: Deletions of empty columns.
---
2. RISK: Would deleting the column transaction_revenue result in a loss of data from the all_sessions table:
---



# QA Process:
Describe your QA process and include the SQL queries used to execute it.

---

1. Probably the easiest example of QA validation: Deletion of empty column.  While sequentially checking tables for data issues:
```sql
SELECT search_keyword FROM all_sessions WHERE search_keyword IS NOT NULL
```
This quick investigative query returned no data, the quick validation check used to confirm the absence of an unnoticed human entry error:
```sql
SELECT search_keyword FROM all_sessions WHERE search_keyword IS NULL
```
responded as expected with all 15134 rows from the all_sessions table represented as a null value, confirming the column was safe to be deleted.

---

2. Risk assessment for transaction_revenue data deletion: When the query below was run for an overview of potentially relevant data towards analyzing a geospacial relationship with transaction revenue data:
```sql
SELECT
    city
  , country
  , transaction_id
  , transaction_revenue
  , total_transaction_revenue
  , transactions
  , COUNT (city) AS counts
FROM all_sessions
GROUP BY 1, 2, 3, 4, 5, 6
ORDER BY counts DESC NULLS LAST;
```
 Two values of transaction_revenue were observed adjacent numerically equivalent total_transaction_revenue values, followed subsequently by a string of null values.  

 This column appears to redundant, and an easy way to manage that many null values would be to eliminate the potentially irrelevant column from teh data set entirely. However; before deleting the transaction_revenue column I needed to validate the data's apparent redundancy first to ensure these two observed values were not actually anomalous: 
 ```sql
 SELECT total_transaction_revenue, transaction_revenue, COUNT(transaction_revenue)
FROM all_sessions
WHERE transaction_revenue IS NOT NULL
GROUP BY 1, 2;
```
This query proves the redundancy of the column data within transaction_revenue by calling up all the non-null values along with their associated total_transaction_revenue values.  Since the output of this query returned only four unique transaction_revenue values, next to all four numerically equivalent values in the adjacent total_transaction_revenue column, we knew we could not possibly derive any additional information from those four transaction_revenue values which were not already preserved within the total_transaction_revenue column values.  The transaction_revenue column data can therefore be safely deleted from the all_sessions data table.

---
