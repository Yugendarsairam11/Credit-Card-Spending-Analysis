# Credit-Card-Spending-Analysis

Lets Explore the world of credit card spending from the people of Indian cities

 - rename table credit_card_transcations to credit_card_trans;
# top 5 cities that lead in credit card spending, and what percentage do they contribute to the total?
 - with fetch_data as (
 - select
 - city,
 - sum(amount) as total_amount_spent,
 - (select sum(amount) from credit_card_trans) as whole_amount
 - from 
 - credit_card_trans
 - group by city
 - order by 2 desc
 - )
 - select
 - city,
 - total_amount_spent,
 - round((total_amount_spent / whole_amount) * 100,2) as contribution_percentage
 - from 
 - fetch_data
 - limit 5;
 
# How can we determine the highest spending month and amounts for each card type?
 - with fetch_data as (
 - select
 - transaction_id,
 - city,
 - STR_TO_DATE(transaction_date, '%d-%m-%Y') as transaction_date,
 - card_type,
 - exp_type,
 - gender,
 - amount
 - from 
 - credit_card_trans
 - ),
 - filter_data as (
 - select
 - card_type,
 - month(transaction_date) as month,
 - year(transaction_date) as year,
 - sum(amount) as total_spent,
 - dense_rank() over(partition by card_type order by sum(amount) desc) as ranking_order
 - from 
 - fetch_data
 - group by 1, 2, 3
 - )
 - select
 - card_type,
 - month as highest_spending_month,
 - year,
 - total_spent 
 - from 
 - filter_data
 - where ranking_order = 1
 
# Retrieve transaction details for each card type upon reaching a cumulative spend of 1,000,000.
 - with fetch_data as (
 - select
 - *, 
 - sum(amount) over(partition by card_type order by transaction_date,transaction_id) as cumulative_sum
 - from 
 - credit_card_trans
 - ),
 - filter_data as (
 - select 
 - *,
 - dense_rank() over(partition by card_type order by cumulative_sum) as ranking_order 
 - from 
 - fetch_data
 - where cumulative_sum >= 1000000
 - )
 - select
 - * 
 - from 
 - filter_data
 - where ranking_order = 1
 
# Discover the city with the lowest percentage spent for gold card holders.
 - with fetch_data as (
 - select
 - city,
 - card_type,
 - sum(case when card_type = 'Gold' then amount end) as gold_amount,
 - sum(amount) as total_amount
 - from 
 - credit_card_trans
 - group by 1, 2
 - order by 1, 2
 - )
 - select
 - city,
 - ( sum(gold_amount) / sum(total_amount) ) * 100 as percentage
 - from 
 - fetch_data
 - group by 1
 - having sum(gold_amount) is not null
 - order by 2
 - limit 1
 
# Generate a dynamic report showcasing city, highest_expense_type, and lowest_expense_type.
with fetch_data as (
select
city,
exp_type,
sum(amount) as total_amount
from
credit_card_trans
group by 1,2 
order by 1
),
min_and_max_expense as (
select
city,
min(total_amount) as min_spent,
max(total_amount) as max_spent
from 
fetch_data
group by 1
)
select
m1.city,
max(case when m1.max_spent = total_amount then exp_type end) as highest_expense_type,
min(case when m1.min_spent = total_amount then exp_type end) as lowest_expense_type
from 
min_and_max_expense as m1
join 
fetch_data as f
on m1.city = f.city
group by 1

# Uncover the percentage contribution of female spending across different expense type
# the reason behind not using where condition over here is we need female contribution over "different expense type"
# that means from the sum of amounts of differente expense types plays a pivot role and among them we need to get the share of female contributors
# so the group by takes care of expense type and returns the overall sum of each of the expense type
# now using a case statment on that will return female contribution and contribution percentage
# using a where condition here will skew the results and will constantly result in same overall sum value irrespective of expense type
# so that has to be avoided

 - select
 - exp_type,
 - (sum(case when gender = 'F' then amount else 0 end ) / sum(amount)) * 100
 - as percentage_contribution
 - from 
 - credit_card_trans
 - group by exp_type
 
# Find the card and expense type combination with the highest month over month growth in January 2014
 - with fetch_data as (
 - select
 - transaction_id,
 - city,
 - STR_TO_DATE(transaction_date, '%d-%m-%Y') as transaction_date,
 - card_type,
 - exp_type,
 - gender,
 - amount
 - from 
 - credit_card_trans
 - ),
 - filter_data as (
 - select
 - card_type,
 - exp_type,
 - extract(year from transaction_date) as year,
 - extract(month from transaction_date) as month,
 - sum(amount) as total_amount,
 - lag(sum(amount)) over(partition by card_type, exp_type order by extract(year from transaction_date), extract(month from transaction_date)) as previous_month
 - from 
 - fetch_data
 - group by card_type, exp_type, extract(year from transaction_date), extract(month from transaction_date)
 - )
 - select
 - *,
 - ((total_amount - previous_month) / previous_month) * 100 as growth
 - from
 - filter_data
 - where previous_month is not null and year = 2014 and month = 1
 - order by growth desc
 - limit 1
 
# Identify the city with the highest spending-to-transaction ratio on weekends.
 - with fetch_data as (
 - select
 - transaction_id,
 - city,
 - STR_TO_DATE(transaction_date, '%d-%m-%Y') as transaction_date,
 - card_type,
 - exp_type,
 - gender,
 - amount 
 - from 
 - credit_card_trans
 - ),
 - filter_data as (
 - select
 - *,
 - dayofweek(transaction_date) as week_day 
 - from 
 - fetch_data
 - where dayofweek(transaction_date) in (1,7)
 - )
 - select
 - city,
 - sum(amount) / count(transaction_id) as spending_to_transaction_ratio 
 - from
 - filter_data
 - group by city
 - order by 2 desc
 - limit 1
 
# Explore the city that achieved its 500th transaction in the shortest time after the first.
 - with fetch_data as (
 - select
 - transaction_id,
 - city,
 - STR_TO_DATE(transaction_date, '%d-%m-%Y') as transaction_date,
 - card_type,
 - exp_type,
 - gender,
 - amount,
 - dense_rank() over(partition by city order by STR_TO_DATE(transaction_date, '%d-%m-%Y'),transaction_id) as ranking_order 
 - from
 - credit_card_trans
 - ),
 - days_took as (
 - select
 - city,
 - min(transaction_date) as min_date,
 - max(transaction_date) as max_date,
 - datediff(max(transaction_date), min(transaction_date)) as days_took
 - from 
 - fetch_data
 - where ranking_order = 1 or ranking_order = 500
 - group by city
 - having count(ranking_order) = 2
 - )
 - select
 - * 
 - from 
 - days_took
 - order by days_took
 - limit 1
