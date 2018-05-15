
## Answers - Task 1
### Question 1
Which customer has made the most rentals at store 2? 
#### Answer
The customer with the most rentals at store 2 is:
```
 customer_id | first_name | last_name | num_rentals | store_id
-------------+------------+-----------+-------------+----------
         473 | Jorge      | Olivares  |          26 |        2
(1 row)
```
473	"Jorge"	"Olivares"	"26"	2
#### Query
```
-- Question 1: Which customer has made the most rentals at store 2? 
with rentals_per_customer
as (
	select customer_id, count(rental_id) num_rentals
	from rental join inventory using (inventory_id)
	where
		1=1
		AND store_id = 2
	group by customer_id
)
select customer_id, first_name, last_name, num_rentals, store_id
from rentals_per_customer join customer using (customer_id)
```
### Question 2
What percentage of movies were out of stock at each store on 29/7/2005 at 
midnight? 
#### Answer
Below you can find the two percentages corresponding to movies out of stock per store. Please note that in the query, we have used a different time-point (2006-02-16 00:00:00 instead of 2005-07-29 00:00:00, whic was requested), in order for the query to return results; since all the inventory table rows had the same last_update value, namely 2006-02-15 10:09:17.
```
 store_id | stock | mv_total | %movies_outof_stock
----------+-------+----------+---------------------
        1 |   759 |     1000 |               24.10
        2 |   762 |     1000 |               23.80
(2 rows)
```
#### Query
```
-- Question 2: What percentage of movies were out of stock at each store on 29/7/2005 at midnight? 
with movie_stock_at_tmpoint
as (
	select film_id, store_id, count(inventory_id) stock
	from inventory
	where last_update <= timestamp '2006-02-16 00:00:00' -- timestamp '2005-07-29 00:00:00'
	group by film_id, store_id
	order by 1, 2, 3 desc
),
movie_stock_per_store
as(
	select store_id, count(distinct film_id) stock
	from movie_stock_at_tmpoint
	group by store_id
),
movie_total
as (
	select count(film_id) mv_total 
	from film
)
select store_id, stock, mv_total, round((mv_total - stock)/mv_total::decimal * 100,2) "%movies_outof_stock"
from movie_stock_per_store, movie_total;
```
### Question 3
Is the employee that performs the rental of a DVD usually the one who also takes the Payment? 
#### Answer
It is the same employee in 45% percent of the rentals.
```
 the_same_empl_#rentals | diffrnt_empl_#rentals | tot_#rentals | %the_same_employee
------------------------+-----------------------+--------------+--------------------
                   7271 |                  7325 |        16044 |              45.32
(1 row)
```
#### Query
```
-- Question 3: Is the employee that performs the rental of a DVD usually the one who also takes the Payment? 
with rental_employees
as(
	select staff_id, rental_id
	from rental
),
payment_employees
as(
	select staff_id, rental_id, payment_id
	from payment
),
same_employee_rental_cnt
as(
	select count(t1.rental_id) num_rentals
	from rental_employees t1 join payment_employees t2 on (t1.rental_id = t2.rental_id)
	where t1.staff_id = t2.staff_id
	
),
diffrnt_employee_rental_cnt
as(
	select count(t1.rental_id) num_rentals
	from rental_employees t1 join payment_employees t2 on (t1.rental_id = t2.rental_id)
	where t1.staff_id != t2.staff_id	
),
tot_rentals
as(
	select count(rental_id) num_rentals
	from rental
)
select 	t1.num_rentals "the_same_empl_#rentals", t2.num_rentals "diffrnt_empl_#rentals", t3.num_rentals "tot_#rentals",
		round(t1.num_rentals/t3.num_rentals::decimal * 100,2) "%the_same_employee"
from  same_employee_rental_cnt t1, diffrnt_employee_rental_cnt t2, tot_rentals t3;
```
### Question 4
How many rentals do we do per month? 
#### Answer
```
  mnth   | num_rentals
---------+-------------
 2005/05 |        1156
 2005/06 |        2311
 2005/07 |        6709
 2005/08 |        5686
 2006/02 |         182
(5 rows)
```
#### Query
```
-- Question 4: How many rentals do we do per month? 
select to_char(rental_date, 'YYYY/MM') mnth, count(rental_id) num_rentals
from rental
group by to_char(rental_date, 'YYYY/MM')
order by 1;
```
### Question 5
What percentage of our customers are active at any given month? We define active as performing at least one rental during that month. 

#### Answer
The percent of active customers per month is shown in the following table:
```
  mnth   | cnt_distinct_customers | tot_customer | %active_customers
---------+------------------------+--------------+-------------------
 2005/05 |                    520 |          599 |             86.81
 2005/06 |                    590 |          599 |             98.50
 2005/07 |                    599 |          599 |            100.00
 2005/08 |                    599 |          599 |            100.00
 2006/02 |                    158 |          599 |             26.38
(5 rows)
```
#### Query
```
-- Question 5: What percentage of our customers are active at any given month? 
-- We define active as performing at least one rental during that month.
with active_customers_per_month
as(
	select to_char(rental_date, 'YYYY/MM') mnth, count(distinct customer_id) cnt_distinct_customers
	from rental
	group by to_char(rental_date, 'YYYY/MM')
),
tot_customers
as(
	select count(customer_id) tot_customer
	from customer
)
select mnth, cnt_distinct_customers, tot_customer, round(cnt_distinct_customers/tot_customer::decimal *100 , 2) "%active_customers"
from active_customers_per_month, tot_customers
order by 1;
```
### Question 6
Are there some films that are particularly popular and are rented all the time, or do people tend to spread their choice evenly among the available films? 
#### Answer
People don't spread their choices among films evenly. This is clearly depicted in the following histogram, where we have broken down the number of rentals per film in 10 distinct ranges (buckets). There we can see that there are 16 very popular films that fall into the range of  [31,34] rentals. Also, there are 86 "unpopular" films that fall into the range [4,7] rentals. The majority of the films have a rental score between 11 to 23 number of rentals in the time period that we have examined.
```
  bucket | range: num of rentals per film | freq: #films in this range |              bar
--------+--------------------------------+----------------------------+--------------------------------
      1 | [4,8)                          |                         86 | =============
      2 | [8,11)                         |                        118 | ==================
      3 | [11,14)                        |                        127 | ===================
      4 | [14,18)                        |                        201 | ==============================
      5 | [18,21)                        |                        131 | ====================
      6 | [21,24)                        |                        127 | ===================
      7 | [24,28)                        |                        110 | ================
      8 | [28,31)                        |                         42 | ======
      9 | [31,34)                        |                         15 | ==
     10 | [34,35)                        |                          1 |
(10 rows)
```
#### Query
```
-- Question 6: Are there some films that are particularly popular and are rented all the time, 
-- or do people tend to spread their choice evenly among the available films? 
with film_stats 
as(
	select film_id, count(rental_id) cnt_rental
	from rental join inventory using(inventory_id)
	group by film_id
	order by 2 desc
),
minmax 
as(
    select min(cnt_rental) as min,
           max(cnt_rental) as max
      from film_stats
),
histogram as (
	select width_bucket(cnt_rental, min, max, 9) as bucket,
		  int4range(min(cnt_rental)::int, max(cnt_rental)::int, '[]') as range,
		  count(1) as freq
	 from minmax, film_stats
	group by bucket
	order by bucket
)
select bucket, range as "range: num of rentals per film", freq as "freq: #films in this range",
	repeat('=',
		   (   freq::float
			 / max(freq) over()
			 * 30
		   )::int
	) as bar
from histogram;
```
### Question 7
 Which film category is the most popular among our customers? 
 ### Answer
 It is the Sports category, as it is shown by the following result
 ```
  category | cnt_rental
----------+------------
 Sports   |       1179
(1 row)
```
### Query
```
-- Question 7:  Which film category is the most popular among our customers? 
select name as category, count(rental_id) cnt_rental
from rental join inventory using(inventory_id) 
				join film_category using(film_id) 
					join category using (category_id)
group by name
order by 2 desc
limit 1;
```
### Question 8
Are there any other insights that you can gather from the data that would be helpful to the owner of this business? 
####  Answer
The data available are invaluable for building several *customer profiles* for serving different business use-cases (e.g., cross-sell, up-sell, acquisition, retention).  In our example, we have chosen to build a customer profile in order to find "disappointed" customers, i.e., customers that have a larger possibility to churn. We want to retain these customers, especially if these are "valuable" customers. 
To this end, we have calculated a core set of customer behavioral indicators that indicate a possible disappointed customer:

 - *Low rental rate indicator*:  customers whose number of rentals is decreasing, or is increasing very slowly in the last 3 months
 - *Too old last rental indicator*:  customers with a last rental before two months
 -  *Active customer indicator*: An active customer is one that has at least one rental in the last two months
 - *Customer tenure*: total days that a customer has been an active customer. In our case, is the number of days from the oldest rental till now (applicable only to active customers).

Our query returns a list of active customers with a large tenure (i.e., loyal customers) and thus valuable to the business, who have at least one of the above "disappointment indicators" turned on.
Of course, a much more enhanced approach would be to feed these indicators (and many others) to a customer churn model, that it would be trained to predict customer churn based on these predictors,
Here are the results of the query:
```
 customer_id | low_rental_rate_ind | max_rentals_diff | last_rental_too_old_ind |  last_rental_date   | low_avg_rentals_per_month_id | avg_num_rentals | active_ind |      tenure       
-------------+---------------------+------------------+-------------------------+---------------------+------------------------------+-----------------+------------+-------------------
          73 |                   0 |               10 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.8 |          1 | 266 days 13:44:37
         100 |                   0 |               13 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.8 |          1 | 266 days 13:33:21
         227 |                   0 |               12 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.8 |          1 | 266 days 05:14:41
          43 |                   0 |               12 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.8 |          1 | 266 days 03:33:18
          94 |                   0 |                7 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.2 |          1 | 266 days 02:49:20
         505 |                   0 |                6 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.2 |          1 | 265 days 22:25:32
          33 |                   0 |                8 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.2 |          1 | 265 days 21:31:24
         548 |                   0 |                6 |                       0 | 2006-02-14 15:16:03 |                            1 |             3.8 |          1 | 265 days 19:45:31
         412 |                   0 |                7 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.2 |          1 | 265 days 17:45:54
         394 |                   0 |               10 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.4 |          1 | 265 days 15:15:52
         162 |                   0 |                7 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.0 |          1 | 265 days 04:18:20
         534 |                   0 |                7 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.8 |          1 | 265 days 02:38:32
         496 |                   0 |                8 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.2 |          1 | 264 days 23:12:25
           9 |                   0 |                9 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.6 |          1 | 264 days 18:58:32
         152 |                   0 |                6 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.2 |          1 | 264 days 17:11:27
          22 |                   0 |                7 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.4 |          1 | 264 days 16:10:17
         525 |                   0 |               11 |                       0 | 2006-02-14 15:16:03 |                            1 |             3.8 |          1 | 264 days 06:12:38
         557 |                   0 |               10 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.8 |          1 | 264 days 02:49:57
         476 |                   0 |                9 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.4 |          1 | 263 days 23:50:48
         315 |                   0 |                8 |                       0 | 2006-02-14 15:16:03 |                            1 |             3.4 |          1 | 263 days 17:39:05
         493 |                   0 |                8 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.6 |          1 | 263 days 17:16:26
         252 |                   0 |               10 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.4 |          1 | 262 days 20:41:41
         352 |                   0 |                9 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.6 |          1 | 262 days 09:15:38
         530 |                   0 |                9 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.6 |          1 | 261 days 22:24:45
          99 |                   0 |               15 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.8 |          1 | 261 days 20:05:17
         192 |                   0 |               10 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.6 |          1 | 261 days 15:09:17
          11 |                   0 |               11 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.8 |          1 | 261 days 01:00:48
         216 |                   0 |                6 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.6 |          1 | 260 days 23:51:35
         355 |                   0 |               11 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.0 |          1 | 260 days 08:37:09
         180 |                   0 |               10 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.6 |          1 | 260 days 07:20:27
         431 |                   0 |                5 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.6 |          1 | 260 days 06:32:15
         191 |                   0 |                9 |                       0 | 2006-02-14 15:16:03 |                            1 |             4.0 |          1 | 260 days 04:45:45
(32 rows)

```
#### Query
```
-- Question 8: Are there any other insights that you can gather from the data 
-- that would be helpful to the owner of this business? 

-- build the profile of each customer in order to predict churn or disappointed customers	
	-- Low rental rate indicator: find customers whose number of rentals is decreasing, or is increasing very slowly in the last 3 months
with
customer_monthly_rentals
as(
	select 	to_char(rental_date, 'YYYY/MM') mnth
			,row_number() over(partition by customer_id order by to_char(rental_date, 'YYYY/MM') desc) month_rank 
			,customer_id
			,count(rental_id) montlhy_rentals			
	from rental
	group by to_char(rental_date, 'YYYY/MM'), date_trunc('month',rental_date), customer_id	
	order by customer_id, to_char(rental_date, 'YYYY/MM')
),
lag_calculation
as(
	select	CASE WHEN 
					lag(montlhy_rentals) over(partition by customer_id order by mnth) IS NULL THEN 0				
				ELSE 
					lag(montlhy_rentals) over(partition by customer_id order by mnth)
			END prev_montlhy_rentals
	 		,mnth
			,customer_id
			,montlhy_rentals
	from customer_monthly_rentals
	where
		month_rank < 4  -- check only the most recent 3 months
	order by customer_id, mnth
),
diff_calculation
as(
	select	mnth
			,customer_id
			,montlhy_rentals - prev_montlhy_rentals as diff_rentals, montlhy_rentals, prev_montlhy_rentals			
	from lag_calculation
	order by customer_id, mnth
),
low_rental_rate_customers
as(
	select customer_id, max(diff_rentals) max_rentals_diff, case when max(diff_rentals) < 4 then 1 else 0 end as low_rental_rate_ind
	from diff_calculation
	group by customer_id
	order by 3 desc
),
	-- Too old last rental indicator:  last rental before two months
last_rental
as (
	select customer_id, max(rental_date) last_rental_date
	from rental 
	group by customer_id
	order by 2 desc
),
last_rental_too_old_customers
as(
select 	customer_id
		,last_rental_date
		,case when last_rental_date < date '2006-02-16' - 60  -- CURRENT_DATE - 60 
				then 1 
				else 0 end 
		as last_rental_too_old_ind
from last_rental
order by 3 desc
),
	-- low avg number of rentals per month
rentals_per_month
as (
	select customer_id, to_char(rental_date, 'YYYY/MM') mnth, count(rental_id) num_rentals
	from rental
	group by customer_id, to_char(rental_date, 'YYYY/MM')
	order by 1,2,3
),
low_avg_rentals_customers
as(
	select customer_id, round(avg(num_rentals),1) avg_num_rentals, case when avg(num_rentals) < 5 then 1 else 0 end as low_avg_rentals_per_month_id 
	from rentals_per_month
	group by customer_id
	order by 3 desc
),
	-- customer tenure: total days the customer has been an active customer
	-- An active customer is one that has at least one rental in the last two months
active_customers
as(
	select distinct customer_id, 1 as active_customer_ind
	from rental
	where 
		rental_date > date '2006-02-16' - 60  -- CURRENT_DATE - 60
),
active_customers_tenure
as(
select 	customer_id
		,case when active_customer_ind IS NULL then 0 else 1 end as active_ind
		, -- tenure
		 date '2006-02-16' - min(rental_date) as tenure
from rental left outer join active_customers using(customer_id)
group by customer_id, active_ind
order by 2 desc, 3 desc
),
customer_churn_profile
as(
select 	customer_id
		,low_rental_rate_ind, max_rentals_diff 
		,last_rental_too_old_ind, last_rental_date
		,low_avg_rentals_per_month_id, avg_num_rentals
		,active_ind
		,case when active_ind = 0 then interval '0 days' else tenure end as tenure
from customer left outer join low_rental_rate_customers using (customer_id)
					left outer join last_rental_too_old_customers using(customer_id)
						left outer join low_avg_rentals_customers using(customer_id)
							left outer join active_customers_tenure using(customer_id)
)
	-- !!SOS!! SAVE THESE CUSTOMERS!
select *
from customer_churn_profile
where
	(
		low_rental_rate_ind = 1
		OR last_rental_too_old_ind = 1
		OR low_avg_rentals_per_month_id = 1
	)
	AND active_ind = 1 -- an active customer
	AND tenure > interval '200 days' -- an old customer
order by low_rental_rate_ind desc, last_rental_too_old_ind desc, low_avg_rentals_per_month_id;
```
> Written with [StackEdit](https://stackedit.io/).
