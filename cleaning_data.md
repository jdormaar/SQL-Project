# Cleaning Data

## Issues that will be addressed by cleaning the data? (6)


- Consistent naming conventions for tables and columns
- Identify irrelevant and duplicate data for removal
- Type conversion for improperly data types
- Syntax errors and typos
- Find missing values that should b filled in to complete the dataset
- Identify outliers for removal to prevent their interference during analysis
- Handle missing data
- Filter out data outliers
- Fix dates and times
- Merge and split columns
- Reconcile table data by adding relationships
- Validate data to ready it for data transformation and analysis
- Perform transformations and analysis

## Queries

Queries and description of process used to interrogate and clean the data.

### Consistent Naming Conventions

Renamed table columns for `all_sessions`, `analytics`, `products`, `sales_by_sku`, `sales_report` to use a common naming convention (snake_case) for tables to create readability between the separate words, see example.

```sql
-- Example of changing a column name to snake_case format

ALTER TABLE all_sessions
  RENAME fullVisitorId TO full_visitor_id;
```

### Remove Duplicate and Irrelevant Data

```sql
-- Investigation into the potentially irrelevant data within transaction columns

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

```sql
-- Verify transaction_revenue is irrelevant

SELECT total_transaction_revenue, transaction_revenue, COUNT(transaction_revenue)
FROM all_sessions
WHERE transaction_revenue IS NOT NULL
GROUP BY 1, 2;
```

Output from the initial query drove the comparison of `total_transaction_revenue` to `transaction_revenue`, and helped identify that `transaction_revenue` had only 4 rows with non-null values, which turned out to be duplicates of `total_transaction_revenue` and lead to a clean up task to remove `transaction_revenue` as a column.

```sql
-- Remove transaction_revenue from all_session as it has 4 values that are not null, and
-- those values match total_transaction_revenue making it duplicate and irrelevant data.

ALTER TABLE all_sessions
  DROP COLUMN transaction_revenue;
```

---

```sql
-- Check for rows that have a value in item_quantity

SELECT item_quantity
FROM all_sessions
WHERE item_quantity IS NOT NULL;

--- Result: 0 rows

-- Remove item_quantity column from the dataset

ALTER TABLE all_sessions
  DROP COLUMN item_quantity;
```

---

```sql
-- Select city and country where city is null

SELECT
    city
  , country
  , COUNT (city) AS count_col
FROM public.all_sessions
GROUP BY city, country
ORDER BY city DESC NULLS FIRST;

-- Result: 447 rows, no `NULL` values.
```

The result did show that there existed quite a few country and city values that should be updated to null.

```SQL
-- Update city to be null when 'not available in demo dataset'

UPDATE all_sessions
   SET city = NULL
WHERE city = 'not available in demo dataset';

-- Result: UPDATE 8302
```

```SQL
-- Update country to be null when '(not set)'

UPDATE all_sessions
   SET country = NULL
WHERE country = '(not set)';

-- Result: UPDATE 24
```

---

### Fix Structural Errors

Not fully performed due to time constraint.

### Handle Missing Data

### Filter Out Data Outliers

### Validate Data

### Reconcile table data by adding relationships

```sql
-- Create unique primary key for each of the five tables in 
-- order to draw relationships

ALTER TABLE (public.all_sessions)
ADD COLUMN id SERIAL PRIMARY KEY;

ALTER TABLE (public.sales_by_sku)
ADD COLUMN id SERIAL PRIMARY KEY;

ALTER TABLE (public.analytics)
ADD COLUMN id SERIAL PRIMARY KEY;

ALTER TABLE (public.sales_report)
ADD COLUMN id SERIAL PRIMARY KEY;

-- SKU appears to be an appropriate primary key

ALTER TABLE (public.products)
ADD COLUMN id SERIAL PRIMARY KEY;
```

---
---

# Work in Progress

I thought to start initially with the all_sessions table, looking to the visitor id columns.

```sql
--sort all_sessions table sequentially by full_visitor_id. 
--screen for NULL values:
SELECT *
FROM public.all_sessions
ORDER BY full_visitor_id NULLS FIRST;
```
###### RETURNED 15134 rows,  no ```NULL``` values. #####  


Next, id numbers were checked for repetitions:

```sql
--display only unique full_visitor_id values from all_sessions:
SELECT DISTINCT(full_visitor_id)
FROM public.all_sessions
ORDER BY full_visitor_id;
```
###### RETURNED 14223 rows.
That's a lot of duplicate id numbers.  6% of the content of ```all_sessions.full_visitor_id``` values are not unique.  

To take a closer look at these duplications:
```sql
--group by the full_visitor_id column,
--display adjacent count column for full_visitor_id repetitions
--arrange rows by descending count values
SELECT full_visitor_id,
	COUNT (full_visitor_id) AS count_col
FROM public.all_sessions
GROUP BY full_visitor_id
ORDER BY count_col DESC NULLS FIRST;
```
###### RETURNED 14223 rows,  no `NULL` values.
There are numerous id duplications in this table, with no obvious pattern yet. Most frequent is 10 instances.

It is possible these id numbers are not actually intended to be unique here. I'll return to analyze this further later on to check the other rows related to those repetitions.  

Continuing to search for the unique identifier columns in other tables to help establish database relationships between the tables might also bring me back to `all_sessions.full_visitor_id`.

The columns that mention sku in `products.sku`, `sales_reports.product_sku`, `sales_by_sku.product_sku` were addressed next:

Full outer join enabled the contents between tables to be compared for inclusivity.  
> **In theory**;  the products table should likely contain the parent sku id list from which all other product sku values in other tables should reference;

 Therefore an outer join without data losses sorted on the product table's sku column, screened for NULL values should validate this:
```sql
--full outer join sales_report.product_sku to products.sku
--order by product's sku column
--screen for NULL values
SELECT p.sku, r.product_sku
FROM public.products AS p
	FULL OUTER JOIN public.sales_report AS r
	ON p.sku = r.product_sku
ORDER BY p.sku NULLS FIRST;
```
###### RETURNED 1092 rows, no `NULL` values.
Having confirmed the set of product.sku values fully envelop those listed within the reports table, this likely identifies the product table sku column is a reasonable starting point for assigning a primary key connection.  

Provided that all listed 1092 values of the product.sku set are unique: 

To maintain the comparative perspective while checking for unique values, I added the aggregate function `COUNT()` to the join query rather than `SELECT DISTINCT` 

```sql
--full outer join  with count on product.sku 
--grouped by product sku
--ordered descending on the counted grouped values
SELECT p.sku, r.product_sku, COUNT(r.product_sku) AS product_sku_repeats
FROM public.products AS p
	FULL OUTER JOIN public.sales_report AS r
	ON p.sku = r.product_sku
GROUP BY 1, 2
ORDER BY product_sku_repeats DESC;
```
###### RETURNED 1092 rows. No counted values exceed 1.

Great! Now that products.sku are all confirmed as unique, it is time to reassign the primary key constraint in the products table:

as per PostgreSQL 15.0 Documentation https://www.postgresql.org/docs/15/ddl-constraints.html (1.);
```sql
--primary key (pk) constraint reassignment:
--remove the current pk constraint from unique serial id 
ALTER TABLE public.products
DROP CONSTRAINT products_pkey; 

--rename column heading from sku to sku_id
ALTER TABLE public.products
RENAME COLUMN sku TO sku_id;

--assign new pk constraint
ALTER TABLE public.products ADD PRIMARY KEY (sku_id);
```


Logically it makes sense for the table of sales reports to have sku number duplications given the nature of selling the same product more than once, but it's also not possible for a sales sku number to connect to more than one value from the parent product list.  So a one to many relationship cardinality is indicated. 

Again, according to the current postgreSQL 15.0 manual (1.); a one to many relationship creation requires the parent primary key for which to reference, and assigning the foreign key constraint to the columns of the referencing table:

```sql
--assign foreign key to sales_report.product_sku
ALTER TABLE public.sales_report 
ADD CONSTRAINT distfk FOREIGN KEY (product_sku) 
REFERENCES public.products (sku_id);
```
Generating an entity relationship diagram (ERD) to check the connection between the products table and the sales_report table, and the appropriate cardinality confirmed this was successful.

Now to compare the product Table sku_id with the sales_by_sku table: 

--SELECT FROM (FULL OUTER JOIN)
```sql
--full outer join sales_by_sku.product_sku to products.sku_id
--order by product's sku_id column
--screen for NULL values
SELECT p.sku_id, s.product_sku
FROM public.products AS p
	FULL OUTER JOIN public.sales_by_sku AS s
	ON p.sku_id = s.product_sku
ORDER BY p.sku_id NULLS FIRST;
```
###### RETURNED 1100 rows, 8 null values. 

There are 8 product_sku values in sales_by_sku not represented within the parent list of products.sku_id values.

To learn more about these mystery sku numbers, this query was nested into a right join with it's own (sales_by_sku) table again, to acquire the sales data for which these mystery numbers are associated:

>To first isolate the numbers that are each matched to a null value, the outer join query was nested for simplification while retaining queried data.

```sql
--single column output of sales_by_sku.product_sku associated with NULL products.sku_id values.
SELECT sp.product_sku 
FROM
	(SELECT p.sku_id, s.product_sku
	FROM public.products AS p
		FULL OUTER JOIN public.sales_by_sku AS s
		ON p.sku_id = s.product_sku
	ORDER BY p.sku_id NULLS FIRST) AS sp
WHERE sp.sku_id IS NULL;
```
###### RETURNED 8 rows.

> Which was then further nested back into the `sales_by_sku` table:
```sql
--SELECT only the values denoted by "missing.product_sku" values,
--from a RIGHT JOIN which returns single column output of the sales_by_sku.product_sku values not within the parent pk products.sku_id
SELECT s.product_sku, s.total_ordered, missing.product_sku
FROM public.sales_by_sku AS s
	RIGHT JOIN 
		(SELECT sp.product_sku 
		FROM
			(SELECT p.sku_id, s.product_sku
			FROM public.products AS p
				FULL OUTER JOIN public.sales_by_sku AS s
				ON p.sku_id = s.product_sku
			ORDER BY p.sku_id NULLS FIRST) AS sp
		WHERE sp.sku_id IS NULL) AS missing
	ON s.product_sku = missing.product_sku
ORDER BY missing.product_sku;
```
###### RETURNED 8 rows, 6 of the total_ordered values are 0.

The sales data returned doesn't appear to provide much additional information yet, but does present more questions.  75% of the `total_ordered` values, which correspond to `NULL` `product_sku_id values`, are 0.   

* Which stimulates the question: what is the meaning of the data input instances within this `sales_by_sku` table, to which apparently orphaned (or nonexistent) `product.sku_id` values were recorded, but not sold.

It is likely that the rows without quantities ordered are of no use to the `sales_by_sku` table since they don't offer any further data enrichment beyond the fact that these sku numbers exist (probably).  I've place these these 8 rows on a list to the side, for all pending deletions to make and leave them for now just in case a reason not delete them has yet to be revealed.  Since we know these numbers wont be in the sales_reports table already, I think this is unlikely. but perhaps the answer to their mystery is within the all_sessions or analytics tables.

To how the product pk sku_id list compares to the product_sku in all_sessions:

--OUTER JOIN ON SKU ORDER BY LEFT
```sql
--full outer join on products.sku_id to all_sessions.product_sku
--order by product's sku_id column
--screen for NULL values
SELECT p.sku_id, al.product_sku
FROM public.products AS p
	FULL OUTER JOIN public.all_sessions AS al
	ON p.sku_id = al.product_sku
ORDER BY p.sku_id NULLS FIRST;
```
###### RETURNED 15837 rows, many `NULL` values.

So many nulls. 

To check the inverse,

```sql
SELECT p.sku_id, al.product_sku
FROM public.products AS p
	FULL OUTER JOIN public.all_sessions AS al
	ON p.sku_id = al.product_sku
ORDER BY al.product_sku NULLS FIRST;
```
###### RETURNED 15837 rows, many `NULL` values.

Still so many nulls... 

For how many null values are in the all_sessions table, but not the products table:

```sql
SELECT lost.sku_id, COUNT(*)
FROM
	(SELECT p.sku_id, al.product_sku
	FROM public.products AS p
		FULL OUTER JOIN public.all_sessions AS al
		ON p.sku_id = al.product_sku
	ORDER BY al.product_sku NULLS FIRST) AS lost
WHERE lost.sku_id IS NULL
GROUP BY 1;
```
|  |sku_id |count |
|--|-------|------|
|1 |[NULL] |  2033|

2033 of the all sku numbers are not listed in the products sku table.

Not `NULL` sku values:
```sql
SELECT COUNT(product_sku IS NOT NULL)
FROM public.all_sessions
```
|  |count |
|--|------|
|1 | 15134|

15134 total number of listed sku values in the all_sessions tbl.
```sql
SELECT lost.product_sku, COUNT(*)
FROM
	(SELECT p.sku_id, al.product_sku
	FROM public.products AS p
		FULL OUTER JOIN public.all_sessions AS al
		ON p.sku_id = al.product_sku
	ORDER BY al.product_sku NULLS FIRST) AS lost
WHERE lost.product_sku IS NULL
GROUP BY 1;
```
This return tells us that 703 of the 1092 unique vales in prod_id are not represented in all_sessions

```sql
--SELECT COUNT (INNER JOIN)
SELECT COUNT(p.sku_id)AS count, al.product_sku
FROM public.products AS p
	INNER JOIN public.all_sessions AS al
	ON p.sku_id = al.product_sku
GROUP BY al.product_sku
ORDER BY count DESC;
```

Counting the duplicated sku values in all_sessions of the inner join tables confirms this mathematically as the output number of rows is 

>389 + 703 above = 1092 unique values in products.sku_id.

More information is needed to understand the source and reasons for this much unmatched values.

until them one might consider relating these 2 tables using a one or none to many or none relationship cardinality.
# References:
1. https://www.postgresql.org/docs/15/index.html, **PostgreSQL 15.0 Documentation**,
*The PostgreSQL Global Development Group*, 1996–2022 The PostgreSQL Global Development Group.

2. https://learn.microsoft.com/en-us/sql/?view=sql-server-ver16, **Microsoft SQL documentation**, *Educational SQL resources* ,Microsoft 2022.

3. https://www.techonthenet.com/sql/index.php, **SQL Tutorial**, *totn SQL*, 2003-2022 TechOnTheNet.com.

4. https://www.tutorialspoint.com/sql/index.htm, **SQL Tutorial**, *LEARN SQL simply easy learning*, 2022 Tutorialspoint.com.

5. https://www.w3schools.com/sql/default.asp, **SQL Syntax**, *W3schools*, 1999-2022 by Refsnes Data.

6. https://towardsdatascience.com/the-ultimate-guide-to-data-cleaning-3969843991d4#b498, **The Ultimate Guide to Data Cleaning - when the data is spewing garbage**, *Towards Data Science*, Feb 28, 2019. Omar Elgabry.