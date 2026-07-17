# 🧱 WanderBricks SQL Analytics Project

> **A fully annotated, production-style SQL portfolio project built on the WanderBricks short-term rental dataset (Databricks `samples.wanderbricks.*`).**  
> 20 analytical queries covering data quality, revenue intelligence, funnel analysis, support operations, and guest reviews — each with full commentary, design notes, and extension ideas.

---

## 👤 Author

**Zack Bradley**  
Platform: Databricks SQL  
Dialect: Apache Spark SQL / ANSI SQL  
Dataset: `samples.wanderbricks.*`

---

## 📁 Project Structure

```
wanderbricks-sql-project/
│
├── README.md                          ← You are here
│
├── schema/
│   └── wanderbricks_schema.sql        ← Full schema reference with all 12 tables
│
├── docs/
│   └── query_index.md                 ← Quick-reference table of all 20 queries
│
└── queries/
    ├── 01_bookings/
    │   ├── q01_booking_status_consistency.sql
    │   ├── q02_average_stay_length.sql
    │   ├── q03_revenue_by_month.sql
    │   └── q04_guest_count_distribution.sql
    │
    ├── 02_properties/
    │   ├── q05_properties_missing_amenities.sql
    │   ├── q06_top_amenity_categories.sql
    │   ├── q07_primary_image_check.sql
    │   ├── q08_destination_coverage.sql
    │   └── q09_country_property_density.sql
    │
    ├── 03_users_and_hosts/
    │   ├── q10_business_vs_individual_spend.sql
    │   └── q11_host_performance.sql
    │
    ├── 04_payments/
    │   ├── q12_refund_detection.sql
    │   └── q13_payment_lag.sql
    │
    ├── 05_engagement/
    │   ├── q14_user_funnel_analysis.sql
    │   └── q15_device_performance.sql
    │
    ├── 06_support/
    │   ├── q16_extract_latest_message.sql
    │   ├── q17_sentiment_analysis.sql
    │   └── q18_conversation_duration.sql
    │
    └── 07_reviews/
        ├── q19_top3_properties_per_destination.sql
        └── q20_low_rating_root_causes.sql
```

---

## 🗄️ Data Model Overview

The WanderBricks dataset models a short-term vacation rental marketplace — think Airbnb/VRBO — with 12 core tables spanning bookings, properties, users, payments, and engagement signals.

```
users ──────────────────────────────────────────────────┐
  │                                                      │
  ├──► bookings ──────► booking_updates                  │
  │         │                                            │
  │         └──► payments                                │
  │                                                      │
  ├──► clickstream ──────► page_views                    │
  │                                                      │
  └──► customer_support_logs (messages[])                │
                                                         │
hosts ──► properties ──► destinations ──► countries      │
              │                                          │
              ├──► property_amenities ──► amenities      │
              │                                          │
              ├──► property_images                       │
              │                                          │
              └──► reviews ◄───────────────────────────-─┘
```

### Table Summary

| Table | Domain | Key Columns |
|---|---|---|
| `bookings` | Booking | booking_id, user_id, status, total_amount, check_in/out |
| `booking_updates` | Booking | booking_id, property_id, status, total_amount, created_at |
| `properties` | Property | property_id, host_id, destination_id, property_type |
| `destinations` | Geography | destination_id, destination, country, state_or_province |
| `countries` | Geography | country, continent |
| `amenities` | Property | amenity_id, name, category |
| `property_amenities` | Property | property_id, amenity_id (bridge table) |
| `property_images` | Property | property_id, url, is_primary |
| `users` | User | user_id, is_business |
| `hosts` | Host | host_id, name, rating |
| `payments` | Finance | payment_id, booking_id, amount, status, payment_date |
| `clickstream` | Engagement | user_id, event, timestamp |
| `page_views` | Engagement | user_id, device_type, referrer |
| `reviews` | Quality | property_id, rating, comment |
| `customer_support_logs` | Support | ticket_id, support_agent_id, messages[] |

> Full DDL with column types, descriptions, and domain context is in [`schema/wanderbricks_schema.sql`](schema/wanderbricks_schema.sql).

---

## 📂 Dataset
The dataset models a vacation rental marketplace (think Airbnb/VRBO) and includes:
- **Bookings** — reservation records, status, dates, amounts
- **Booking Updates** — event-driven mutations logged after initial booking
- **Properties** — listing registry with type, host, and destination
- **Destinations & Countries** — geographic reference tables
- **Amenities & Property Amenities** — feature assignments per listing
- **Property Images** — image records with primary image flag
- **Users & Hosts** — guest accounts and host profiles
- **Payments** — financial transactions per booking
- **Clickstream & Page Views** — user engagement and device/referral signals
- **Customer Support Logs** — support tickets with nested message arrays
- **Reviews** — guest ratings and freetext comments

---

## 📈 SQL Queries & Explanations

### 🔹 Q1: Booking Status Consistency Find bookings where bookings.status and booking_updates.status disagree.
Identifies bookings where:
- `bookings.status` and `booking_updates.status` disagree

**Purpose:** Detect data pipeline sync failures or race conditions between the source-of-truth table and the event log.

```sql
select
a.booking_id
,a.status
,b.status
from samples.wanderbricks.bookings a
inner join samples.wanderbricks.booking_updates b
on a.booking_id = b.booking_id
where a.status != b.status
order by a.booking_id asc
```

**Explanation:** Uses an INNER JOIN to compare status values across both tables. Any row returned signals a mismatch that needs reconciliation. An empty result = tables are in sync ✅

<<img width="301" height="368" alt="wanderbricks_Q1" src="https://github.com/user-attachments/assets/d557ab44-ecc4-4b95-8ac7-a589ef21c1a1" />
>

---

### 🔹 Q2: Average Stay Length by Destination
Calculates:
- Average number of nights stayed per destination
- Sorted longest to shortest

**Purpose:** Understand destination appeal and inform pricing tiers and marketing campaigns.

```sql
select
c.destination 
,round(avg(date_diff(a.check_out, a.check_in)),2) as avg_stay_length
from samples.wanderbricks.booking_updates a
join samples.wanderbricks.properties b
on a.property_id = b.property_id
join samples.wanderbricks.destinations c
on b.destination_id = c.destination_id
group by c.destination
order by avg_stay_length desc
```

**Explanation:** Uses `DATEDIFF(check_out, check_in)` to compute nights per booking, then averages across all bookings per destination via a 3-table join chain.

<img width="276" height="365" alt="wanderbricks_Q2" src="https://github.com/user-attachments/assets/5fa1f6d5-0208-4396-b8c6-bfd25744da29" />

---

### 🔹 Q3: Revenue by Month — Bookings vs. Booking Updates
Calculates per month (completed bookings only):
- Total original booking amount
- Total updated/adjusted amount
- Signed price difference between the two

**Purpose:** Audit post-booking price adjustments and measure revenue reconciliation gaps month over month.

```sql
with main as(
select 
date_format(a.check_in,'yyyy-MM') as month
,sum(a.total_amount) as total_amount
,sum(b.total_amount) as total_amount_updates
,b.status
from samples.wanderbricks.bookings a
join samples.wanderbricks.booking_updates b
on a.booking_id = b.booking_id
where b.status = 'completed'
group by all
order by month desc
)
,
solution (
select
month
,status
,format_number(total_amount, '$###,###.##') as total_amount
,format_number(total_amount_updates, '$###,###.##') as total_amount_updates
,format_number(total_amount_updates - total_amount, '$###,###.##') as price_diff
from main
)
select * from solution
```

**Explanation:** Two-CTE pattern — `main` aggregates raw sums by month, `solution` formats them as currency strings and surfaces the delta. A positive `price_diff` = upward adjustment; negative = discount or correction applied post-booking.

<img width="543" height="360" alt="wanderbricks_Q3" src="https://github.com/user-attachments/assets/c22c11ac-9fd4-43be-a53f-cac858c95969" />


---

### 🔹 Q4: Guest Count Distribution by Property Type
Calculates:
- Number of bookings grouped by guest count
- Broken down by property type

**Purpose:** Understand which property types attract solo travelers vs. large groups for capacity planning and pricing tiers.

```sql
select
b.property_id
,count(a.booking_id) as bookings
,a.guests_count as guest_count
,b.property_type
from samples.wanderbricks.booking_updates a
join samples.wanderbricks.properties b
on a.property_id = b.property_id
group by all
order by guest_count desc
```

**Explanation:** Uses `GROUP BY ALL` (Databricks shorthand) to group on all non-aggregated columns. `COUNT(booking_id)` tallies bookings per property-type and guest-count combination.

<img width="437" height="361" alt="wanderbricks_Q4" src="https://github.com/user-attachments/assets/74e83e22-e21d-4b9f-8d93-e3d4c42bb67b" />


---


### 🔹 Q5: Top  Amenity Categories per Property
Ranks:
- Amenity categories by how often they appear per property
- Using `DENSE_RANK()` so tied categories share the same rank

**Purpose:** Reveal which amenity categories are most common — informs search filters and host guidance on what guests expect most.

```sql
select 
property_id
,category
,dense_rank() over(partition by b.property_id order by count(*) desc) ranking
,count(*) as count_frequency
from samples.wanderbricks.amenities a
join samples.wanderbricks.property_amenities b
on a.amenity_id = b.amenity_id
group by b.property_id, category
order by property_id, ranking
```

**Explanation:** Groups by property and category to count amenity frequency, then applies `DENSE_RANK()` so the most-common category per property = rank 1. Unlike `RANK()`, `DENSE_RANK()` produces no gaps when categories tie.

<img width="445" height="365" alt="wanderbricks_Q5" src="https://github.com/user-attachments/assets/b6e02db7-bd4b-42c0-8011-21a81715b758" />


---

### 🔹 Q6: Primary Image Audit
Ensures each property has:
- Exactly one `is_primary = TRUE` image

Classifies each property as:
- `Valid` — exactly 1 primary image ✅
- `Missing primary image` — 0 primary images
- `Multiple primary images` — more than 1

**Purpose:** Prevent broken thumbnails and ambiguous display states across the platform.

```sql
with image_counts as (
select
property_id
,sum(case when is_primary = true then 1 else 0 end) as primary_count
,count(*) as total_images
from samples.wanderbricks.property_images
group by property_id
)

select
ic.property_id
,ic.total_images
,ic.primary_count
,case
when ic.primary_count = 0 then 'Missing primary image'
when ic.primary_count = 1 then 'Valid'
else 'Multiple primary images'
end as image_status
from image_counts ic
order by ic.primary_count, ic.property_id
```

**Explanation:** CTE pre-aggregates primary image counts per property. The CASE expression in the main query then classifies each property into one of three audit states — rows where `image_status != 'Valid'` are remediation targets.

<img width="467" height="362" alt="wanderbricks_Q6" src="https://github.com/user-attachments/assets/56da04cc-32b0-47b8-9940-c0e8bd6ffa7c" />


---

### 🔹 Q7: Destination Coverage
Counts:
- Number of properties per destination
- Enriched with country and continent from reference tables

**Purpose:** Geographic inventory view to identify underserved markets and guide host acquisition strategy.

```sql
with property_counts as (
select
d.destination_id
,d.destination
,d.country
,d.state_or_province
,count(p.property_id) as property_count
from samples.wanderbricks.destinations d
left join samples.wanderbricks.properties p
on d.destination_id = p.destination_id
group by
d.destination_id, d.destination,d.country, d.state_or_province
)
select
pc.destination_id
,pc.destination
,pc.country
,c.continent
,pc.state_or_province
,pc.property_count
from property_counts pc
left join samples.wanderbricks.countries c
on pc.country = c.country
order by pc.property_count desc
```

**Explanation:** Double LEFT JOIN preserves destinations with zero properties (registered markets with no supply) and destinations whose country has no entry in the countries table. Both are strategically important gaps to surface.

<img width="682" height="361" alt="wanderbricks_Q7" src="https://github.com/user-attachments/assets/448317f1-b4a1-457b-a968-02781ce5800c" />


---

### 🔹 Q8: Country Property Density
Ranks:
- Countries by total number of properties listed

**Purpose:** Country-level supply view to prioritize business development and host recruitment in underserved markets.

```sql
with property_count as
(

select 
count(property_id) as pcount
,destination_id
,title
from samples.wanderbricks.properties
group by all
)
,
countries as
(
select 
destination_id
,country
from samples.wanderbricks.destinations

)
select 
sum(pc.pcount) as number_of_properties
-- ,c.destination_id
,c.country
from property_count pc
join countries c
on pc.destination_id = c.destination_id
group by all
order by number_of_properties desc
```

**Explanation:** Two-CTE pattern — first counts properties per destination, then re-aggregates by country to sum across all destinations within each nation. Produces a clean country-level supply leaderboard.

<img width="300" height="363" alt="wanderbricks_Q8" src="https://github.com/user-attachments/assets/13cf2eed-7466-40b6-bc58-dac0eaf5ab29" />


---

### 🔹 Q9: Business vs. Individual Spend
Compares:
- Average booking revenue for `is_business = TRUE` accounts
- vs. `is_business = FALSE` accounts
- Completed bookings only

**Purpose:** Quantify the B2B revenue premium to inform corporate pricing strategy and sales team priorities.

```sql
select 
b.is_business,
avg(a.total_amount) as avg_total_amount
from samples.wanderbricks.bookings a
join samples.wanderbricks.users b
on a.user_id = b.user_id
where a.status = 'completed'
group by b.is_business
order by b.is_business desc
```

**Explanation:** Joins bookings to users on `user_id` and segments by the boolean `is_business` flag. Filtering `WHERE status = 'completed'` ensures only realized revenue is measured — cancellations and pending bookings are excluded.

<img width="272" height="98" alt="wanderbricks_Q9" src="https://github.com/user-attachments/assets/1662cc88-e6be-4c6e-bd6b-64fb640fec0d" />


---

### 🔹 Q10: Host Performance
Calculates:
- Running average rating per host
- Ordered from highest to lowest rating

**Purpose:** Score host quality for Superhost tiering programs and identify low-performing hosts who need coaching.

```sql
select 
a.name
,round(avg(a.rating) over(partition by name order by rating desc),2) as avg_rating
from samples.wanderbricks.hosts a
```

**Explanation:** Window function `AVG() OVER (PARTITION BY name ORDER BY rating DESC)` computes a running cumulative average per host, starting at their highest rating. For a simple single-number average per host, use `GROUP BY name` with `AVG(rating)` instead.

<img width="292" height="360" alt="wanderbricks_Q10" src="https://github.com/user-attachments/assets/fd2ef79a-8529-408f-8ac7-1ec27202dbd3" />


---

### 🔹 Q11: Refund Detection
Finds bookings that have:
- A `completed` payment AND a `refunded` payment

**Purpose:** Monitor chargeback risk, cancellation policy disputes, and potential fraud patterns.

```sql
-- Summary: which booking_ids have both statuses?
select 
booking_id
,amount
,status
from samples.wanderbricks.payments
where status in ('completed','refunded')
and booking_id in (
select booking_id
from samples.wanderbricks.payments
where status in ('completed','refunded')
group by booking_id
having count(distinct status) = 2
)
order by booking_id, status
```

**Explanation:** `HAVING COUNT(DISTINCT status) = 2` is the core logic — a booking_id only qualifies if both 'completed' and 'refunded' statuses exist for it. The detail query then surfaces full payment rows so original and refund amounts appear side by side for audit.

<img width="311" height="367" alt="wanderbricks_Q11" src="https://github.com/user-attachments/assets/0249b807-3bb7-426a-ad99-e2be89894d8a" />



---

### 🔹 Q12: Payment Lag
Calculates:
- Days between `booking_updates.created_at` and `payments.payment_date`

**Purpose:** Measure payment processing delays to detect billing friction, slow invoicing, or pipeline bottlenecks.

```sql
select distinct
a.booking_id
,cast(a.created_at as date)
,cast(b.payment_date as date)
,datediff(b.payment_date, a.created_at) as days_between
from samples.wanderbricks.booking_updates a
join samples.wanderbricks.payments b
on a.booking_id = b.booking_id
order by created_at asc
```

**Explanation:** Both timestamps are CAST to DATE to normalize any time component before computing the difference. A negative `days_between` means payment was logged before the booking update — a data ordering anomaly worth investigating.

<img width="472" height="361" alt="wanderbricks_Q12" src="https://github.com/user-attachments/assets/165fe34a-84b2-44de-b782-c84248f35a7c" />


---

### 🔹 Q13: User Funnel Analysis
Calculates conversion rates across:
- View → Click → Search → Booking

**Purpose:** Pinpoint where users drop off in the booking journey to prioritize UX improvements and A/B test targets.

```sql
with events as (
select 
user_id
,event
,min(timestamp) as first_event
from samples.wanderbricks.clickstream
group by user_id, event
),

funnel as (
select
e.user_id
,max(case when e.event = 'view' then 1 else 0 end) as viewed
,max(case when e.event = 'click' then 1 else 0 end) as clicked
,max(case when e.event = 'search' then 1 else 0 end) as searched
,case when b.booking_id is not null then 1 else 0 end as booked
from events e
left join samples.wanderbricks.bookings b
on e.user_id = b.user_id
group by e.user_id, b.booking_id
),

aggregation as (
select
count(*) as total_users
,sum(viewed) as total_views
,sum(clicked) as total_clicks
,sum(searched) as total_searches
,sum(booked) as total_bookings
from funnel
)

select
total_views
,total_clicks
,total_searches
,total_bookings
,total_users
,round((total_views/total_users)*100,2) as pct_viewed
,round((total_clicks/total_users)*100,2) as pct_clicked
,round((total_searches/total_users)*100,2) as pct_searched
,round((total_bookings/total_users)*100,2) as pct_booked
from aggregation
```

**Explanation:** Three-CTE pipeline — `events` collapses clickstream to one row per user/event, `funnel` pivots events into binary flags using `MAX(CASE WHEN)`, and `aggregation` sums them to produce platform-wide totals before the final SELECT computes percentage rates.

<img width="967" height="78" alt="wanderbricks_Q13" src="https://github.com/user-attachments/assets/c1e534d2-975c-4f37-8c68-98b215e70c25" />


---

### 🔹 Q14: Device & Channel Performance
Counts engagement by:
- Device type: desktop, mobile, tablet
- Referrer channel: ad, direct, email, google

**Purpose:** Allocate marketing spend and inform responsive design priorities based on where users actually come from.

```sql
select 
sum(case when b.device_type = 'desktop' then 1 else 0 end) as total_desktop
,sum(case when b.device_type = 'mobile' then 1 else 0 end) as total_mobile
,sum(case when b.device_type = 'tablet' then 1 else 0 end) as total_tablet
,sum(case when b.referrer = 'ad' then 1 else 0 end) as total_ad
,sum(case when b.referrer = 'direct' then 1 else 0 end) as total_direct
,sum(case when b.referrer = 'email' then 1 else 0 end) as total_email
,sum(case when b.referrer = 'google' then 1 else 0 end) total_google_search
from samples.wanderbricks.clickstream a
left join samples.wanderbricks.page_views b
on a.user_id = b.user_id
```

**Explanation:** Conditional aggregation pivots categorical values into separate numeric columns in a single pass — more efficient than multiple filtered subqueries. The LEFT JOIN preserves all clickstream users even if they have no page_view record.

<img width="772" height="76" alt="wanderbricks_Q14" src="https://github.com/user-attachments/assets/6914d716-5888-430e-bb92-7dfd324e0471" />


---

### 🔹 Q15: Extract Latest Support Message
Unnests the `messages[]` array and returns:
- The most recent message per support ticket

**Purpose:** Power support dashboards, SLA tracking, and agent handoff summaries with the latest context per ticket.

```sql
WITH unnested AS (
    SELECT created_at, EXPLODE(messages) AS message,
           support_agent_id, ticket_id, user_id
    FROM samples.wanderbricks.customer_support_logs
),
message_conversion AS (
    SELECT created_at, support_agent_id, ticket_id, user_id,
           CAST(message AS STRING) AS message
    FROM unnested
),
date_extract AS (
    SELECT created_at, message, support_agent_id, ticket_id, user_id,
           REGEXP_SUBSTR(message, '[0-9]{4}-[0-9]{2}-[0-9]{2}') AS real_date
    FROM message_conversion
),
date_rank AS (
    SELECT support_agent_id, message, ticket_id, user_id, real_date,
           ROW_NUMBER() OVER (PARTITION BY ticket_id ORDER BY real_date DESC) AS most_recent_date
    FROM date_extract
)
SELECT DISTINCT support_agent_id, message, ticket_id, real_date
FROM date_rank
WHERE most_recent_date = 1
ORDER BY ticket_id ASC;
```

**Explanation:** Four-CTE pipeline — `EXPLODE()` flattens the nested messages array into rows, `CAST AS STRING` enables regex parsing, `REGEXP_SUBSTR` extracts the embedded ISO date, and `ROW_NUMBER()` isolates the latest message per ticket before the final filter.

<!-- Add screenshot here -->

---

### 🔹 Q16: Support Message Sentiment Analysis
Extracts from each message string:
- `reporter` — who sent the message (agent or user)
- `mood` — sentiment label (angry, caring, helpful, etc.)

**Purpose:** Monitor emotional tone of support interactions to trigger escalations and measure agent quality.

```sql
SELECT
    support_agent_id,
    CAST(message AS STRING)                                               AS message,
    ticket_id,
    REGEXP_SUBSTR(CAST(message AS STRING), '[0-9]{4}-[0-9]{2}-[0-9]{2}') AS real_date,
    REGEXP_EXTRACT(
        CAST(message AS STRING),
        ',\\s*([^,]+),\\s*[^,]+,\\s*[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9:]{8}', 1
    ) AS reporter,
    REGEXP_EXTRACT(
        CAST(message AS STRING),
        ',\\s*([^,]+),\\s*[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9:]{8}', 1
    ) AS mood
FROM (
    SELECT support_agent_id, ticket_id, user_id, EXPLODE(messages) AS message
    FROM samples.wanderbricks.customer_support_logs
) AS base
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY ticket_id
    ORDER BY REGEXP_SUBSTR(CAST(message AS STRING), '[0-9]{4}-[0-9]{2}-[0-9]{2}') DESC
) = 1
ORDER BY ticket_id ASC;
```

**Explanation:** Two `REGEXP_EXTRACT` calls target different positional fields within the serialized message struct string. `QUALIFY` filters to the most recent message per ticket without a subquery wrapper — a Databricks SQL shorthand for post-window filtering.

<!-- Add screenshot here -->

---

### 🔹 Q17: Conversation Duration
Calculates:
- Days between first and last message in each support ticket

**Purpose:** Measure SLA compliance and resolution time benchmarking across support agents.

```sql
WITH exploded AS (
    SELECT support_agent_id, ticket_id,
           CAST(message AS STRING) AS message,
           REGEXP_SUBSTR(CAST(message AS STRING), '[0-9]{4}-[0-9]{2}-[0-9]{2}') AS real_date
    FROM samples.wanderbricks.customer_support_logs
    LATERAL VIEW EXPLODE(messages) AS message
),
ranked AS (
    SELECT support_agent_id, ticket_id, message, real_date,
           ROW_NUMBER() OVER (PARTITION BY ticket_id ORDER BY real_date ASC)  AS first_message,
           ROW_NUMBER() OVER (PARTITION BY ticket_id ORDER BY real_date DESC) AS last_message
    FROM exploded
),
date_extract AS (
    SELECT ticket_id,
           MAX(CASE WHEN first_message = 1 THEN message END) AS first_message,
           MAX(CASE WHEN last_message  = 1 THEN message END) AS last_message,
           MIN(real_date) AS first_date,
           MAX(real_date) AS last_date
    FROM ranked
    GROUP BY ticket_id
)
SELECT
    a.ticket_id, b.messages, a.first_date, a.last_date,
    DATEDIFF(a.last_date, a.first_date) AS duration
FROM date_extract AS a
JOIN samples.wanderbricks.customer_support_logs AS b ON a.ticket_id = b.ticket_id
ORDER BY a.ticket_id;
```

**Explanation:** Dual `ROW_NUMBER()` window functions assign first and last message ranks simultaneously. `MAX(CASE WHEN)` then pivots the ranked rows into a single-row-per-ticket summary before `DATEDIFF` computes the conversation span.

<!-- Add screenshot here -->

---

### 🔹 Q18: Top 3 Properties per Destination
Ranks properties within each destination:
- By guest rating using `ROW_NUMBER()`
- Returns top 3 per destination only

**Purpose:** Power "Top Picks" features, host recognition programs, and destination marketing pages.

```sql
SELECT
    a.property_id,
    a.title,
    a.property_type,
    b.destination,
    c.rating,
    ROW_NUMBER() OVER (
        PARTITION BY b.destination
        ORDER BY c.rating DESC
    ) AS rank
FROM samples.wanderbricks.properties   AS a
JOIN samples.wanderbricks.destinations AS b
    ON a.destination_id = b.destination_id
JOIN samples.wanderbricks.reviews      AS c
    ON a.property_id = c.property_id
QUALIFY rank <= 3
ORDER BY b.destination ASC, rank ASC;
```

**Explanation:** `ROW_NUMBER()` assigns a unique sequential rank within each destination partition — unlike `RANK()`, it never ties, guaranteeing exactly 3 rows per destination. `QUALIFY rank <= 3` filters post-window without a subquery.

<!-- Add screenshot here -->

---

### 🔹 Q19: Low Rating Root Causes
Joins reviews under 3 stars to properties and destinations to:
- Surface repeated complaint patterns
- Count frequency of the same comment across the same property and destination

**Purpose:** Detect systemic quality issues — a comment that recurs many times signals a structural problem, not a one-off complaint.

```sql
SELECT
    a.property_id,
    a.title,
    a.property_type,
    b.destination,
    c.rating,
    c.comment,
    COUNT(*) OVER (
        PARTITION BY c.comment, b.destination, a.title
    ) AS ratings_count
FROM samples.wanderbricks.properties   AS a
JOIN samples.wanderbricks.destinations AS b
    ON a.destination_id = b.destination_id
JOIN samples.wanderbricks.reviews      AS c
    ON a.property_id = c.property_id
WHERE c.rating IS NOT NULL
  AND c.rating < 3
ORDER BY a.title ASC, b.destination ASC, a.property_id ASC, c.rating ASC;
```

**Explanation:** `COUNT(*) OVER (PARTITION BY comment, destination, title)` adds a frequency score to every row without collapsing them — high `ratings_count` values surface recurring complaints that should be prioritized for host coaching or listing removal.

<!-- Add screenshot here -->

---

## 📊 Key Insights
- Status mismatches between bookings and booking_updates signal ETL sync gaps that need pipeline fixes
- Business accounts generate higher average booking revenue than personal accounts
- Some properties have no amenities assigned — a completeness gap that hurts conversion
- User engagement drops sharply between viewing and clicking — the biggest funnel leak
- Support tickets with repeated low-rating comments often point to systemic property issues, not one-off complaints

---

## 💡 Skills Demonstrated
- SQL joins and aggregations across multi-table schemas
- Window functions: `ROW_NUMBER()`, `DENSE_RANK()`, `AVG() OVER`, `COUNT() OVER`
- CTEs for modular, readable query design
- Anti-join pattern for data completeness checks
- Array handling: `EXPLODE()`, `LATERAL VIEW EXPLODE()`
- Regex extraction from nested struct strings
- Conditional aggregation and pivot patterns
- `QUALIFY` clause for post-window filtering
- Business-driven analytics across 7 analytical domains

---

*Built with Apache Spark SQL on Databricks | July 2026*




