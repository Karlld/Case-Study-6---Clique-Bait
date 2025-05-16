**1. Enterprise Relationship Diagram**

![Clique_bait](https://github.com/user-attachments/assets/efb9d134-af10-4645-b453-71568852ffc1)


**2. Digital Analysis**

Using the available datasets - answer the following questions using a single query for each one:

How many users are there?

```sql
SELECT COUNT(DISTINCT(user_id)) AS total_users
       FROM users;
```
	  
| total_users |
|-------------|
|  500 |

How many cookies does each user have on average?

```sql
WITH total_cookies AS (SELECT user_id,
                             COUNT(cookie_id) AS total_cookies
                             FROM users
	                         GROUP BY user_id)
			SELECT ROUND(AVG(total_cookies)) AS avg_cookies
			       FROM total_cookies;
```

| avg_cookies |
|-------------|
|    4   |

What is the unique number of visits by all users per month?

```sql
SELECT DATE_TRUNC('month', event_time):: DATE AS date_month,
      COUNT(DISTINCT visit_id) AS total_visits
      FROM events
  GROUP BY date_month
  ORDER BY date_month;
```

| date_month | total_visits |
|------------|--------------|
| 2020-01-01 | 876 |
| 2020-02-01 | 1488 |
| 2020-03-01 |	916 |
| 2020-04-01 | 248 |
| 2020-05-01 | 36 |

What is the number of events for each event type?

```sql
SELECT event_type,
      COUNT(visit_id) AS total_events
      FROM events
  GROUP BY event_type
  ORDER BY event_type;
```

| event_type | total_events |
|------------|--------------|
| 1 |	20928 |
| 2 |	8451 |
| 3 |	1777 |
| 4 |	876 |
| 5 |	702 |

What is the percentage of visits which have a purchase event?

```sql
WITH total_purchases AS (SELECT event_type,
                                COUNT(distinct(visit_id)) AS total_purchases
                             FROM events
	                           WHERE event_type = 3
                         GROUP BY event_type
                         ORDER BY event_type),

total_visits AS (SELECT COUNT(distinct(visit_id)) AS total_visits
                      FROM events)

SELECT ROUND((total_purchases*100.00/total_visits),2) AS purchase_percentage
		      FROM total_purchases, total_visits;
```

| purchase_percentage |
|---------------------|
|     49.86  |
