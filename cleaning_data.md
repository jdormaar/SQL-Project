# Cleaning Data

## "What issues will you address by cleaning the data?"

* 

Queries:
Below, provide the SQL queries you used to clean your data.

Repeated for each of the 5 tables to facilitate initial comparisons for beginning to to understand the data:
## Cleaning the Database:
In order to get started, I felt like I needed each table to have a unique id to set temporarily as the table's primary key, in order to facilitate the initial comparative observations. 

```sql
--create unique primary key for each of the five tables

ALTER TABLE (public.all_sessions)
ADD COLUMN id SERIAL PRIMARY KEY;

ALTER TABLE (public.sales_by_sku)
ADD COLUMN id SERIAL PRIMARY KEY;

ALTER TABLE (public.products)
ADD COLUMN id SERIAL PRIMARY KEY;

ALTER TABLE (public.analytics)
ADD COLUMN id SERIAL PRIMARY KEY;

ALTER TABLE (public.sales_report)
ADD COLUMN id SERIAL PRIMARY KEY;
```

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
1. https://www.postgresql.org/docs/15/index.html **PostgreSQL 15.0 Documentation**,
*The PostgreSQL Global Development Group*, 1996–2022 The PostgreSQL Global Development Group.