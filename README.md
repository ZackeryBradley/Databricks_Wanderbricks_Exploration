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

### 🔹 Q1: Booking Status Consistency
Identifies bookings where:
- `bookings.status` and `booking_updates.status` disagree

**Purpose:** Detect data pipeline sync failures or race conditions between the source-of-truth table and the event log.

```sql
SELECT
    a.booking_id,
    a.status                AS bookings_status,
    b.status                AS booking_updates_status
FROM samples.wanderbricks.bookings        AS a
INNER JOIN samples.wanderbricks.booking_updates AS b
    ON a.booking_id = b.booking_id
WHERE a.status != b.status
ORDER BY a.booking_id;
```

**Explanation:** Uses an INNER JOIN to compare status values across both tables. Any row returned signals a mismatch that needs reconciliation. An empty result = tables are in sync ✅

<!-- Add screenshot here -->

---

### 🔹 Q2: Average Stay Length by Destination
Calculates:
- Average number of nights stayed per destination
- Sorted longest to shortest

**Purpose:** Understand destination appeal and inform pricing tiers and marketing campaigns.

```sql
SELECT
    c.destination,
    ROUND(AVG(DATEDIFF(a.check_out, a.check_in)), 2) AS avg_stay_length
FROM samples.wanderbricks.booking_updates  AS a
JOIN samples.wanderbricks.properties       AS b
    ON a.property_id = b.property_id
JOIN samples.wanderbricks.destinations     AS c
    ON b.destination_id = c.destination_id
GROUP BY c.destination
ORDER BY avg_stay_length DESC;
```

**Explanation:** Uses `DATEDIFF(check_out, check_in)` to compute nights per booking, then averages across all bookings per destination via a 3-table join chain.

<!-- Add screenshot here -->

---

### 🔹 Q3: Revenue by Month — Bookings vs. Booking Updates
Calculates per month (completed bookings only):
- Total original booking amount
- Total updated/adjusted amount
- Signed price difference between the two

**Purpose:** Audit post-booking price adjustments and measure revenue reconciliation gaps month over month.

```sql
WITH main AS (
    SELECT
        DATE_FORMAT(a.check_in, 'yyyy-MM')   AS month,
        b.status,
        SUM(a.total_amount)                  AS total_amount,
        SUM(b.total_amount)                  AS total_amount_updates
    FROM samples.wanderbricks.bookings        AS a
    JOIN samples.wanderbricks.booking_updates AS b
        ON a.booking_id = b.booking_id
    WHERE b.status = 'completed'
    GROUP BY ALL
    ORDER BY month DESC
),
solution AS (
    SELECT
        month,
        status,
        FORMAT_NUMBER(total_amount,         '$###,###.##') AS total_amount,
        FORMAT_NUMBER(total_amount_updates, '$###,###.##') AS total_amount_updates,
        FORMAT_NUMBER(total_amount_updates - total_amount, '$###,###.##') AS price_diff
    FROM main
)
SELECT * FROM solution;
```

**Explanation:** Two-CTE pattern — `main` aggregates raw sums by month, `solution` formats them as currency strings and surfaces the delta. A positive `price_diff` = upward adjustment; negative = discount or correction applied post-booking.

<!-- Add screenshot here -->

---

### 🔹 Q4: Guest Count Distribution by Property Type
Calculates:
- Number of bookings grouped by guest count
- Broken down by property type

**Purpose:** Understand which property types attract solo travelers vs. large groups for capacity planning and pricing tiers.

```sql
SELECT
    b.property_id,
    COUNT(a.booking_id)  AS bookings,
    a.guests_count       AS guest_count,
    b.property_type
FROM samples.wanderbricks.booking_updates AS a
JOIN samples.wanderbricks.properties      AS b
    ON a.property_id = b.property_id
GROUP BY ALL
ORDER BY guest_count DESC;
```

**Explanation:** Uses `GROUP BY ALL` (Databricks shorthand) to group on all non-aggregated columns. `COUNT(booking_id)` tallies bookings per property-type and guest-count combination.

<!-- Add screenshot here -->

---

### 🔹 Q5: Properties Missing Amenities
Identifies listings that have:
- No rows in `property_amenities` (no amenities assigned)

**Purpose:** Flag incomplete listings that may hurt search ranking, guest trust, and conversion rates.

```sql
SELECT
    p.property_id,
    p.title,
    p.destination_id
FROM samples.wanderbricks.properties              AS p
LEFT JOIN samples.wanderbricks.property_amenities AS pa
    ON p.property_id = pa.property_id
WHERE pa.property_id IS NULL
ORDER BY p.property_id;
```

**Explanation:** Classic anti-join pattern — LEFT JOIN retains all properties; `WHERE pa.property_id IS NULL` isolates those with no match in the bridge table. These listings need amenities assigned before going live.

<!-- Add screenshot here -->

---

### 🔹 Q6: Top Amenity Categories per Property
Ranks:
- Amenity categories by how often they appear per property
- Using `DENSE_RANK()` so tied categories share the same rank

**Purpose:** Reveal which amenity categories are most common — informs search filters and host guidance on what guests expect most.

```sql
SELECT
    b.property_id,
    a.category,
    DENSE_RANK() OVER (
        PARTITION BY b.property_id
        ORDER BY COUNT(*) DESC
    )               AS ranking,
    COUNT(*)        AS count_frequency
FROM samples.wanderbricks.amenities          AS a
JOIN samples.wanderbricks.property_amenities AS b
    ON a.amenity_id = b.amenity_id
GROUP BY b.property_id, a.category
ORDER BY b.property_id, ranking;
```

**Explanation:** Groups by property and category to count amenity frequency, then applies `DENSE_RANK()` so the most-common category per property = rank 1. Unlike `RANK()`, `DENSE_RANK()` produces no gaps when categories tie.

<!-- Add screenshot here -->

---

### 🔹 Q7: Primary Image Audit
Ensures each property has:
- Exactly one `is_primary = TRUE` image

Classifies each property as:
- `Valid` — exactly 1 primary image ✅
- `Missing primary image` — 0 primary images
- `Multiple primary images` — more than 1

**Purpose:** Prevent broken thumbnails and ambiguous display states across the platform.

```sql
WITH image_counts AS (
    SELECT
        property_id,
        SUM(CASE WHEN is_primary = TRUE THEN 1 ELSE 0 END) AS primary_count,
        COUNT(*)                                            AS total_images
    FROM samples.wanderbricks.property_images
    GROUP BY property_id
)
SELECT
    ic.property_id,
    ic.total_images,
    ic.primary_count,
    CASE
        WHEN ic.primary_count = 0 THEN 'Missing primary image'
        WHEN ic.primary_count = 1 THEN 'Valid'
        ELSE                          'Multiple primary images'
    END AS image_status
FROM image_counts AS ic
ORDER BY ic.primary_count, ic.property_id;
```

**Explanation:** CTE pre-aggregates primary image counts per property. The CASE expression in the main query then classifies each property into one of three audit states — rows where `image_status != 'Valid'` are remediation targets.

<!-- Add screenshot here -->

---

### 🔹 Q8: Destination Coverage
Counts:
- Number of properties per destination
- Enriched with country and continent from reference tables

**Purpose:** Geographic inventory view to identify underserved markets and guide host acquisition strategy.

```sql
WITH property_counts AS (
    SELECT
        d.destination_id,
        d.destination,
        d.country,
        d.state_or_province,
        COUNT(p.property_id) AS property_count
    FROM samples.wanderbricks.destinations AS d
    LEFT JOIN samples.wanderbricks.properties AS p
        ON d.destination_id = p.destination_id
    GROUP BY d.destination_id, d.destination, d.country, d.state_or_province
)
SELECT
    pc.destination_id,
    pc.destination,
    pc.country,
    c.continent,
    pc.state_or_province,
    pc.property_count
FROM property_counts AS pc
LEFT JOIN samples.wanderbricks.countries AS c
    ON pc.country = c.country
ORDER BY pc.property_count DESC;
```

**Explanation:** Double LEFT JOIN preserves destinations with zero properties (registered markets with no supply) and destinations whose country has no entry in the countries table. Both are strategically important gaps to surface.

<!-- Add screenshot here -->

---

### 🔹 Q9: Country Property Density
Ranks:
- Countries by total number of properties listed

**Purpose:** Country-level supply view to prioritize business development and host recruitment in underserved markets.

```sql
WITH property_count AS (
    SELECT
        COUNT(property_id) AS pcount,
        destination_id,
        title
    FROM samples.wanderbricks.properties
    GROUP BY ALL
),
countries AS (
    SELECT destination_id, country
    FROM samples.wanderbricks.destinations
)
SELECT
    SUM(pc.pcount) AS number_of_properties,
    c.country
FROM property_count AS pc
JOIN countries      AS c
    ON pc.destination_id = c.destination_id
GROUP BY ALL
ORDER BY number_of_properties DESC;
```

**Explanation:** Two-CTE pattern — first counts properties per destination, then re-aggregates by country to sum across all destinations within each nation. Produces a clean country-level supply leaderboard.

<!-- Add screenshot here -->

---

### 🔹 Q10: Business vs. Individual Spend
Compares:
- Average booking revenue for `is_business = TRUE` accounts
- vs. `is_business = FALSE` accounts
- Completed bookings only

**Purpose:** Quantify the B2B revenue premium to inform corporate pricing strategy and sales team priorities.

```sql
SELECT
    b.is_business,
    ROUND(AVG(a.total_amount), 2) AS avg_total_amount
FROM samples.wanderbricks.bookings AS a
JOIN samples.wanderbricks.users    AS b
    ON a.user_id = b.user_id
WHERE a.status = 'completed'
GROUP BY b.is_business
ORDER BY b.is_business DESC;
```

**Explanation:** Joins bookings to users on `user_id` and segments by the boolean `is_business` flag. Filtering `WHERE status = 'completed'` ensures only realized revenue is measured — cancellations and pending bookings are excluded.

<!-- Add screenshot here -->

---

### 🔹 Q11: Host Performance
Calculates:
- Running average rating per host
- Ordered from highest to lowest rating

**Purpose:** Score host quality for Superhost tiering programs and identify low-performing hosts who need coaching.

```sql
SELECT
    a.name,
    ROUND(
        AVG(a.rating) OVER (
            PARTITION BY a.name
            ORDER BY a.rating DESC
        ), 2
    ) AS avg_rating
FROM samples.wanderbricks.hosts AS a;
```

**Explanation:** Window function `AVG() OVER (PARTITION BY name ORDER BY rating DESC)` computes a running cumulative average per host, starting at their highest rating. For a simple single-number average per host, use `GROUP BY name` with `AVG(rating)` instead.

<!-- Add screenshot here -->

---

### 🔹 Q12: Refund Detection
Finds bookings that have:
- A `completed` payment AND a `refunded` payment

**Purpose:** Monitor chargeback risk, cancellation policy disputes, and potential fraud patterns.

```sql
-- Summary: which booking_ids have both statuses?
SELECT
    booking_id
FROM samples.wanderbricks.payments
WHERE status IN ('completed', 'refunded')
GROUP BY booking_id
HAVING COUNT(DISTINCT status) = 2;

-- Detail: full payment rows for flagged bookings
SELECT
    booking_id,
    amount,
    status
FROM samples.wanderbricks.payments
WHERE status IN ('completed', 'refunded')
  AND booking_id IN (
      SELECT booking_id
      FROM   samples.wanderbricks.payments
      WHERE  status IN ('completed', 'refunded')
      GROUP BY booking_id
      HAVING COUNT(DISTINCT status) = 2
  )
ORDER BY booking_id, status;
```

**Explanation:** `HAVING COUNT(DISTINCT status) = 2` is the core logic — a booking_id only qualifies if both 'completed' and 'refunded' statuses exist for it. The detail query then surfaces full payment rows so original and refund amounts appear side by side for audit.

<!-- Add screenshot here -->

---

### 🔹 Q13: Payment Lag
Calculates:
- Days between `booking_updates.created_at` and `payments.payment_date`

**Purpose:** Measure payment processing delays to detect billing friction, slow invoicing, or pipeline bottlenecks.

```sql
SELECT DISTINCT
    a.booking_id,
    CAST(a.created_at   AS DATE) AS booking_created_date,
    CAST(b.payment_date AS DATE) AS payment_date,
    DATEDIFF(b.payment_date, a.created_at) AS days_between
FROM samples.wanderbricks.booking_updates AS a
JOIN samples.wanderbricks.payments        AS b
    ON a.booking_id = b.booking_id
ORDER BY a.created_at ASC;
```

**Explanation:** Both timestamps are CAST to DATE to normalize any time component before computing the difference. A negative `days_between` means payment was logged before the booking update — a data ordering anomaly worth investigating.

<!-- Add screenshot here -->

---

### 🔹 Q14: User Funnel Analysis
Calculates conversion rates across:
- View → Click → Search → Booking

**Purpose:** Pinpoint where users drop off in the booking journey to prioritize UX improvements and A/B test targets.

```sql
WITH events AS (
    SELECT user_id, event, MIN(timestamp) AS first_event
    FROM samples.wanderbricks.clickstream
    GROUP BY user_id, event
),
funnel AS (
    SELECT
        e.user_id,
        MAX(CASE WHEN e.event = 'view'   THEN 1 ELSE 0 END) AS viewed,
        MAX(CASE WHEN e.event = 'click'  THEN 1 ELSE 0 END) AS clicked,
        MAX(CASE WHEN e.event = 'search' THEN 1 ELSE 0 END) AS searched,
        CASE WHEN b.booking_id IS NOT NULL THEN 1 ELSE 0 END AS booked
    FROM events AS e
    LEFT JOIN samples.wanderbricks.bookings AS b ON e.user_id = b.user_id
    GROUP BY e.user_id, b.booking_id
),
aggregation AS (
    SELECT
        COUNT(*)        AS total_users,
        SUM(viewed)     AS total_views,
        SUM(clicked)    AS total_clicks,
        SUM(searched)   AS total_searches,
        SUM(booked)     AS total_bookings
    FROM funnel
)
SELECT
    total_views, total_clicks, total_searches, total_bookings, total_users,
    ROUND((total_views    / total_users) * 100, 2) AS pct_viewed,
    ROUND((total_clicks   / total_users) * 100, 2) AS pct_clicked,
    ROUND((total_searches / total_users) * 100, 2) AS pct_searched,
    ROUND((total_bookings / total_users) * 100, 2) AS pct_booked
FROM aggregation;
```

**Explanation:** Three-CTE pipeline — `events` collapses clickstream to one row per user/event, `funnel` pivots events into binary flags using `MAX(CASE WHEN)`, and `aggregation` sums them to produce platform-wide totals before the final SELECT computes percentage rates.

<!-- Add screenshot here -->

---

### 🔹 Q15: Device & Channel Performance
Counts engagement by:
- Device type: desktop, mobile, tablet
- Referrer channel: ad, direct, email, google

**Purpose:** Allocate marketing spend and inform responsive design priorities based on where users actually come from.

```sql
SELECT
    SUM(CASE WHEN b.device_type = 'desktop' THEN 1 ELSE 0 END) AS total_desktop,
    SUM(CASE WHEN b.device_type = 'mobile'  THEN 1 ELSE 0 END) AS total_mobile,
    SUM(CASE WHEN b.device_type = 'tablet'  THEN 1 ELSE 0 END) AS total_tablet,
    SUM(CASE WHEN b.referrer   = 'ad'       THEN 1 ELSE 0 END) AS total_ad,
    SUM(CASE WHEN b.referrer   = 'direct'   THEN 1 ELSE 0 END) AS total_direct,
    SUM(CASE WHEN b.referrer   = 'email'    THEN 1 ELSE 0 END) AS total_email,
    SUM(CASE WHEN b.referrer   = 'google'   THEN 1 ELSE 0 END) AS total_google_search
FROM samples.wanderbricks.clickstream AS a
LEFT JOIN samples.wanderbricks.page_views AS b
    ON a.user_id = b.user_id
GROUP BY ALL;
```

**Explanation:** Conditional aggregation pivots categorical values into separate numeric columns in a single pass — more efficient than multiple filtered subqueries. The LEFT JOIN preserves all clickstream users even if they have no page_view record.

<!-- Add screenshot here -->

---

### 🔹 Q16: Extract Latest Support Message
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

### 🔹 Q17: Support Message Sentiment Analysis
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

### 🔹 Q18: Conversation Duration
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

### 🔹 Q19: Top 3 Properties per Destination
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

### 🔹 Q20: Low Rating Root Causes
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




