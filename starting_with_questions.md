Answer the following questions and provide the SQL queries used to find the answer.

## Question 1 
Which cities and countries have the highest level of transaction revenues on the site?

### SQL Queries:

```sql
-- Sum transaction levels by country
SELECT
    country
  , SUM (ROUND(total_transaction_revenue)) AS ttr
FROM sessions
WHERE total_transaction_revenue IS NOT NULL
GROUP BY country
ORDER BY ttr DESC;
```

```sql
-- Sum transaction levels by city
SELECT
    city
  , country
  , SUM(total_transaction_revenue) AS ttr
FROM sessions
WHERE total_transaction_revenue IS NOT NULL
AND city IS NOT NULL
GROUP BY city, country
ORDER BY ttr DESC;
```
### Answer:

Countries with highest level of transactions:
| country       | total_transaction_revenue |
| ------------- | ------------------------- |
| United States | 13153                     |
| Israel        | 602                       |
| Australia     | 358                       |
| Canada        | 150                       |
| Switzerland   | 17                        |

Cities with the highest level of transactions:

| city          | country       | total_transaction_revenue |
| ------------- | ------------- | ------------------------- |
| San Francisco | United States | 1564.32                   |
| Sunnyvale     | United States | 992.23                    |
| Atlanta       | United States | 854.44                    |
| Palo Alto     | United States | 608.00                    |
| Tel Aviv-Yafo | Israel        | 602.00                    |
| New York      | United States | 530.36                    |
| Mountain View | United States | 483.36                    |
| Los Angeles   | United States | 479.48                    |
| Chicago       | United States | 449.52                    |
| Sydney        | Australia     | 358.00                    |
| Seattle       | United States | 358.00                    |
| San Jose      | United States | 262.38                    |
| Austin        | United States | 157.78                    |
| Nashville     | United States | 157.00                    |
| San Bruno     | United States | 103.77                    |
| Toronto       | Canada        | 82.16                     |
| New York      | Canada        | 67.99                     |
| Houston       | United States | 38.98                     |
| Columbus      | United States | 21.99                     |
| Zurich        | Switzerland   | 16.99                     |
## Question 2
What is the average number of products ordered from visitors in each city and country?

### SQL Queries:

```sql
-- Average number of products ordered by country
SELECT
    country
  , ROUND(AVG(product_quantity)) AS country_quantity
FROM sessions
WHERE product_quantity IS NOT NULL
GROUP BY country
ORDER BY country_quantity DESC;
```

```sql
-- Average number of products ordered by city
SELECT
    city
  , country
  , ROUND(AVG(product_quantity)) AS city_quantity
FROM sessions
WHERE product_quantity IS NOT NULL
  AND city IS NOT NULL
GROUP BY city, country
ORDER BY city_quantity DESC;
```
### Answer:

Average number of products ordered by country:

| country       | country_quantity |
| ------------- | ---------------- |
| Spain         | 10               |
| United States | 4                |
| Colombia      | 1                |
| Finland       | 1                |
| France        | 1                |
| Argentina     | 1                |
| Ireland       | 1                |
| Mexico        | 1                |
| India         | 1                |
| Canada        | 1                |

Average number of products ordered by city:

| city          | country       | city_quantity |
| ------------- | ------------- | ------------- |
| Madrid        | Spain         | 10            |
| Salem         | United States | 8             |
| Atlanta       | United States | 4             |
| Houston       | United States | 2             |
| Dallas        | United States | 1             |
| Detroit       | United States | 1             |
| Dublin        | Ireland       | 1             |
| Los Angeles   | United States | 1             |
| Mountain View | United States | 1             |
| New York      | United States | 1             |
| Palo Alto     | United States | 1             |
| San Francisco | United States | 1             |
| San Jose      | United States | 1             |
| Seattle       | United States | 1             |
| Ann Arbor     | United States | 1             |
| Sunnyvale     | United States | 1             |
| Bengaluru     | India         | 1             |
| Chicago       | United States | 1             |
| Columbus      | United States | 1             |

## Question 3

Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?
### SQL Queries:

```sql
SELECT
    city
  , country
  , v2_product_category
  , v2_product_name
FROM sessions
WHERE product_quantity IS NOT NULL
  AND city IS NOT NULL
GROUP BY city, country, v2_product_category, v2_product_name
ORDER BY city DESC;
```

### Answer:

## Question 4 

What is the top-selling product from each city/country? Can we find any pattern worthy of noting in the products sold?
### SQL Queries:

```sql
```

### Answer:

## Question 5 

Can we summarize the impact of revenue generated from each city/country?
### SQL Queries:

```sql
```
### Answer: