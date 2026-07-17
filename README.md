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
select
support_agent_id
,cast(message as string) as message
,ticket_id
,regexp_substr(cast(message as string), '[0-9]{4}-[0-9]{2}-[0-9]{2}') as real_date
from (
select
support_agent_id
,ticket_id
,user_id
,explode(messages) as message
from samples.wanderbricks.customer_support_logs
) base
qualify row_number() over(partition by ticket_id order by regexp_substr(cast(message as string), '[0-9]{4}-[0-9]{2}-[0-9]{2}') desc) = 1
order by ticket_id asc
```

**Explanation:** Four-CTE pipeline — `EXPLODE()` flattens the nested messages array into rows, `CAST AS STRING` enables regex parsing, `REGEXP_SUBSTR` extracts the embedded ISO date, and `ROW_NUMBER()` isolates the latest message per ticket before the final filter.

<img width="910" height="357" alt="wanderbricks_Q15" src="https://github.com/user-attachments/assets/c56adf01-425f-440d-bd49-bb29bbddd759" />


---

### 🔹 Q16: Support Message Sentiment Analysis 
Extracts from each message string:
- `reporter` — who sent the message (agent or user)
- `mood` — sentiment label (angry, caring, helpful, etc.)

**Purpose:** Monitor emotional tone of support interactions to trigger escalations and measure agent quality.

```sql
select
support_agent_id
,cast(message as string) as message
,ticket_id
,regexp_substr(cast(message as string), '[0-9]{4}-[0-9]{2}-[0-9]{2}') as real_date
,regexp_extract(CAST(message as string), ',\\s*([^,]+),\\s*[^,]+,\\s*[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9:]{8}', 1) as reporter
,regexp_extract(cast(message as string), ',\\s*([^,]+),\\s*[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9:]{8}', 1) as mood
from (
select
support_agent_id
,ticket_id
,user_id
,explode(messages) as message
from samples.wanderbricks.customer_support_logs
) base
qualify row_number() over(partition by ticket_id order by regexp_substr(cast(message as string), '[0-9]{4}-[0-9]{2}-[0-9]{2}') desc) = 1
order by ticket_id asc
```

**Explanation:** Two `REGEXP_EXTRACT` calls target different positional fields within the serialized message struct string. `QUALIFY` filters to the most recent message per ticket without a subquery wrapper — a Databricks SQL shorthand for post-window filtering.

<img width="1067" height="372" alt="wanderbricks_Q16" src="https://github.com/user-attachments/assets/407e0488-036e-4f86-8b40-bf1389bf85aa" />


---

### 🔹 Q17: Conversation Duration
Calculates:
- Days between first and last message in each support ticket

**Purpose:** Measure SLA compliance and resolution time benchmarking across support agents.

```sql
with exploded as (
select
support_agent_id
,ticket_id
,cast(message as string) as message
,regexp_substr(cast(message as string), '[0-9]{4}-[0-9]{2}-[0-9]{2}') as real_date
from samples.wanderbricks.customer_support_logs
lateral view explode(messages) as message
),
ranked as (
select
support_agent_id
,ticket_id
,message
,real_date
,row_number() over (partition by ticket_id order by real_date asc) as first_message
,row_number() over (partition by ticket_id order by real_date desc) as last_message
from exploded
),
date_extract as (
select
ticket_id
,max(case when first_message = 1 then message end) as first_message
,max(case when last_message = 1 then message end) as last_message
,min(real_date) as first_date
,max(real_date) as last_date
from ranked
group by ticket_id
)
select 
a.ticket_id
,b.messages as message
,first_date
,last_date
,date_diff(a.last_date, a.first_date) as duration
from date_extract a
join samples.wanderbricks.customer_support_logs b
on a.ticket_id = b.ticket_id
order by ticket_id
```

**Explanation:** Dual `ROW_NUMBER()` window functions assign first and last message ranks simultaneously. `MAX(CASE WHEN)` then pivots the ranked rows into a single-row-per-ticket summary before `DATEDIFF` computes the conversation span.

<img width="946" height="366" alt="wanderbricks_Q17" src="https://github.com/user-attachments/assets/b0d1139d-d5ed-441b-a6af-de2526fd1f5c" />


---

### 🔹 Q18: Top 3 Properties per Destination
Ranks properties within each destination:
- By guest rating using `ROW_NUMBER()`
- Returns top 3 per destination only

**Purpose:** Power "Top Picks" features, host recognition programs, and destination marketing pages.

```sql
select 
a.property_id
,a.title
,a.property_type
,b.destination
,c.rating
,row_number() over(partition by destination order by rating desc) as rank
from 
samples.wanderbricks.properties a
join samples.wanderbricks.destinations b
on a.destination_id = b.destination_id
join samples.wanderbricks.reviews c
on a.property_id = c.property_id
qualify rank <= 3
order by destination asc
```

**Explanation:** `ROW_NUMBER()` assigns a unique sequential rank within each destination partition — unlike `RANK()`, it never ties, guaranteeing exactly 3 rows per destination. `QUALIFY rank <= 3` filters post-window without a subquery.

<img width="650" height="362" alt="wanderbricks_Q18" src="https://github.com/user-attachments/assets/ed348d68-2c8f-4daf-8908-31e1b41862f6" />


---

### 🔹 Q19: Low Rating Root Causes
Joins reviews under 3 stars to properties and destinations to:
- Surface repeated complaint patterns
- Count frequency of the same comment across the same property and destination

**Purpose:** Detect systemic quality issues — a comment that recurs many times signals a structural problem, not a one-off complaint.

```sql
select 
a.property_id
,a.title
,a.property_type
,b.destination
,c.rating
,c.comment
,count(*) over(partition by comment, b.destination, a.title) as ratings_count
from 
samples.wanderbricks.properties a
join samples.wanderbricks.destinations b
on a.destination_id = b.destination_id
join samples.wanderbricks.reviews c
on a.property_id = c.property_id
where c.rating is not null
and c.rating < 3
order by title, destination, a.property_id, rating asc
```

**Explanation:** `COUNT(*) OVER (PARTITION BY comment, destination, title)` adds a frequency score to every row without collapsing them — high `ratings_count` values surface recurring complaints that should be prioritized for host coaching or listing removal.

<img width="943" height="366" alt="wanderbricks_Q19" src="https://github.com/user-attachments/assets/44be83f3-40bb-461e-8ccd-81c3b2d352b3" />

### 🔹 Q20: Property Distance Check (Haversine Formula)

Computes:

- Distance in kilometers between every pair of properties  
- Using the Haversine spherical distance formula  
- Sorted from nearest to farthest

**Purpose:**  
Identify clusters of nearby properties for dynamic pricing, neighborhood grouping, map-based search features, and geo‑deduplication of duplicate listings.

```sql
select
a.property_id as property_id_1
,b.property_id as property_id_2
,a.title as property_title_1
,b.title as property_title_2
,a.property_latitude as lat1
,a.property_longitude as lon1
,b.property_latitude as lat2
,b.property_longitude as lon2
,round(6371 * 2 * asin(sqrt(power(sin(radians((b.property_latitude - a.property_latitude) / 2)), 2) 
+ cos(radians(a.property_latitude)) 
* cos(radians(b.property_latitude)) 
* power(sin(radians((b.property_longitude - a.property_longitude) / 2)), 2)
)), 2) as distance_km
from samples.wanderbricks.properties a
join samples.wanderbricks.properties b
on a.property_id < b.property_id
order by distance_km asc
```

**Explanation:** 
Uses the Haversine formula to compute great‑circle distance between two latitude/longitude points on Earth.
The a.property_id < b.property_id condition prevents duplicate pairings (A–B and B–A).
Sorting by distance_km surfaces extremely close listings — useful for detecting duplicates, multi‑unit buildings, or hyper‑dense markets.


<img width="930" height="332" alt="wanderbricks_Q20pt1" src="https://github.com/user-attachments/assets/b49bf297-830d-4fd2-a252-a05b24cef8e4" />

### 🔹 Q21: Suspicious Booking Amounts — Payment vs. Booking Total Mismatch

Flags bookings where:

- `bookings.total_amount` does **not** match  
- The sum of actual payments recorded in `payments.amount`

**Purpose:**  
Detect billing inconsistencies, partial payments, overcharges, undercharges, and potential fraud or reconciliation errors.

```sql
with base as (
select 
a.booking_id
,check_in
,check_out
,format_number(total_amount,'#.##') as booking_payment
,abs(round(amount,2)) as actual_payment
-- ,payment_method
from samples.wanderbricks.bookings a
join samples.wanderbricks.payments b
on a.booking_id = b.booking_id
),
differences as (
select 
*
,round(try_subtract(booking_payment, actual_payment),2) price_difference
from base
where try_subtract(booking_payment, actual_payment)<> 0
)
select 
*
,case when price_difference > 0 then 'increase'
when price_difference < 0 then 'decrease'
else 'none' end as price_difference
from differences
order by booking_id asc
```

**Explanation:**
The base CTE aligns booking totals with actual payment rows.
try_subtract() safely computes the difference even when formatting or type mismatches occur.
Any non‑zero difference indicates a suspicious discrepancy — either overpayment (increase) or underpayment (decrease).
This query is essential for financial audits, chargeback investigations, and monthly reconciliation workflows.

<img width="841" height="407" alt="wanderbricks_Q20" src="https://github.com/user-attachments/assets/9d82f5f4-8ae2-413e-a8f4-39ddac362694" />



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

*I Built this with Apache Spark SQL on Databricks | July 2026*




