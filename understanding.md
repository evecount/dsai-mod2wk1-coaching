Understanding the Big-Picture Architecture
Great! Let me break down the architecture in this lesson so you can see how all the pieces fit together.

The Core Problem
ShopStream has 10 million+ orders in PostgreSQL spanning 5 years. The operations team needs a live dashboard showing today's revenue, but:

It's slow — scanning 10M rows takes several seconds
It's risky — heavy analytics queries compete with the application's real transactions, potentially slowing down the checkout for customers
The Solution: The Multi-Layer Operational Data Store
Instead of hitting PostgreSQL directly, you add specialized layers, each optimized for a specific job:

Code
PostgreSQL (Source of Truth)
    ↓
MongoDB (Operational queries on recent data)
    ↓
Redis (Ultra-fast cached results)
    ↓
GCS (Long-term archive)
    ↓
BigQuery (Historical analysis - future lesson)
Why Multiple Layers?
Layer	Purpose	Time Window	Use Case
PostgreSQL	Source of truth	All history (5+ years)	Application writes & reads
MongoDB	Operational analytics	Recent window (35 days in this lesson)	Fast aggregations for dashboards
Redis	Cache layer	Minutes to hours	Sub-millisecond reads
GCS	Cold storage archive	Compliance, historical reference	Cheap long-term storage
BigQuery	Data warehouse	All history	Complex historical analysis (3+ years trends)
Real-World Examples
Uber's Driver/Rider Operations
Code
PostgreSQL (all trip history)
    → MongoDB (last 24 hours of trips)
    → Redis (active rides right now)
    → Dashboard: "How many drivers are within 2km of this rider?"
Why? Querying years of trip history to answer "active drivers near me right now" is overkill. A rolling 24-hour window in MongoDB is way faster.

Shopify's Merchant Dashboards
Code
PostgreSQL (all orders, all merchants)
    → MongoDB (last 30 days per merchant)
    → Redis (today's totals)
    → Dashboard: "Your sales today"
Why? A merchant expects their dashboard to load in <1 second. Running a SUM across every order they've ever received would be too slow.

Why MongoDB Specifically for the ODS?
MongoDB is the natural choice because:

Flexible document model — You can store denormalized order data (customer + items in one document) without complex joins
TTL indexes — MongoDB can automatically delete old documents, keeping only the rolling window you need
Fast aggregations — Its aggregation pipeline handles group-by-sum queries that operational dashboards need
Scales well — For tens of thousands of records, it's much faster than PostgreSQL without the operational risk
Why Redis on Top of MongoDB?
The dashboard might refresh 50 times per minute. Running the same MongoDB aggregation 50 times is wasteful when you can:

Compute the result once (takes ~50ms in MongoDB)
Store it in Redis with a 5-minute TTL
Serve dashboard reads from Redis (takes ~1ms or less)
This is the cache-miss / cache-hit pattern:

Cache miss (key not found): Query MongoDB, store result, return it
Cache hit (key exists): Return cached result instantly
Data Flow in This Lesson
Code
Step 1: Generate synthetic data (500 orders, last 35 days)
            ↓
Step 2: Insert into MongoDB
            ↓
Step 3: Write an aggregation query
        (revenue by category, last 30 days, exclude returns)
            ↓
Step 4: Cache the result in Redis with a 5-minute TTL
            ↓
Step 5: Measure the speedup (MongoDB vs Redis)
            ↓
Step 6: Export orders to JSON Lines format
            ↓
Step 7: Upload to Google Cloud Storage
Key Architectural Insight
This isn't about replacing PostgreSQL. It's about specialization:

PostgreSQL is great at transactional consistency (OLTP)
MongoDB is great at operational analytics (ODS)
Redis is great at caching hot data
GCS is great at cheap archival
BigQuery is great at analytical queries (OLAP)
Each tool does one thing really well. By using them together, you get fast dashboards without compromising the reliability of your application's transaction database.

Questions to deepen your understanding:
Why not skip MongoDB and cache PostgreSQL queries directly in Redis? (Hint: think about what happens with billions of rows)
What would happen if the TTL on Redis was set to 24 hours instead of 5 minutes? (Think: stale data vs. cache hit rate)
When would you actually need the GCS archive step? (Hint: compliance, audit trails, backup strategy)
