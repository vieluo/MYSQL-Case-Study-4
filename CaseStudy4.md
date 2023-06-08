# 8 Weeks SQL Challenge - 4th Week

### This [case study](https://8weeksqlchallenge.com/case-study-4/)  is provided by Danny Ma

## Introduction
There is a new innovation in the financial industry called Neo-Banks: new aged digital only banks without physical branches.

Danny thought that there should be some sort of intersection between these new age banks, cryptocurrency and the data world…so he decides to launch a new initiative - Data Bank!

Data Bank runs just like any other digital bank - but it isn’t only for banking activities, they also have the world’s most secure distributed data storage platform!

Customers are allocated cloud data storage limits which are directly linked to how much money they have in their accounts. There are a few interesting caveats that go with this business model, and this is where the Data Bank team need your help!

The management team at Data Bank want to increase their total customer base - but also need some help tracking just how much data storage their customers will need.

This case study is all about calculating metrics, growth and helping the business analyse their data in a smart way to better forecast and plan for their future developments!

## Case Study Questions
* Customer Nodes Exploration
* Customer Transactions
* Data Allocation Challenge

## Create tables
Access the complete Schema [Here](https://8weeksqlchallenge.com/case-study-4/)

#

## Question - Customer Nodes Exploration
1. How many unique nodes are there on the Data Bank system?

```sql
SELECT COUNT(DISTINCT node_id) AS num_of_node
FROM customer_nodes
```

|num_of_node|
| - | 
| 5 | 

2. What is the number of nodes per region?

```sql
SELECT region_id, COUNT(node_id) AS num_of_node
FROM customer_nodes
GROUP BY region_id
ORDER BY region_id
```

| region_id |num_of_node|
| - | - | 
| 1 | 770 |
| 2 | 735 |
| 3 | 714 |
| 4 | 665 |
| 5 | 616 |

3. How many customers are allocated to each region?

```sql
SELECT region_id, COUNT(DISTINCT customer_id) AS num_of_customer
FROM customer_nodes
GROUP BY region_id
```

| region_id |num_of_customer
| - | - | 
| 1 | 110 |
| 2 | 105 |
| 3 | 102 |
| 4 | 95 |
| 5 | 88 |

4. How many days on average are customers reallocated to a different node?

```sql
SELECT ROUND(AVG(datediff(end_date,start_date) +1)) average_duration
WHERE end_date < '2021-01-01';
```
|average_duration|
| - | 
| 16 | 

5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?


For different region, the median number, 80, 95 percentil. should be: 

```sql
SELECT region_id, COUNT(*)/2 AS median_per_region, COUNT(*)/5*4 AS eightieth_per_region, COUNT(*)/20*19 AS ninety_fifth_per_region
FROM temp
GROUP BY region_id
```

For region 1, the median number, 80, 95 percentil are: 
```sql
WITH temp1 AS (
WITH temp AS (
SELECT (datediff(end_date,start_date) +1) days, region_id
FROM customer_nodes
WHERE end_date < '2021-01-01'AND region_id = 1 /* Organize the rows in order to generate the number of row with correct order */
)
SELECT *, ROW_NUMBER() over () AS row_num /* I created a new column for row number, with ROW_NUMBER() I can number the rows and take the row of medin, 80, 95 percentil. */
FROM temp
),

/* If the row number is odd, we can use it directly, if not, we need to use the average value of (row_num and row_num+1) */
median1 AS (
SELECT *
FROM temp1
WHERE row_num = 330
),
median2 AS (
SELECT *
FROM temp1
WHERE row_num = 331
),
eightieth1 AS (
SELECT *
FROM temp1
WHERE row_num = 528
),
eightieth2 AS (
SELECT *
FROM temp1
WHERE row_num = 529
),
ninety_fifth AS (
SELECT *
FROM temp1
WHERE row_num = 627
),
consolidated AS (
SELECT median1.region_id, median1.days AS median1_days, median2.days AS median2_days, eightieth1.days AS eightieth1_days, 
eightieth2.days AS eightieth2_days, ninety_fifth.days AS ninety_fifth_days
FROM median1 JOIN median2
ON median1.region_id = median2.region_id
JOIN eightieth1
ON median1.region_id = eightieth1.region_id
JOIN eightieth2
ON median1.region_id = eightieth2.region_id
JOIN ninety_fifth
ON median1.region_id = ninety_fifth.region_id
)
SELECT region_id, ROUND((median1_days + median2_days)/2,0) AS median, ROUND((eightieth1_days + eightieth2_days)/2,0) AS eightieth, ninety_fifth_days AS ninety_fifth
FROM consolidated
```

| region_id | median | eightieth | ninety_fifth
| - | - | - | - | 
| 1 | 16 | 24 | 29 |

Change the region number to 2,3,4,5 and put the corresponding row number, we can get the rest of median, 80, 95 percentil. 

| region_id | median | eightieth | ninety_fifth
| - | - | - | - | 
| 2 | 16 | 24 | 29 |
| 3 | 16 | 25 | 29 |
| 4 | 16 | 24 | 29 |
| 5 | 16 | 25 | 29 |

## Question - Customer Transactions

1. What is the unique count and total amount for each transaction type?

```sql
SELECT txn_type, COUNT(DISTINCT(customer_id)) AS unique_count, SUM(txn_amount) AS total_amount
FROM customer_transactions
GROUP BY txn_type
```

| txn_type | unique_count | total_amount
|------------|-----|---------|
| deposit    | 500 | 1359168 |
| purchase   | 448 | 806537  |
| withdrawal | 439 | 793003  | 

2. What is the average total historical deposit counts and amounts for all customers?

```sql
SELECT ROUND(COUNT(txn_type)/COUNT(DISTINCT(customer_id)),2) AS deposit_counts, ROUND(AVG(txn_amount),2) AS avg_amount_deposit
FROM customer_transactions
WHERE txn_type = 'deposit'
```
| deposit_counts | unavg_amount_depositique_count 
| - | - |
| 5.34 | 508.86

3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

I created 3 temp table to calculate the movement times of each user in each month. Then I join these 3 tables together in order to get a consolidated table with all the information. 

```sql
WITH count_deposit_by_month AS (
SELECT customer_id, month, COUNT(Txn_type) AS count_deposit
FROM (
SELECT *, MONTH(txn_date) AS month
FROM customer_transactions
) AS temp
WHERE txn_type = 'deposit'
GROUP BY customer_id, month
),

count_withdrawal_by_month AS (
SELECT customer_id, month, COUNT(Txn_type) AS count_withdrawal
FROM (
SELECT *, MONTH(txn_date) AS month
FROM customer_transactions
) AS temp
WHERE txn_type = 'withdrawal'
GROUP BY customer_id, month
),

count_purchase_by_month AS (
SELECT customer_id, month, COUNT(Txn_type) AS count_purchase
FROM (
SELECT *, MONTH(txn_date) AS month
FROM customer_transactions
) AS temp
WHERE txn_type = 'purchase'
GROUP BY customer_id, month
)

SELECT count_deposit_by_month.customer_id, count_deposit_by_month.month, count_deposit_by_month.count_deposit, 
count_withdrawal_by_month.count_withdrawal, count_purchase_by_month. count_purchase
FROM count_deposit_by_month
LEFT JOIN count_withdrawal_by_month ON
count_deposit_by_month.customer_id = count_withdrawal_by_month.customer_id
LEFT JOIN count_purchase_by_month ON
count_deposit_by_month.customer_id = count_purchase_by_month.customer_id
```

Then I calculate the nunmber of customer filtering their movement time, deposit > 1, withdrawal or purchase = 1

```sql
SELECT month, COUNT(DISTINCT(customer_id)) AS count_customer
FROM consolidated
WHERE count_deposit > 1 AND (count_withdrawal =1 OR count_purchase =1 )
GROUP BY month
ORDER BY month
```

| month | count_customer |
|---|-----|
| 1 | 214 | 
| 2 | 192 | 
| 3 | 200 | 
| 4 | 82  | 

4. What is the closing balance for each customer at the end of the month?

First I calculate separately the total deposit and total spend in each month for each customer, then I join the temp tables together in order to calcuate the final balance. 

```sql
WITH total_deposit AS (
SELECT customer_id, month(txn_date) AS month, SUM(txn_amount) AS sum_deposit
FROM customer_transactions
WHERE txn_type = 'deposit'
GROUP BY customer_id, month
),

total_spend AS (
SELECT customer_id, month(txn_date) AS month, SUM(txn_amount) AS sum_spend
FROM customer_transactions
WHERE txn_type = 'withdrawal' OR txn_type ='purchase'
GROUP BY customer_id, month
),

movement AS (
SELECT  total_deposit.customer_id, total_deposit.month, total_deposit.sum_deposit, CASE WHEN
sum_spend IS NULL THEN 0
ELSE sum_spend END AS sum_spend /* If I dont set the null number to 0, when calculating the final balance, a value minus a null value will return a null value, this will case a null balance in the end */ 
FROM total_deposit
LEFT JOIN total_spend
ON  total_deposit.customer_id = total_spend.customer_id AND total_spend.month = total_deposit.month
UNION ALL
SELECT total_spend.customer_id, total_spend.month, CASE WHEN
sum_deposit IS NULL THEN 0
ELSE sum_deposit END AS sum_deposit,
sum_spend
FROM total_spend
LEFT JOIN total_deposit
ON  total_deposit.customer_id = total_spend.customer_id AND total_spend.month = total_deposit.month
WHERE sum_deposit IS NULL /* In order to dont repeat the rows that already contain in the first join query */
)

SELECT customer_id, month, sum_deposit - sum_spend AS final_balance
FROM movement
ORDER BY customer_id, month

```
Here I will show the first 6 customers´ balance

| customer_id | month | final_balance |  
|---|---|--------|
| 1 | 1 | 312    |
| 1 | 3 | -952   |
| 2 | 1 | 549    |
| 2 | 3 | 61     |
| 3 | 1 | 144    |
| 3 | 2 | -965   |
| 3 | 3 | -401   |
| 3 | 4 | 493    |
| 4 | 1 | 848    |
| 4 | 3 | -193   |
| 5 | 1 | 954    |
| 5 | 3 | -2877  |
| 5 | 4 | -490   |
| 6 | 1 | 733    |
| 6 | 2 | -785   |
| 6 | 3 | 392    |

5. What is the percentage of customers who increase their closing balance by more than 5%?

With COUNT(DISTINCT customer_id), we can know that totally we have 500 distinct customer in the transaction table.

Using the table we solved the last quesition, as month_balance. I take the first month and last month balance of customer, then calculate the percentage of increase balalance.

```sql
first_and_final_balance AS (
SELECT DISTINCT customer_id, FIRST_VALUE(final_balance) OVER (PARTITION BY customer_id) AS first_balance,
LAST_VALUE(final_balance) OVER (PARTITION BY customer_id order by month range between unbounded preceding and unbounded following) AS final_balance
FROM month_balance
),

percentage AS (
SELECT *, ABS((final_balance-first_balance)/first_balance*100) AS percentage /* Here I used Absolute function because some users have negative balance in the end and in the beginning, but however, the negative value deduce, so we can still consider it as increase in balance. */
FROM first_and_final_balance
WHERE final_balance > first_balance
)

SELECT ROUND(COUNT(customer_id)/500*100,2) AS increase_five_percent_more
FROM percentage
WHERE percentage > 5
```

| increase_five_percent_more | 
| - | 
| 33.20 | 

## Question - Data Allocation Challenge

To test out a few different hypotheses - the Data Bank team wants to run an experiment where different groups of customers would be allocated data using 3 different options:

* Option 1: data is allocated based of the amount of money at the end of the previous month
* Option 2: data is allocated on the average amount of money kept in the account in the previous 30 days
* Option 3: data is updated real-time
For this multi-part challenge question - you have been requested to generate the following data elements to help the Data Bank team estimate how much data will need to be provisioned for each option:

* running customer balance column that includes the impact each transaction
* customer balance at the end of each month
* minimum, average and maximum values of the running balance for each customer
Using all of the data available - how much data would have been required for each option on a monthly basis?


### Running customer balance column that includes the impact each transaction

```sql
WITH temp1 AS (
SELECT customer_id, txn_date, txn_type,txn_amount,CASE 
WHEN txn_type = "purchase" OR txn_type = "withdrawal"
THEN txn_amount*-1
ELSE txn_amount
END AS actual_transaction
FROM customer_transactions
)
SELECT customer_id, txn_date, actual_transaction,
SUM(actual_transaction) OVER(PARTITION BY customer_id ORDER BY txn_date) AS balance
FROM temp1
ORDER BY customer_id, txn_date
```

|customer_id|txn_date|actual_transaction|balance|
|---|------------|------|--------|
| 1 | 2020-01-02 | 312  | 312    |
| 1 | 2020-03-05 | -612 | -300   |
| 1 | 2020-03-17 | 324  | 24     |
| 1 | 2020-03-19 | -664 | -640   |
| 2 | 2020-01-03 | 549  | 549    |
| 2 | 2020-03-24 | 61   | 610    |
| 3 | 2020-01-27 | 144  | 144    |
| 3 | 2020-02-22 | -965 | -821   |
| 3 | 2020-03-05 | -213 | -1034  |
| 3 | 2020-03-19 | -188 | -1222  |
| 3 | 2020-04-12 | 493  | -729   |
| 4 | 2020-01-07 | 458  | 458    |
| 4 | 2020-01-21 | 390  | 848    |
| 4 | 2020-03-25 | -193 | 655    |


### Customer balance at the end of each month
```sql
WITH temp1 AS (
SELECT customer_id, txn_date, txn_type,txn_amount,CASE 
WHEN txn_type = "purchase" OR txn_type = "withdrawal"
THEN txn_amount*-1
ELSE txn_amount
END AS actualactual_transaction
FROM customer_transactions
),
temp2 AS (
SELECT customer_id, txn_date, actual_transaction,
SUM(actual_transaction) OVER(PARTITION BY customer_id ORDER BY txn_date) AS balance
FROM temp1
), /* here I calculate the balance after wach transaction */
temp3 AS (SELECT *,MAX(txn_date) OVER(PARTITION BY customer_id, MONTH(txn_date)) AS max_date
FROM temp2
) /* here I filter out the last transaction date for each user each month */
SELECT customer_id, MONTHNAME(txn_date) AS month, balance
FROM temp3
WHERE txn_date = max_date
```
| customer_id | month  | balance    |
|---|----------|------|
| 1 | January  | 312    |
| 1 | March    | -640   |
| 2 | January  | 549    |
| 2 | March    | 610    |
| 3 | January  | 144    |
| 3 | February | -821   |
| 3 | March    | -1222  |
| 3 | April    | -729   |
| 4 | January  | 848    |
| 4 | March    | 655    |
| 5 | January  | 954    |
| 5 | March    | -1923  |
| 5 | April    | -2413  |


### Minimum, average and maximum values of the running balance for each customer
Using the script from the first quetion, we can calculate the min, avg, max values easily.

```sql
WITH temp1 AS (
SELECT customer_id, txn_date, txn_type,txn_amount,CASE 
WHEN txn_type = "purchase" OR txn_type = "withdrawal"
THEN txn_amount*-1
ELSE txn_amount
END AS actual_transaction
FROM customer_transactions
),
temp2 AS (SELECT customer_id, txn_date, actual_transaction,
SUM(actual_transaction) OVER(PARTITION BY customer_id ORDER BY txn_date) AS balance
FROM temp1
ORDER BY customer_id, txn_date
)
SELECT customer_id, COUNT(txn_date) AS transaction_times, MIN(balance) AS min_balance, MAX(balance) AS max_balance, ROUND(AVG(balance),2) AS avg_balance
FROM temp2
GROUP BY customer_id
```

|customer_id|transaction_times|min_balance|max_balance|avg_balance|
|----|----|-------|------|-----------|
| 1  | 4  | -640  | 312  | -151.00   |
| 2  | 2  | 549   | 610  | 579.50    |
| 3  | 5  | -1222 | 144  | -732.40   |
| 4  | 3  | 458   | 848  | 653.67    |
| 5  | 11 | -2413 | 1780 | -135.45   |
| 6  | 19 | -552  | 2197 | 624.00    |
| 7  | 13 | 887   | 3539 | 2268.69   |
| 8  | 10 | -1029 | 1363 | 173.70    |
| 9  | 10 | -91   | 2030 | 1021.70   |
| 10 | 18 | -5090 | 556  | -2229.83  |
| 11 | 17 | -2529 | 60   | -1950.82  |
| 12 | 4  | -647  | 295  | -14.50    |
| 13 | 13 | 379   | 1444 | 901.15    |
| 14 | 4  | 205   | 989  | 751.00    |
| 15 | 2  | 379   | 1102 | 740.50    |
