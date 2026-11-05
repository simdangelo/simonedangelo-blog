---
date: 2026-05-18
modified: 2026-05-19T00:04:46+02:00
---

# The Interview Framework
This guide follows a systematic framework applicable to any system design interview. The approach consists of six distinct phases
1. **First, define the system requirements.** This includes both functional requirements (the core features) and non-functional requirements (system qualities such as scale, durability, and fault tolerance).
2. **Second, outline the core entities.** Create a high-level list of the primary entities that will be persisted and exchanged throughout the system. These correspond to the tables or collections in the data model.
3. **Third, specify the API.** Define the user-facing API, which establishes the contract between clients and the backend service.
4. **Fourth, address data flow** (applicable only to infrastructure-heavy designs like rate limiters or message queues, but not to user-facing products like URL shorteners).
5. **Fifth, develop the high-level design.** Diagram the system architecture with the primary goal of satisfying the functional requirements. At this stage, the system remains simple and may not yet handle scale or advanced features—it merely implements the core functionality.
6. **Sixth, conduct deep dives.** Iterate through each non-functional requirement, evolving the high-level design to satisfy them one by one. By the end of this phase, the system will satisfy all functional and non-functional requirements.

# Functional Requirements
**Functional requirements** represent the core features of the system—typically two to three essential capabilities that can be designed within a 35 to 45-minute interview. When designing an unfamiliar system, clarify its purpose through targeted questions.

A URL shortener **converts long URLs into short ones**, which then **redirect users to the original destination**. For this system, the functional requirements are:
1. **Users can create a short URL from a long URL.**
2. **Users can be redirected to the original URL from the short URL.**
3. **Users can optionally specify a custom alias** rather than accepting an auto-generated short code.
4. **Users can optionally specify an expiration time**, after which the short URL returns an error instead of redirecting.

# Non-Functional Requirements
**Non-functional requirements** define the system's qualities: scalability, latency, durability, security, fault tolerance, and consistency guarantees. The approach is to systematically evaluate each quality and determine which are uniquely relevant to this system.
## Low Latency on Redirects
**The system should provide redirects with latency under 200 milliseconds.** This threshold represents the human perception of real-time response—anything faster appears instantaneous. Low latency is critical specifically for the redirect operation, as users expect immediate navigation.
## Scale
Assume the system must support **100 million daily active users** and **1 billion URLs** over its lifetime. These figures inform capacity planning and architectural decisions throughout the design.
## Uniqueness of Short Codes
**The system must ensure uniqueness of short codes.** Collisions would cause users to be redirected to unexpected destinations, representing a critical failure mode. This requirement is non-negotiable.
## CAP Theorem Considerations
**The CAP theorem** states that distributed systems can **guarantee only two of three properties: consistency, availability, and partition tolerance**. Since partition tolerance is mandatory in distributed systems (which system design interviews almost always require), the real choice is between strong consistency and high availability.

**Strong read-after-write consistency** means every read must return the most recently written value, or the system fails. Banking applications and ticket booking systems require this property—if two users book the last Taylor Swift concert ticket due to stale reads, the system has failed catastrophically.

**For URL shorteners, strong consistency is not required.** When a user creates a short URL, some delay before it becomes globally accessible is acceptable. Users need time to share the URL before others attempt to use it. Even if someone attempts to use a brand-new short URL within seconds of creation, a temporary error message is far less severe than consistency violations in financial or booking systems.

Therefore, this system prioritizes **high availability** and can tolerate **eventual consistency** for URL creation.
## A Note on Back-of-the-Envelope Calculations
Many guides recommend performing capacity estimations upfront—calculating storage requirements, query-per-second estimates, and bandwidth projections. However, **performing calculations without a specific decision to inform is often wasteful**. With 100 million daily active users and 1 billion URLs, the conclusion is obvious: this will be a large distributed system. The real value of calculations emerges during design phases when they directly inform architectural choices, such as determining whether to shard a database or whether a cache layer is necessary.

# Core Entities
The core entities represent the **data structures persisted in the database** and **exchanged via APIs**. For a URL shortener:
1. **Original URL** (the long URL provided by users)
2. **Short URL** (the generated short code)
3. **User** (the entity that creates short URLs)

At this stage, avoid detailing the complete data model. For simple systems like this, the full schema may be obvious, but for complex systems, premature schema design can be counterproductive. The data model becomes clearer after defining the API and understanding all requirements.

# API Design
The API defines the contract between clients and the backend service. Map each functional requirement to its corresponding endpoint methodically.
## Create Short URL
```
POST /urls
```

**Request body:**
- `original_url` (required)
- `custom_alias?` (optional)
- `expiration_time?` (optional)

**Response:**
- Returns a `short_url` object

This endpoint accepts a long URL and optional parameters, returning the generated short URL. When using REST APIs, ensure proper HTTP verbs: POST for creation, PUT or PATCH for updates, DELETE for deletion, and GET for retrieval. Use plural nouns for resource paths (e.g., `/urls` not `/url`). While these conventions may seem pedantic, some interviewers care deeply about RESTful principles.
## Redirect to Original URL
```
GET /{short_url}
```

**Response:**
- Returns a redirect to the `original_url`

When a client issues this GET request with a short code, it hits the URL shortener service, which looks up the corresponding long URL and issues a redirect response. The client then navigates to the original destination.

# High-Level Design
With requirements, entities, and APIs defined, the next phase involves diagramming the system to satisfy the core functional requirements. Work through each API endpoint methodically, drawing the components necessary to handle each request.
## Architecture Overview
The initial architecture consists of three components:
1. **Client** (e.g., a web browser)
2. **Primary Server** (the main compute resource)
3. **Database** (for persistence)

This represents the classic client-server-database pattern.
## Data Model
Rather than maintaining separate tables for short and long URLs, combine them into a single unified schema:

**URL Table:**
- `short_url` (primary key)
- `original_url`
- `creation_time`
- `user_id` (foreign key)
- `custom_alias?`
- `expiration_time?`

**User Table:**
- `user_id`
- (plus additional metadata: email, password hash, salt, etc.)

Avoid getting distracted by auxiliary user table fields. Focus on the core functionality.
## URL Creation Flow
When a user creates a short URL:
1. Client sends `POST /urls` with the original URL to the primary server
2. Server generates a short code (implementation details deferred)
3. Server stores the mapping in the database (short URL → original URL, along with creation time and user ID)
4. Server returns the short URL to the client

![[Pasted image 20260518235803.png]]

If the user provides a custom alias, the server first checks whether it already exists. If available, it uses the custom alias; otherwise, it returns an error. If an expiration time is provided, it's stored alongside the URL mapping.
## Redirect Flow
When a user accesses a short URL:
1. Client sends `GET /{short_url}` to the primary server
2. Server queries the database using the short URL as the key
3. Database returns the corresponding original URL
4. Server checks if the URL has expired (if an expiration time exists)
5. If not expired, server returns a **302 redirect** to the original URL
6. Client automatically navigates to the original URL

![[Pasted image 20260518235908.png]]

## HTTP Redirect Status Codes: 301 vs 302
**301 (Permanent Redirect):** Indicates the redirect is permanent. Clients, browsers, and DNS servers may cache this redirect, meaning future requests might bypass the URL shortener service entirely.

**302 (Temporary Redirect):** Indicates the redirect is temporary. Requests always go through the URL shortener service, which can log each redirect.

The choice between 301 and 302 involves trade-offs. The 301 redirect reduces server load and scales better since cached redirects don't hit the backend. However, it eliminates visibility into redirect usage. Without analytics requirements, this loss of observability is still problematic—even for internal monitoring, understanding traffic patterns is essential. **If redirect counts suddenly drop, this could indicate a client-side failure.** Without requests hitting the server, such issues become invisible. **Therefore, a 302 redirect is preferred**, as it maintains observability while accepting the cost of additional server requests. This trade-off prioritizes operational visibility over marginal efficiency gains.

At this point, the high-level design satisfies all functional requirements. The system can create short URLs (with optional custom aliases and expiration times) and redirect users to original URLs. Now it's time to address the non-functional requirements through targeted deep dives.

# Deep Dive: Ensuring Uniqueness of Short Codes
The high-level design intentionally abstracted away the core challenge: generating short codes. This black-box approach prevented premature optimization and ensured all functional requirements were addressed first. Now, examine this critical component in detail.
## Requirements for Short Code Generation
A short code generation mechanism must satisfy three properties:
1. **Fast** – Generation should be rapid
2. **Unique** – No collisions are acceptable
3. **Short** – Codes should be 5-7 characters long
## Approach 1: URL Prefix (Inadequate)
Taking the first 5-7 characters of the long URL would be fast and short, but fails the uniqueness requirement spectacularly. Thousands of URLs share common prefixes (every Twitter URL starts with `www.twitter.com`, every Facebook URL starts with `www.facebook.com`). This creates a one-to-many mapping, making it impossible to determine which long URL a given short code represents. **This approach is fundamentally flawed.**
## Approach 2: Random Number Generator with Base-62 Encoding
Generate a random number within a sufficiently large range, then encode it using base-62 to create a compact representation.

**Base-62 encoding** uses digits 0-9 (10 options), uppercase A-Z (26 options), and lowercase a-z (26 options), giving 62 possible values per character position. With a 6-character code, this yields 62^6 ≈ 56 billion possible combinations—far exceeding the 1 billion URL requirement.

However, collision probability is surprisingly high due to the **birthday paradox**. Just as 23 people in a room create a >50% chance of shared birthdays (despite 365 days in a year), generating 1 billion random values from a 56 billion space results in approximately **880,000 collisions**.

To handle collisions, check the database before insertion: query whether the generated short code already exists, and only write if it's unique. This adds a database read operation to the creation flow but guarantees uniqueness.
## Approach 3: Hashing the Long URL
Apply a hash function (MD5, MurmurHash, SHA-256) to the long URL, base-62 encode the result, and take the first 6 characters.

Despite being deterministic, hash functions produce pseudo-random outputs due to the avalanche effect (changing one input bit dramatically alters the entire hash). **This approach produces identical collision probabilities as the random number generator**—both yield approximately 880,000 collisions from 1 billion URLs. The collision check requirement remains.
## Approach 4: Sequential Counter with Base-62 Encoding (Preferred)
Assign the first URL code 0, the second code 1, the third code 2, and so on. Apply base-62 encoding to keep codes compact. **This guarantees zero collisions** through sequential assignment while meeting all three requirements: fast, unique, and short.

The sequential counter grows from 0 → 9 → A → Z → a → z → 10 (in base-62), reaching 56 billion combinations with 6 characters before requiring a 7th character.
### Security Concern: Predictability
Sequential codes are predictable, creating two security issues:
1. **Competitors can determine total URL count** by incrementing codes until no valid short URL is found
2. **Competitors can scrape all long URLs** by systematically requesting every sequential short code

Mitigation strategies include:
- **User warnings:** Inform users not to shorten private URLs
- **Rate limiting:** Prevent mass scraping attempts
- **Bijective functions:** Use libraries like Sqids to create one-to-one mappings that appear random while maintaining deterministic encoding/decoding

Bijective functions obfuscate the sequential nature without introducing collision risks, making them the optimal solution when security concerns exist.
## Implementation Decision
**The sequential counter with base-62 encoding (optionally with bijective obfuscation) is recommended** because it eliminates the database collision check required by random/hash approaches, reducing latency and database load.

The counter itself will need special handling when the system scales horizontally (addressed in the scaling deep dive).

# Deep Dive: Low Latency on Redirects
The target is sub-200ms redirects. The current bottleneck is the database lookup: querying 1 billion rows to find a matching short URL.

### Optimization 1: Database Indexing

Without indexing, the database performs a full table scan—examining every row sequentially. **Indexing dramatically improves lookup performance.**

Set `short_url` as the **primary key**, which provides two benefits:

1. **Enforces uniqueness** at the database level
2. **Automatically creates an index** on that column

Most relational databases (like PostgreSQL) implement primary key indexes using **B-trees** (self-balancing tree structures). B-trees provide O(log n) lookup time, though modern database optimizations often achieve better performance for equality checks.

Some systems support **hash indexes** for O(1) lookups. However, modern B-tree implementations are so optimized that the practical difference is negligible for this use case.

Indexing reduces database query time significantly, but still requires disk access. Further optimization is possible.

### Optimization 2: In-Memory Cache

Introduce an in-memory cache (Redis or Memcached) to avoid disk access entirely. The cache acts as a **read-through, least-recently-used (LRU) cache**:

**Read-through behavior:**

- First check the cache for the short URL
- On cache miss, query the database, update the cache, then return the result
- On cache hit, return immediately from memory (O(1) lookup)

**LRU eviction policy:**

- When the cache fills, evict the least recently used entries
- This matches URL shortener usage patterns: recent URLs are accessed frequently, but old URLs see declining traffic
- Evicted URLs experience slower first access (cache miss + database query), but subsequent accesses are fast again

The cache stores simple key-value pairs: `{short_url: original_url}`. This provides extremely fast lookups while the database maintains the source of truth.

### Optimization 3: CDN Caching (Optional)

A **Content Delivery Network (CDN)** consists of geographically distributed edge servers. A user in China requesting a short URL hosted in California experiences significant latency. Edge servers in China could cache redirect responses, eliminating the round-trip to California.

However, **CDN caching has the same drawbacks as 301 redirects**: requests never reach the origin server, eliminating observability. If analytics or internal monitoring matter, avoid CDN caching. If pure performance is paramount and observability can be sacrificed, combine CDN caching with 301 redirects for maximum efficiency.

**Recommendation:** Use in-memory caching (Redis) with 302 redirects to balance performance and observability.

## Deep Dive: Scaling to Support 100M Daily Active Users

Identify bottlenecks by analyzing each system component from left to right.

### Scaling the Primary Server

**Calculate request load:**

- 100 million daily active users (10^8)
- Assume 1 redirect per user per day
- Seconds per day ≈ 100,000 (10^5)
- Requests per second: 10^8 ÷ 10^5 = 10^3 = 1,000 RPS

For peak traffic, multiply by 10-100×, yielding **10,000 to 100,000 requests per second at peak**.

A standard EC2 instance (e.g., t3.medium) handles approximately 1,000 concurrent requests (this is intentionally approximate—actual capacity depends on computation complexity, memory usage, payload size, and bandwidth). **A single server cannot handle peak load.**

#### Scaling Options

**Vertical scaling** means upgrading to a larger instance with more CPU, memory, and bandwidth. This is expensive and has limits.

**Horizontal scaling** means deploying multiple identical instances with requests distributed across them. This is the standard approach and provides near-linear scaling.

**However, read and write traffic differ significantly.** If 100 million users perform redirects daily but only 100,000 create new short URLs, write traffic is 1,000× lower (approximately 1 write request per second). **These workloads don't need identical scaling.**

### Microservices Architecture

Split the system into separate services:

1. **Read Service** – Handles redirects (`GET /{short_url}`)
2. **Write Service** – Handles URL creation (`POST /urls`)

Each service scales independently based on its traffic profile. The read service deploys many instances to handle high redirect volume, while the write service deploys fewer instances.

**API Gateway:** Introduce an API gateway as the entry point. It routes requests to the appropriate service based on the endpoint. The gateway determines whether each request targets the read or write service and forwards accordingly.

The read service queries the cache and database (read-through pattern), while the write service generates short codes and persists URL mappings. Both services scale horizontally with auto-scaling configurations.

#### Auto-Scaling

Modern cloud platforms (AWS, Google Cloud, Azure) handle scaling automatically. Configure resource thresholds such as:

- "If cluster memory usage exceeds 75%, add an instance"
- "If CPU usage drops below 20%, remove an instance"

A **load balancer** distributes requests across available instances using round-robin or other algorithms. The infrastructure automatically adjusts capacity based on real-time demand.

#### Microservices Trade-off

While separating read and write services allows independent scaling, the URL shortener has minimal business logic in either service. **Maintaining two separate codebases introduces operational overhead.** For such simple services, the complexity may not be justified. This is a valid trade-off to discuss in interviews—neither approach is definitively superior.

### Scaling the Global Counter

With multiple write service instances, a counter stored in local memory would create collisions—each instance would generate duplicate codes (both start at 0, then 1, then 2, creating duplicates).

**Solution: External Global Counter**

Move the counter to an external service (e.g., Redis) that all write service instances query. Redis provides an `INCR` command that atomically increments a counter. Since Redis is single-threaded, concurrent requests are serialized, guaranteeing each instance receives a unique sequential number.

**Optimization: Batch Allocation**

Requesting a counter value for every URL creation adds a network round-trip. Instead, **pre-allocate blocks of 1,000 IDs** when a write service instance starts. The instance uses its local allocation until exhausted, then requests the next block. If an instance crashes before using all allocated IDs, those values are lost—but with 3.5 trillion possible 6-character codes, losing occasional thousands is inconsequential.

### Scaling the Database

**Estimate storage requirements:**

- Short code: 8 bytes
- Long URL: ~100 bytes
- Creation time: 8 bytes
- Custom alias: ~100 bytes
- Expiration time: 8 bytes
- User ID: 8 bytes

Total: ~232 bytes per record, rounded to 500 bytes for additional fields.

**Total storage:** 500 bytes × 1 billion records = 500 GB

Modern SSDs provide hundreds of terabytes of capacity. **500 GB fits comfortably on a single database instance.** Storage is not a bottleneck.

**Read throughput:** The cache (Redis) absorbs most read traffic. Cache hits never touch the database, and cache misses only query on first access (then populate the cache). **Database read load is minimal.**

**Conclusion:** A single database instance suffices. If scaling were necessary, shard by `short_url` using modulo hashing (e.g., `short_url % 3` distributes across 3 database instances).

## Deep Dive: High Availability

Horizontal scaling and auto-scaling provide inherent availability—if one instance fails, others continue serving traffic.

### Redis High Availability

**Cache Redis:** If the read-through cache fails, the system degrades gracefully. All requests become cache misses and hit the database, increasing latency temporarily. When Redis restarts, the cache repopulates organically through read-through behavior. This is acceptable; no special redundancy is required.

**Global Counter Redis:** If the counter service fails, **URL creation stops entirely**—a critical failure. Implement Redis high availability modes:

- **Replication:** Maintain standby replicas that take over if the primary fails
- **Periodic snapshots:** Save counter state to persistent storage (e.g., S3) every few minutes to enable recovery

### Database High Availability

**Replication:** Maintain at least one read replica. If the primary database fails, promote the replica to primary. This minimizes downtime.

**Periodic snapshots:** Backup the entire database state to blob storage (e.g., S3) hourly. If catastrophic failure occurs, restore from the most recent snapshot. Data created since the last snapshot is lost, but the system recovers quickly.

## Conclusion

The system now satisfies all requirements:

**Functional Requirements:**

- ✓ Create short URLs from long URLs
- ✓ Redirect users to original URLs
- ✓ Support custom aliases
- ✓ Support expiration times

**Non-Functional Requirements:**

- ✓ Sub-200ms redirect latency (via indexing and caching)
- ✓ Scale to 100M daily active users and 1B URLs (via horizontal scaling, microservices, and auto-scaling)
- ✓ Guarantee uniqueness of short codes (via sequential counter with base-62 encoding)
- ✓ High availability with eventual consistency (via replication, snapshots, and distributed architecture)

This systematic approach—defining requirements, building a simple design to satisfy functional requirements, then iteratively addressing non-functional requirements through deep dives—provides a clear roadmap for any system design interview.