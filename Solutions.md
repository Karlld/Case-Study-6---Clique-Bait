**1. Enterprise Relationship Diagram**

Using the following code in dbdiagram.io an ERD can be created:

```sql
Table event_identifier {
  event_type int
  event_name varchar(13)
}

Table campaign_identifier {
  campaign_id int
  products varchar(3)
  campaign_name varchar(33)
  start_date timestamp
  end_date timestamp
}

Table page_hierarchy {
  page_id int
  page_name varchar(14)
  product_category varchar(9)
  product_id int
}

Table users {
  user_id int
  cookie_id varchar(6)
  start_date timestamp
}

Table events {
  visit_id varchar(6)
  cookie_id varchar(6)
  page_id int
  event_type int
  sequence_number int
  event_time timestamp
}

Ref: events.cookie_id > users.cookie_id
Ref: events.page_id > page_hierarchy.page_id
Ref: events.event_type > event_identifier.event_type
```


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

What is the percentage of visits which view the checkout page but do not have a purchase event?

```sql
WITH user_events AS (SELECT u.user_id,
                           e.cookie_id,
	                       e.visit_id,
	                       CASE WHEN page_id = 12 THEN 1
	                            ELSE 0
	                            END AS checkout_view,
	                       CASE WHEN event_type = 3 THEN 1
	                            ELSE 0
	                            END AS purchase
					FROM events AS e 
					JOIN users AS u ON e.cookie_id = u.cookie_id),

   total_events AS(SELECT visit_id,
                          SUM(checkout_view) AS total_checkout,
                          SUM(purchase) AS total_purchase
                       FROM user_events
                       GROUP BY visit_id),
						
   total_non_purchase AS (SELECT COUNT(visit_id) AS total_non 
                             FROM total_events 
                             WHERE total_checkout >0
                             AND total_purchase<1),
   
    total_checkouts AS (SELECT COUNT(visit_id) AS total_checkouts
                             FROM total_events
                             WHERE total_checkout > 0)
					  
SELECT ROUND((total_non*100.0)/total_checkouts,2) AS non_purchase_events
          FROM total_non_purchase, total_checkouts;
```

| non_purchase_events |
|---------------------|
| 15.50 |

What are the top 3 pages by number of views?

```sql
SELECT page_id,
       COUNT(visit_id) AS total_visits
    FROM events
	GROUP BY page_id
	ORDER BY total_visits DESC
	LIMIT 3;
```

| page_id |  total_visits |
|---------|---------------|
| 2 |	4752 |
| 9 |	2515 |
| 10 |	2513 |

What is the number of views and cart adds for each product category?

```sql
WITH page_views AS(SELECT p.product_category,
	   COUNT(e.event_type) AS total_views
    FROM events AS e
	JOIN page_hierarchy AS p ON p.page_id = e.page_id
	WHERE event_type = 1 
	GROUP BY product_category),

 cart_adds AS (SELECT p.product_category,
	   COUNT(e.event_type) AS total_carts
    FROM events AS e
	JOIN page_hierarchy AS p ON p.page_id = e.page_id
	WHERE event_type = 2 
	GROUP BY product_category)

SELECT c.product_category,
       total_views AS page_views,
	   total_carts AS cart_adds
  FROM page_views AS p
  JOIN cart_adds AS c ON c.product_category = p.product_category
  GROUP BY c.product_category, total_views, total_carts;
```

| product_category | page_views | cart_adds |
|------------------|------------|-----------|
| Fish |	4633 |	2789 |
| Luxury |	3032 |	1870 |
| Shellfish |	6204 |	3792 |

What are the top 3 products by purchases?

```sql
WITH purchases AS (SELECT visit_id 
                       FROM events
                       WHERE event_type = 3)
	SELECT h.page_name,
	       COUNT(e.event_type) AS total_purchases
		   FROM events AS e
		   JOIN purchases AS p ON p.visit_id = e.visit_id 
		   JOIN page_hierarchy AS h ON h.page_id = e.page_id
		   WHERE event_type = 2
		   GROUP BY h.page_name
		   ORDER BY total_purchases DESC
		   LIMIT 3;
```

| page_name |	total_purchases |
|-----------|-------------------|
| Lobster |	754 |
| Oyster |	726 |
| Crab |	719 |

**3. Product Funnel Analysis**
   
Using a single SQL query - create a new output table which has the following details:

How many times was each product viewed?
How many times was each product added to cart?
How many times was each product added to a cart but not purchased (abandoned)?
How many times was each product purchased?

```sql
WITH purchases AS (SELECT DISTINCT visit_id 
                      FROM events
                      WHERE event_type = 3),

     abandons AS (SELECT DISTINCT visit_id
                     FROM events
                     WHERE event_type = 2
                     AND visit_id NOT IN (SELECT visit_id FROM purchases)),

    total_purchases AS (SELECT h.page_name,
                               COUNT(e.event_type) AS total_purchases
                           FROM events AS e
                           JOIN purchases AS p ON p.visit_id = e.visit_id 
                           JOIN page_hierarchy AS h ON h.page_id = e.page_id
                           WHERE e.event_type = 2
                           GROUP BY h.page_name),

    total_abandons AS (SELECT h.page_name,
                               COUNT(e.event_type) AS total_abandons
                           FROM events AS e
                           JOIN abandons AS a ON a.visit_id = e.visit_id 
                           JOIN page_hierarchy AS h ON h.page_id = e.page_id
                           WHERE e.event_type = 2
                           GROUP BY h.page_name),
	
         adds_views AS (SELECT p.page_name,
                               SUM(CASE WHEN event_type = 1 THEN 1 ELSE 0 END) AS views,
                               SUM(CASE WHEN event_type = 2 THEN 1 ELSE 0 END) AS cart_add
			  FROM events AS e
			  JOIN page_hierarchy AS p ON p.page_id = e.page_id
			  GROUP BY p.page_name)

SELECT COALESCE(p.page_name, a.page_name) AS page_name,
       views,
       cart_add,
       p.total_purchases,
       a.total_abandons
    FROM total_purchases AS p
    FULL OUTER JOIN total_abandons AS a 
    ON p.page_name = a.page_name
    JOIN adds_views AS av ON av.page_name = p.page_name;
```
	
| page_name |	views 	| cart_add |	total_purchases |  total_abandons |
|-----------|-----------|----------|--------------------|-----------------|
| Abalone |	1525 |	932 |	699 |	233 |
| Black Truffle | 1469 | 	924 |	707 |	217 |
| Crab |	1564 |	949 |	719 |	230 |
| Kingfish |	1559 |	920 |	707 |	213 |
| Lobster |	1547 |	968 |	754 |	214 |
| Oyster |	1568 |	943 |	726 |	217 |
| Russian Caviar | 1563 |	946 |	697 |	249 |
| Salmon |	1559 |	938 |	711 |	227 |
| Tuna |	1515 |	931 |	 697 |	234 |

Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.

```sql
WITH purchases AS (SELECT DISTINCT visit_id 
                        FROM events
                        WHERE event_type = 3),

   abandons AS (SELECT DISTINCT visit_id
                       FROM events
                       WHERE event_type = 2
                       AND visit_id NOT IN (SELECT visit_id FROM purchases)),

   total_purchases AS (SELECT h.product_category,
                             COUNT(e.event_type) AS total_purchases
                         FROM events AS e
                         JOIN purchases AS p ON p.visit_id = e.visit_id 
                         JOIN page_hierarchy AS h ON h.page_id = e.page_id
                         WHERE e.event_type = 2
                         GROUP BY h.product_category),

 total_abandons AS (SELECT h.product_category,
                           COUNT(e.event_type) AS total_abandons
                       FROM events AS e
                       JOIN abandons AS a ON a.visit_id = e.visit_id 
                       JOIN page_hierarchy AS h ON h.page_id = e.page_id
                       WHERE e.event_type = 2
                       GROUP BY h.product_category),

    adds_views AS (SELECT p.product_category,
                          SUM(CASE WHEN event_type = 1 THEN 1 ELSE 0 END) AS views,
                          SUM(CASE WHEN event_type = 2 THEN 1 ELSE 0 END) AS cart_add
                       FROM events AS e
                       JOIN page_hierarchy AS p ON p.page_id = e.page_id
                       GROUP BY p.product_category)

SELECT COALESCE(p.product_category, a.product_category, av.product_category) AS product_category,
       av.views,
       av.cart_add,
       p.total_purchases,
       a.total_abandons
   FROM total_purchases AS p
   FULL OUTER JOIN total_abandons AS a 
      ON p.product_category = a.product_category
  FULL OUTER JOIN adds_views AS av 
      ON COALESCE(p.product_category, a.product_category) = av.product_category
  WHERE p.product_category NOTNULL;
```

| product_category |	views | cart_add | total_purchases | total_abandons |
|------------------|----------|----------|-----------------|----------------|
| Luxury |	3032 |	1870 |	1404 |	466 |
| Shellfish |	6204 |	3792 |	2898 |	894 |
| Fish |	4633 |	 2789 |	2115 |	674 |

Use your 2 new output tables - answer the following questions:

Which product had the most views, cart adds and purchases?

Oyster views = 1568
Lobster adds = 968
Lobster purchases = 754

Which product was most likely to be abandoned?

Russian caviar abandons = 249

Which product had the highest view to purchase percentage?

```sql  
SELECT page_name,
       ROUND((purchases*100.0/views),2) AS purchase_view_pcent
	   FROM summary_table
	   GROUP BY page_name, purchases, views
	   ORDER BY purchase_view_pcent DESC
	   LIMIT 1;
```

| page_name | purchase_view_pcent |
|-----------|---------------------|
| Lobster  |	48.74 |

What is the average conversion rate from view to cart add?

```sql  
SELECT ROUND(SUM(cart_add)*100.0/SUM(views),2) as view_add_conversion
       FROM summary_table;
```
	  
| view_add_conversion |
|---------------------|
| 60.93 |

What is the average conversion rate from cart add to purchase?

```sql  
SELECT ROUND(SUM(total_purchases)*100.0/SUM(cart_add),2) as add_purchase_conversion
       FROM summary_table;
```

| add_purchase_conversion |
|-------------------------|
| 75.93  |

**4. Campaigns Analysis**

Generate a table that has 1 single row for every unique visit_id record and has the following columns:

user_id
visit_id
visit_start_time: the earliest event_time for each visit
page_views: count of page views for each visit
cart_adds: count of product cart add events for each visit
purchase: 1/0 flag if a purchase event exists for each visit
campaign_name: map the visit to a campaign if the visit_start_time falls between the start_date and end_date
impression: count of ad impressions for each visit
click: count of ad clicks for each visit
(Optional column) cart_products: a comma separated text value with products added to the cart sorted by the order they were added to the cart (hint: use the sequence_number)
Use the subsequent dataset to generate at least 5 insights for the Clique Bait team - bonus: prepare a single A4 infographic that the team can use for their management reporting sessions, be sure to emphasise the most important points from your findings.

```sql
SELECT u.user_id,
       e.visit_id,
       COUNT(e.page_id) AS page_visits,
       MIN(e.event_time) AS visit_start_time,
       SUM(CASE WHEN e.event_type = 2 THEN 1 ELSE 0 END) AS cart_adds,
       SUM(CASE WHEN e.event_type = 3 THEN 1 ELSE 0 END) AS purchase,
       c.campaign_name,
       SUM(CASE WHEN e.event_type = 4 THEN 1 ELSE 0 END) AS ad_impressions,
       SUM(CASE WHEN e.event_type = 5 THEN 1 ELSE 0 END) AS ad_clicks,
       STRING_AGG(CASE WHEN p.product_id NOTNULL AND e.event_type = 2 THEN p.page_name ELSE NULL END, ', ' ORDER BY e.sequence_number) AS 
       purchase_items
   FROM events AS e
   JOIN users AS u ON u.cookie_id = e.cookie_id
   JOIN page_hierarchy AS p ON p.page_id = e.page_id
   LEFT JOIN campaign_identifier AS c ON e.event_time BETWEEN c.start_date AND c.end_date
   GROUP BY e.visit_id, u.user_id, c.campaign_name
   LIMIT 10;
```

| user_id | visit_id | page_visits | visit_start_time | cart_adds | purchase | campaign_name | ad_impressions | ad_clicks |	 purchase_items |
|---------|----------|-------------|------------------|-----------|----------|---------------|----------------|-----------|-------|
| 155	| 001597 |	19 |	2020-02-17 00:21:45.295141 |	6 |	 1 |	Half Off - Treat Your Shellf(ish) |	1 |	1 |	   Salmon, Russian Caviar, Black Truffle, Lobster, Crab, Oyster |
| 243 |	002809 |	4 |	2020-03-13 17:49:55.45987 |	0 |	0 |	Half Off - Treat Your Shellf(ish) |	0 |	0 | |	
| 78 |	0048b2 |	10 |	2020-02-10 02:59:51.335452 |	4 |	0 |	Half Off - Treat Your Shellf(ish) |	0 |	0 |	|Kingfish, Russian Caviar, Abalone, Lobster |
| 228	 |004aaf |	9 |	2020-03-18 13:23:07.97394 | 2 |	1 |	  Half Off - Treat Your Shellf(ish) |	0 |	0 |	Tuna, Lobster |
| 237 | 005fe7 |	14 |	 2020-04-02 18:14:08.257711 |	4 |	1 |		0 |	0 |	Kingfish, Black Truffle, Crab, Oyster |
| 420 |	006a61 |	17 |	2020-01-25 20:54:14.630253 |	5 |	1 |	25% Off - Living The Lux Life |	1 |	1 |	Tuna, Russian Caviar, Black Truffle, Abalone, Crab |
| 252	| 006e8c |	1 |	 2020-02-21 03:14:44.965938 |	0 |	0 |	Half Off - Treat Your Shellf(ish) |	0 |	0 | |	
| 20	| 006f7f |	9 |	2020-02-23 01:36:34.786358 |	1 |	1 |	Half Off - Treat Your Shellf(ish) |	1 |	1 |	Tuna |
| 436	| 007330 |	22 |	 2020-01-07 22:30:35.775068 |	8 |	1 |	BOGOF - Fishing For Compliments |	1 |	1 |	Salmon, Kingfish, Tuna, Russian Caviar, Black Truffle, Abalone, Lobster, Oyster |
| 161	| 009e0e |	13 |	2020-02-20 06:17:50.907354 |	5 |	0 |	Half Off - Treat Your Shellf(ish) |	0 |	0 |	Kingfish, Tuna, Black Truffle, Abalone, Lobster |




