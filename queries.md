# SQL Queries

## QUERY 1
What is the total dollar amount of debit card transactions on this platform for a given week? 

```sql
SELECT
	'2023-07-23' as week_starting,
	'2023-07-29' as week_ending,
	SUM(amount) as total_debit_transactions
FROM fact_transaction
WHERE transaction_type = 'debit purchase'
AND transaction_status = 'completed'
AND DATE(transaction_created_at) BETWEEN '2023-07-23' AND '2023-07-29'
GROUP BY transaction_type
```


Notes:
* When filtering for dates in the where clause you could also use a DATE_TRUNC function and specify 'week' as the parameter to get the beginning of a week and then a DATE_ADD function to get 7 days later. Since we are looking for a 'given week' we hardcode it instead of aggregating across the dataset.



## QUERY 2
Who are the largest customers by payment volume?

```sql
SELECT
	b.business_key,
	b.business_id,
	b.business_dba_name as customer_name,
	SUM(t.amount) as payment_total,
	RANK() OVER(ORDER BY SUM(t.amount) DESC) as payment_ranking
FROM fact_transaction t
JOIN dim_business b ON b.business_key = t.business_key
WHERE transaction_type = 'payments'
AND transaction_status = 'completed'
GROUP BY b.business_dba_name
ORDER BY payment_ranking ASC
```


Notes:
* Making the assumption that we only want completed payments


## QUERY 3
What is the total end-of-week balance across all accounts at the bank?

```sql
SELECT
	LAST_DAY(CURRENT_DATE, 'week') as week_ending,
	SUM(amount) as week_end_balance
FROM fact_transaction
WHERE DATE(transaction_created_at) <= LAST_DAY(CURRENT_DATE, 'week')
AND transaction_status IN ('completed')
AND is_pending = False
```


Notes:
* Assuming that we only want transactions that are completed as of week-end. They may be additional statuses that you would want to include depending on how the 'current balance' for an account is calculated. IE: Do pending payments subtract from the current_balance that is available to a customer?

* Assuming that there are transaction types for both 'fee_credit' and 'fee_debit'. A fee debit would be recorded as a negative number to indicate that a fee was incurred by a customer and an amount is to be debited from their account. A fee credit would be in the event of a fee reversal which would should a positive dollar amount being added back to an account.

* Assuming that transaction fees are recorded as their own row in the fact_transaction table with the correct sign +/- with a transaction_type = 'account_fees'. So when you run a SUM function on the AMOUNT column, everything adds up correctly. For example, a customer could initiate a payment of $50.00 to a payee and there would be a transaction row that indicates an amount of -50.00 as well as another row that indicates a fee of $0.25 as -0.25 for a total balance hit to the account owner of -50.25. If it were an internal transfer to another business within the bank, there would be a credit of $50.00 to their account. The 0.25 fee would be retained by the bank's internal accounts.