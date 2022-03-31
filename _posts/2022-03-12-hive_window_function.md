---
layout: post
title: "Hive window functions"
subtitle: 'A handy sql tool for data scientists'
category: SQL
tags: [hive, sql, code]
---

## Introduction

Window functions are summary operations over a window or range of values (called Frame) on a 
table. For each input row, these functions can take values from a specified number of rows 
from the current row, and return a single output corresponding to current row.

Lets start with a simple example. Lets say we have a table with `id` and `price`. 
We want to find cumulative sum (or running total) of these prices. To get cumulative sum, 
we add current value of  `price` with the summation of all previous values of `price`. 
The final table is:

Table 1: Example of cumulative sum

|  id   | price | cumulative price |
| :---: | :---: | :--------------: |
|   a   |   1   |        1         |
|   b   |   3   |        4         |
|   c   |   4   |        8         |

Here, we effectively used window function to do this. In Table 1, window is defined as all previous 
values from the current row (inclusive). These functions can be very handy to find 
values like:

* Moving averages
* Percentage contribution
* Percentile of a value
* Find values that occur before the current row

The window functions make SQL queries syntactically cleaner, easier to read and maintain. 
The same task could be done by some complex queries, e.g., joining multiple queries. Window functions 
allow us to avoid those complex multiple joins, so we can avoid storing intermediate temporary 
tables.

A simplified query syntax with window clause:

```sql
SELECT <column(s)>, <window function>(column)
OVER (<window_definition>)
FROM <table>
```

All major SQL engines provides window functions. Here, I focused on hive window functions. The 
following are the window functions that hive provides. 

* Standard aggregation window functions:
    - AVG
    - SUM
    - MAX
    - MIN
    - COUNT
* Built-in windowing functions:
	- FIRST_VALUE
	- LAST_VALUE
	- LEAD
	- LAG
* Analytics functions
	- RANK
	- ROW_NUMBER
	- DENSE_DIST
	- PERCENT_RANK
	- NTILE

The partitioning window is defined by the `OVER` clause. Every window function must have a `OVER` 
clause. For a given row, it defines which rows will be in the same partition. 
The window definition has the following parts:

* `PARTITION BY`: partition the whole table specified by this column values.	
* `ORDER BY`: Specified the Order of column(s) either Ascending or Descending.
* Frame: Specified the boundary of the frame by stat and end value. 

`OVER` clause uses one or more of these parts. It can also be used without any parts. In such cases,
there will be no partition or order and Frame will take the default window selection.

**Default frame:**

* If the `ORDER BY` clause is specified, then the frame is:  
    `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

* Otherwise:
    `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

It's time to set up an example hive table and see how window functions help us find answers for 
particular types of questions.

## Example dataset
Let create a hive table "daily_sales" that keeps sales records each items sold in a grocery store 
chain. The following columns are available:

* id: transaction id
* store: store number
* product: product sold
* date: transaction date
* sales: sale amount

```sql
CREATE DATABASE IF NOT EXISTS ds;

USE ds;

CREATE TABLE daily_sales (
    id string,
    store string,
    product string,
    sales_date date, 
    sales DOUBLE) ROW format delimited fields terminated BY ',';
```

Insert these rows into the new table and check the table values

```sql
INSERT INTO TABLE daily_sales
VALUES ("101", "store_1", "Bread", "2022-01-01", "2"),
       ("102", "store_1", "Milk", "2022-01-01", "3"),
       ("103", "store_1", "Bread", "2022-01-01", "2"),
       ("201", "store_2", "Bread", "2022-01-01", "3"),
       ("202", "store_2", "Cheese", "2022-01-01", "5"),
       ("301", "store_3", "Milk", "2022-01-03", "3"),
       ("104", "store_1", "Water", "2022-01-02", "6"),
       ("302", "store_3", "Water", "2022-01-03", "7"),
       ("303", "store_3", "Bread", "2022-01-03", "2"),
       ("203", "store_2", "Water", "2022-01-03", "5");

SELECT * FROM daily_sales;
```
```text
OK
id      store   product sales_date      sales
101     store_1 Bread   2022-01-01      2.0
102     store_1 Milk    2022-01-01      3.0
103     store_1 Bread   2022-01-01      2.0
201     store_2 Bread   2022-01-01      3.0
202     store_2 Cheese  2022-01-01      5.0
301     store_3 Milk    2022-01-03      3.0
104     store_1 Water   2022-01-02      6.0
302     store_3 Water   2022-01-03      7.0
303     store_3 Bread   2022-01-03      2.0
203     store_2 Water   2022-01-03      5.0
```

Now we have the table ready, let's try out window functions.

## Standard aggregation

Aggregation functions: SUM, AVG, COUNT, MAX, MIN

Lets start with a simple window query, just with aggregation function `SUM` and no window/partition/
order specification for `OVER` :
```sql
SELECT id,
       sales_date,
       sales,
       SUM(sales) OVER ( ) AS total
FROM   daily_sales; 
```

```text
OK
id      sales_date      sales   total
101     2022-01-01      2.0     38.0
102     2022-01-01      3.0     38.0
103     2022-01-01      2.0     38.0
201     2022-01-01      3.0     38.0
202     2022-01-01      5.0     38.0
301     2022-01-03      3.0     38.0
104     2022-01-02      6.0     38.0
302     2022-01-03      7.0     38.0
303     2022-01-03      2.0     38.0
203     2022-01-03      5.0     38.0
```
Hmm! It did find the sum of sales, but each row shows total sales. This is because we didn’t 
partition or order the data. Adding other windows parts would give more useful information.

### For entire table

**Objective : Find cumulative sum / running total of sales.**
```sql
SELECT id,
       sales_date,
       sales,
       SUM(sales) OVER (
           ORDER BY id ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW 
        ) AS running_total
FROM   daily_sales; 
```
```text
OK
id      sales_date      sales   running_total
101     2022-01-01      2.0     2.0
102     2022-01-01      3.0     5.0
103     2022-01-01      2.0     7.0
104     2022-01-02      6.0     13.0
201     2022-01-01      3.0     16.0
202     2022-01-01      5.0     21.0
203     2022-01-03      5.0     26.0
301     2022-01-03      3.0     29.0
302     2022-01-03      7.0     36.0
303     2022-01-03      2.0     38.0
```
Here, Frame is defined as "from the beginning of the table
(`unbounded preceding`) to  `current row`". This is, in fact, the default window Frame, since
 `ORDER BY` clause is specified. We can omit this Frame definition to get the exact query output.

This window function `ORDER BY id` sort the rows by `id` first, then calculate running total by 
adding current sales value with all previous sales values.


**Objective: Find running average of sales.**
```sql
SELECT id,
       sales_date,
       sales,
       AVG(sales) OVER (
           ORDER BY id
        ) AS running_avg
FROM   daily_sales; 
```
```text
OK
id      sales_date      sales   running_avg
101     2022-01-01      2.0     2.0
102     2022-01-01      3.0     2.5
103     2022-01-01      2.0     2.3333333333333335
104     2022-01-02      6.0     3.25
201     2022-01-01      3.0     3.2
202     2022-01-01      5.0     3.5
203     2022-01-03      5.0     3.7142857142857144
301     2022-01-03      3.0     3.625
302     2022-01-03      7.0     4.0
303     2022-01-03      2.0     3.8
```
This is similar to the running total. Here, we are calculating the running average. In each row,
it calculates the average of all the previous sales from current row, inclusive. For example, 
for `id = 102`, output is `{(2.0 + 3.0) / 2} = 2.5` and for `id = 103`, output is 
`{(2.0 + 3.0 + 2. 0) / 3} = 2.33`. Note that we didn't use any window Frame, so the default
window Frame is used here.

**Objective: Find running maximum of sales.**
```sql
SELECT id,
       sales_date,
       sales,
       MAX(sales) OVER (
           ORDER BY id
        ) AS running_max
FROM   daily_sales;
```
```text
OK
id      sales_date      sales   running_max
101     2022-01-01      2.0     2.0
102     2022-01-01      3.0     3.0
103     2022-01-01      2.0     3.0
104     2022-01-02      6.0     6.0
201     2022-01-01      3.0     6.0
202     2022-01-01      5.0     6.0
203     2022-01-03      5.0     6.0
301     2022-01-03      3.0     6.0
302     2022-01-03      7.0     7.0
303     2022-01-03      2.0     7.0
```
With default window Frame, each row can see current and previous values. So when we want to find
running maximum value, it will find maximum value from current to previous values.


### For each partition

We can partition/group the table by a column and apply window function for each partition. This is
done by using `PARTITION BY` with `OVER` clause.
For example, we can apply window functions for each day. When a new day comes up, the counting
restarts.

**Objective : Find running average of sales for each day.**

```sql
SELECT id,
       sales_date,
       sales,
       Sum(sales)
         OVER (
           PARTITION BY sales_date
           ORDER BY id ) AS running_total
FROM   daily_sales; 
```
```text
OK
id      sales_date      sales   running_total
101     2022-01-01      2.0     2.0
102     2022-01-01      3.0     5.0
103     2022-01-01      2.0     7.0
201     2022-01-01      3.0     10.0
202     2022-01-01      5.0     15.0
104     2022-01-02      6.0     6.0
203     2022-01-03      5.0     5.0
301     2022-01-03      3.0     8.0
302     2022-01-03      7.0     15.0
303     2022-01-03      2.0     17.0
```
Without the `ORDER BY` clause, we are not doing ordering within a partition, then the default 
row specification becomes:
`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`
we will get one value per day 

```sql
SELECT id,
       sales_date,
       sales,
       Sum(sales)
         OVER (
           PARTITION BY sales_date ) AS running_total
FROM   daily_sales; 
```
```text
OK
id      sales_date      sales   running_total
101     2022-01-01      2.0     15.0
102     2022-01-01      3.0     15.0
103     2022-01-01      2.0     15.0
201     2022-01-01      3.0     15.0
202     2022-01-01      5.0     15.0
104     2022-01-02      6.0     6.0
301     2022-01-03      3.0     17.0
302     2022-01-03      7.0     17.0
303     2022-01-03      2.0     17.0
203     2022-01-03      5.0     17.0
```
Since there is no `ORDER BY` clause, it will count all the rows for each day. So, we get the total 
count for each day. And since this is window function, we get output for each id. This is redundant.
This can be useful, however, if we want to find total count for each day. 
We can use the `DISTINCT` keyword to get total `COUNT` for each day.
```sql
SELECT DISTINCT sales_date,
                COUNT(sales)
                  OVER (
                    PARTITION BY sales_date ) AS number_sales
FROM   daily_sales;
```
```
OK
sales_date      number_sales
2022-01-01      5
2022-01-02      1
2022-01-03      4
```

### Moving total/average
In moving total/average, the window moves relative to the current row. For example, we want to 
calculate the total of the last three items that were sold at any point in time. This is done by
defining the window frame as `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`.

```sql
SELECT id,
       sales_date,
       sales,
       SUM(sales)
         over (
           ORDER BY id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS
       moving_total
FROM   daily_sales; 
```
```text
OK
id      sales_date      sales   moving_total
101     2022-01-01      2.0     2.0
102     2022-01-01      3.0     5.0
103     2022-01-01      2.0     7.0
104     2022-01-02      6.0     11.0
201     2022-01-01      3.0     11.0
202     2022-01-01      5.0     14.0
203     2022-01-03      5.0     13.0
301     2022-01-03      3.0     13.0
302     2022-01-03      7.0     15.0
303     2022-01-03      2.0     12.0
```
Look as `id` 103 and 104. For `id` 103, moving total is calculated by adding `sales` for `id` 103,
which is 2 and two previous `sales`, giving (2 + 3 + 2 = 7). For `id` 104, the window changes. 
Now the window is row for `id` 104 and the previous two rows. So, the moving total for `id` is 
(6 + 2 + 3 = 11). 

Same way, we can get moving average. But this time, we will do that with `PARTITION BY` clause. We
want to find moving averages, partitioned by each day.
```sql
 SELECT id,
       sales_date,
       sales,
       AVG(sales)
         OVER (
           PARTITION BY sales_date
           ORDER BY id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW ) AS moving_avg
FROM   daily_sales; 
```
```text
OK
id      sales_date      sales   moving_avg
101     2022-01-01      2.0     2.0
102     2022-01-01      3.0     2.5
103     2022-01-01      2.0     2.3333333333333335
201     2022-01-01      3.0     2.6666666666666665
202     2022-01-01      5.0     3.3333333333333335
104     2022-01-02      6.0     6.0
203     2022-01-03      5.0     5.0
301     2022-01-03      3.0     4.0
302     2022-01-03      7.0     5.0
303     2022-01-03      2.0     4.0
```

### Percent contribution
**Objective:** How much a particular order has contributed towards the total sales. It can be done for the entire dataset or for each day partition.
```sql
SELECT id,
       sales_date,
       sales,
       sales * 100 / SUM(sales)
            OVER ( ) AS percent_contr
FROM   daily_sales; 
```
```text
OK
id      sales_date      sales   percent_contr
101     2022-01-01      2.0     5.2631578947368425
102     2022-01-01      3.0     7.894736842105263
103     2022-01-01      2.0     5.2631578947368425
201     2022-01-01      3.0     7.894736842105263
202     2022-01-01      5.0     13.157894736842104
301     2022-01-03      3.0     7.894736842105263
104     2022-01-02      6.0     15.789473684210526
302     2022-01-03      7.0     18.42105263157895
303     2022-01-03      2.0     5.2631578947368425
203     2022-01-03      5.0     13.157894736842104
```
Notice that we didn't use `ORDER BY`. So, entire rows are selected as frame. Hence, `SUM(sales)` would calculate total sales. It then multiply individual sales with total sales times 100 to give us percentage contribution for each row.

## Built-in functions
### FIRST_VALUE
It will return the first record from each partition. For example, We want to find the first date of transaction for each `store`. Window function returns output for each row. We need to use `DISTINCT` to get only one value for each store.
```sql
SELECT DISTINCT store,
                FIRST_VALUE(sales_date)
                  OVER(
                    PARTITION BY store) AS first_sale
FROM   daily_sales; 
```
```text
OK
store   first_sale
store_1 2022-01-01
store_2 2022-01-01
store_3 2022-01-03
```

We can partition the data by more than one column. We can find the first `product` sold in each `store`, for each `sales_date`. For that, we need to partition the data both by `store` and `sales_date`
```sql
SELECT DISTINCT store,
                sales_date,
                FIRST_VALUE(product)
                  OVER(
                    PARTITION BY store, sales_date) AS first_sale
FROM   daily_sales; 
```
```text
OK
store   sales_date      first_sale
store_1 2022-01-01      Bread
store_1 2022-01-02      Water
store_2 2022-01-01      Bread
store_2 2022-01-03      Water
store_3 2022-01-03      Milk
```
We can add `ORDER BY`. By default, it will order the data in ascending order, i.e., `FIRST_VALUE` will show the minimum value for each partition. To get the maximum value, we need to order the partition in descending order by `DESC`. For example, We want to find the maximum sales price for each product. 
```sql
SELECT DISTINCT product,
                FIRST_VALUE(sales)
                  OVER(
                    PARTITION BY product
                    ORDER BY sales DESC) AS max_price
FROM   daily_sales; 
```
```text
OK
product max_price
Bread   3.0
Cheese  5.0
Milk    3.0
Water   7.0
```



## Analytics functions
### ROW_NUMBER
This window function assigns a unique increment number to every record specified within a window. 

At this point, we add one row, which has same `id` 102. Lets say, the customer bought two
product in one `id`. So, this is basically a duplicate row.
```sql
INSERT INTO daily_sales
VALUES      
("102", "store_1", "milk", "2022-01-01", "3"); 
```
Now we run the row_number window function.
```sql
SELECT id,
       sales_date,
       sales,
       ROW_NUMBER()
         OVER (
           PARTITION BY sales_date
           ORDER BY id ) AS row_number
FROM   daily_sales; 
```
```text
OK
id      sales_date      sales   row_number
101     2022-01-01      2.0     1
102     2022-01-01      3.0     2
102     2022-01-01      3.0     3
103     2022-01-01      2.0     4
201     2022-01-01      3.0     5
202     2022-01-01      5.0     6
104     2022-01-02      6.0     1
203     2022-01-03      5.0     1
301     2022-01-03      3.0     2
302     2022-01-03      7.0     3
303     2022-01-03      2.0     4
```
The table is partitioned by `sales_date` and we see the row number within each date. Notice that we get different row number for duplicated row (`id` 102).

### RANK
If we want to get same row number for duplicated row, we need to use `RANK` function.
```sql
SELECT id,
       sales_date,
       sales,
       RANK()
         OVER (
           PARTITION BY sales_date
           ORDER BY id ) AS rank
FROM   daily_sales; 
```
```text
OK
id      sales_date      sales   rank
101     2022-01-01      2.0     1
102     2022-01-01      3.0     2
102     2022-01-01      3.0     2
103     2022-01-01      2.0     4
201     2022-01-01      3.0     5
202     2022-01-01      5.0     6
104     2022-01-02      6.0     1
203     2022-01-03      5.0     1
301     2022-01-03      3.0     2
302     2022-01-03      7.0     3
303     2022-01-03      2.0     4
```
Now the duplicate rows have the row number.

### NTILE
This is to calculate what percentile a row falls into. Say, we want to divide the `sales` into four parts. Then with `NTILE` we can see which part an individual entry falls into.
```sql
SELECT id,
       sales_date,
       sales,
       NTILE(4)
         OVER (
           ORDER BY sales ) AS quantile
FROM   daily_sales; 
```
```text
OK
id      sales_date      sales   quantile
101     2022-01-01      2.0     1
103     2022-01-01      2.0     1
303     2022-01-03      2.0     1
102     2022-01-01      3.0     2
201     2022-01-01      3.0     2
301     2022-01-03      3.0     2
102     2022-01-01      3.0     3
202     2022-01-01      5.0     3
203     2022-01-03      5.0     3
104     2022-01-02      6.0     4
302     2022-01-03      7.0     4
```