# System Design Case Studies — Full Walkthroughs

> Each case study follows: requirements → estimation → high-level design → deep dive → tradeoffs. Use these as model answers/structure, not scripts to memorize verbatim — interviewers want to see your reasoning.

---

## 1. Design a URL Shortener (e.g., bit.ly)

**Requirements**: Shorten a long URL to a short one; redirect short → long with low latency; custom aliases (optional); analytics (optional); links may expire.

**Estimation**: 100M new URLs/month ≈ 40 writes/sec average. Read:write ratio for a shortener is typically heavily read-skewed (~100:1), so ≈ 4,000 reads/sec. Storage: 100M URLs/month × 500 bytes/record × 5 years ≈ a few TB — modest; this is a compute/latency problem, not a storage-volume problem.

**High-level design**: Client → Load Balancer → App servers → (1) a key-generation service, (2) a key-value store (short code → long URL) → Cache (Redis) in front of the store for hot redirects → optional async analytics pipeline (Kafka → analytics DB) fed by redirect events, not on the critical redirect path.

**Deep dive — generating unique short codes**: Option A: hash the long URL (MD5/SHA) and take the first 6-7 characters, base62-encode — simple, but collisions are possible and must be detected/handled (check-and-retry with a salt, or append a counter). Option B: a centralized counter/ID generator (see the distributed unique ID case study) that hands out unique, monotonically increasing IDs, base62-encoded into the short code — guarantees no collisions, but the ID generator must itself be highly available and not a bottleneck (typically solved via pre-allocated ID ranges handed out to app servers in batches, so most ID assignment happens locally without a round trip per request). Most real designs prefer Option B for its collision-free guarantee.

**Deep dive — redirect latency**: A redirect is on the user's critical path (they clicked a link and are waiting), so this read path must be fast: cache the short-code → long-URL mapping aggressively (Redis, extremely high cache hit ratio expected since a small fraction of links get the vast majority of clicks — see hot-key discussion in the fundamentals file), and use a `301` (permanent, cacheable by the browser/CDN itself, reducing repeat-click server load) vs. `302` (temporary, needed if you want every click to be trackable server-side for analytics) redirect — a real tradeoff between caching efficiency and analytics completeness worth stating explicitly.

**Tradeoffs to mention**: 301 vs 302 (caching vs. analytics), whether to support custom aliases (adds a uniqueness-check requirement, see the data storage file's global-uniqueness question), and whether/how to handle expired or deleted links (a background job cleaning up expired entries, or lazy deletion checked at redirect time).

---

## 2. Design a Rate Limiter (as a standalone system/service)

**Requirements**: Limit each client to N requests per time window; low latency (must not meaningfully slow down normal requests); accurate enough to actually protect backend capacity; works across multiple app server instances consistently.

**High-level design**: Implemented as middleware at the API gateway/edge (see the fundamentals file — enforce as early as possible). Backed by a centralized, fast, shared store (Redis) so the limit is enforced consistently regardless of which app instance handles a given request.

**Deep dive — algorithm choice**: (See the networking file's full explanation of token bucket/leaky bucket/fixed/sliding window.) For an interview, implement sliding-window-counter as a good practical middle ground: maintain a counter per client per fixed window (e.g., per minute) plus a weighted contribution from the previous window's counter proportional to how much of the current window has elapsed — approximates a true sliding window's smoothness without the memory cost of storing every individual request timestamp (which a true sliding-log approach would require, at higher memory cost per client).

**Deep dive — making the Redis check atomic under high concurrency**: A naive "read counter, check if under limit, increment" from the application is a race condition (see backend fundamentals file) under concurrent requests from the same client hitting different app servers simultaneously. Use a Redis Lua script (executed atomically by Redis itself) to perform the check-and-increment as one atomic server-side operation, or Redis's native `INCR` (atomic) combined with `EXPIRE` set only on the first increment of a window.

**Tradeoffs**: A single centralized Redis is simple but becomes a bottleneck/SPOF at extreme scale — mitigated by sharding rate-limit keys across a Redis Cluster (by client ID). Discuss the choice between hard rejection (429) vs. throttling/queuing — and whether the limit should be global (all endpoints share one budget) or per-endpoint (finer-grained, more complex to configure/reason about).

---

## 3. Design a Real-Time Chat Application (e.g., WhatsApp/Slack-like)

**Requirements**: 1:1 and group messaging; near-real-time delivery; message persistence/history; online/offline/typing status; delivery/read receipts (optional); support for millions of concurrent connections.

**High-level design**: Clients maintain a persistent WebSocket connection to a "connection/gateway server" layer (many instances, each holding a large number of live connections). A message from sender → gateway server → message service (validates, persists to a message store) → looks up which gateway server(s) the recipient(s) are currently connected to (via a connection-registry service, e.g., backed by Redis mapping user ID → gateway server instance) → forwards the message to that specific gateway server → pushed down the recipient's live WebSocket. If the recipient is offline, the message is queued/stored and a push notification is triggered instead; delivered on next connect.

**Deep dive — the connection registry problem**: Since a user can be connected to any one of many gateway server instances (and that mapping changes as they connect/disconnect/reconnect, including during a gateway server's own deploy/restart), a fast, shared, low-latency lookup (user ID → current gateway instance, or "offline") is essential and sits on the hot path of every message send — Redis is a natural fit given the need for fast reads/writes and the acceptable (indeed necessary) property that this data is inherently ephemeral/derived, not a system of record.

**Deep dive — message ordering and delivery guarantees**: Within a single conversation, messages should generally be delivered/displayed in the order sent — assign each message a sequence number (per-conversation, not global) so clients can detect/reorder out-of-order delivery and detect gaps (indicating a missed message needing re-fetch). For delivery guarantees, at-least-once delivery (see the message queues file) combined with a client-generated message ID for deduplication (avoiding a message appearing twice if a retry occurs after an ambiguous network failure) is the standard practical approach — true exactly-once is not required if idempotent client-side deduplication is in place.

**Tradeoffs**: WebSocket (full-duplex, persistent) vs. long-polling (simpler infra, higher latency/overhead) as the transport — WebSocket is standard for this use case given the two-way, high-frequency messaging need. Group chat fan-out at large group sizes (thousands of members) has the same "hot key" fan-out cost problem as the social feed case study — mitigations are analogous (fan-out on write for typical group sizes, special-cased handling for extremely large groups/channels).

---

## 4. Design a News Feed (e.g., Twitter/Facebook-like)

**Requirements**: Users post content; users see a feed of posts from people/pages they follow, roughly reverse-chronological or ranked; feed generation must be fast even for users following thousands of accounts; system must handle "celebrity" accounts with millions of followers.

**High-level design**: A post-creation service persists new posts to a posts store and publishes a "new post" event. A feed-generation approach combining fan-out-on-write (for most users) with fan-out-on-read (for celebrity accounts) — see the fundamentals file's dedicated question on this exact tradeoff — populates/serves a per-user, pre-computed feed cache (often in Redis, storing just post IDs per user, hydrated with full post content at read time from the posts store/cache).

**Deep dive — the celebrity/hot-fan-out problem**: (Elaborated fully in the fundamentals file, Q14.) The core design decision to state explicitly: define a follower-count threshold above which an account is treated specially — its posts are NOT fanned out to millions of individual follower feed lists on write (too expensive/slow); instead, a follower's feed is assembled at read time by merging their normal pre-computed feed with a live lookup of any celebrity accounts they follow's recent posts, adding a small amount of read-time work only for users who follow at least one celebrity, rather than imposing a massive write-time cost on every celebrity post regardless of whether anyone reads their feed soon after.

**Deep dive — ranking vs. chronological**: A purely chronological feed is simple to reason about and implement with the fan-out approach above. A ranked feed (ordering by predicted relevance/engagement rather than recency) requires a separate ranking/scoring step (often ML-model-based) applied at read time to the candidate set of recent posts — this is a legitimate, common follow-up direction in interviews; the key point to make is that ranking is a *read-time* concern layered on top of the same underlying candidate-generation (fan-out) infrastructure, not a replacement for it.

**Tradeoffs**: Feed staleness (fan-out-on-write feeds can lag slightly behind the absolute latest posts, an acceptable tradeoff for read performance) vs. read cost (fan-out-on-read is always fully fresh but expensive per read) — the hybrid approach is explicitly a pragmatic middle ground, and stating that explicitly is a strong interview signal.

---

## 5. Design a Notification System (push, email, SMS, in-app)

**Requirements**: Send notifications to users across multiple channels (push, email, SMS, in-app); support both a small number of individual notifications and huge bulk sends (e.g., "notify all users about an outage"); avoid duplicate/spam notifications; respect user preferences (opted out of certain channels/types); some notifications are time-critical, others can tolerate delay.

**High-level design**: A Notification Service exposes an API (or consumes internal events, e.g., "order shipped") that other services call/publish to request a notification. It looks up the user's preferences (which channels they've enabled for this notification type) and pushes the request onto a message queue, partitioned/prioritized by channel and urgency. Separate, independently-scalable worker pools per channel (a push-notification worker pool, an email worker pool, an SMS worker pool) consume from their respective queues and call the actual third-party provider (APNs/FCM for push, an email service, an SMS gateway).

**Deep dive — why per-channel queues and worker pools, not one shared pipeline**: Each channel has wildly different throughput characteristics and third-party rate limits (SMS providers often have strict per-second limits and high per-message cost; push notification services can handle much higher throughput) — a shared pipeline would mean a slow/rate-limited channel (SMS) creates backpressure that unnecessarily delays a fast channel (push) that shares the same queue, so isolating them (the Bulkhead pattern, see the microservices file) lets each channel be scaled and rate-limited independently according to its own actual constraints.

**Deep dive — deduplication and idempotency**: Since underlying business events can occasionally be published more than once (at-least-once delivery, see the message queues file), and retries can occur if a third-party provider call times out ambiguously, the Notification Service should generate/require an idempotency key per logical notification (e.g., `order_id + "shipped"`) and check against a store of recently-sent notification keys before actually dispatching, to avoid a user receiving the same "your order shipped" notification five times due to upstream retries.

**Tradeoffs**: Real-time push through the pipeline vs. batching (bulk notifications, like a mass outage announcement to millions of users, are usually deliberately throttled/rate-limited on dispatch — not because the queue can't hold them, but because sending millions of notifications instantly can itself overwhelm the notification service's own downstream third-party providers, or even the app's own backend if millions of users open the app simultaneously in response — a real "thundering herd" caused by the notification system's own success).

---

## 6. Design a Distributed Cache / Key-Value Store (e.g., a simplified Redis/DynamoDB)

**Requirements**: Store and retrieve key-value pairs with very low latency; scale horizontally across many nodes; remain available and (to a chosen degree) consistent under node failures; support a configurable replication factor.

**High-level design**: Data is partitioned across nodes using consistent hashing (see the fundamentals file) so nodes can be added/removed with minimal data reshuffling. Each key's data is replicated to N nodes (the "preference list" of nodes responsible for that key, per the hash ring). Reads/writes use tunable quorum consistency: a write succeeds once acknowledged by W replicas, a read is considered valid once R replicas respond (and, if they disagree, the most recent version — often determined via vector clocks or simple last-write-wins timestamps — is returned, with conflicting versions in some designs surfaced back to the application to resolve).

**Deep dive — the W+R > N quorum guarantee**: If W (write quorum) plus R (read quorum) exceeds N (total replicas), there's guaranteed to be at least one overlapping node between any write's replica set and any subsequent read's replica set, ensuring the read will see at least one copy reflecting the most recent write — this is the mathematical basis for "strong-ish" consistency in an otherwise eventually-consistent leaderless system, and tuning W and R lets you trade off between write availability/latency, read availability/latency, and consistency strength per your specific needs (e.g., W=1,R=N favors fast writes but requires all replicas for a guaranteed-fresh read; W=N,R=1 is the reverse).

**Deep dive — handling node failure and "hinted handoff"**: If a node responsible for a key is temporarily down when a write arrives, rather than rejecting the write (hurting availability), the write can be temporarily stored on a different, available node with a "hint" that it actually belongs to the down node — once the down node recovers, the hinted data is handed off to it — trading strict consistency (the data isn't actually on its "correct" node yet) for continued write availability during a transient failure, a deliberate AP-leaning design choice (see the CAP theorem question in the backend fundamentals file).

**Tradeoffs**: This entire design is explicitly an AP (available, partition-tolerant) system by default, favoring availability over strict consistency — contrast this explicitly with a CP alternative (e.g., a single-primary system that would reject writes during a partition rather than risk inconsistency) and state which is more appropriate depending on the stated use case (a shopping cart tolerates eventual consistency fine; a bank ledger typically would not accept this tradeoff without much more careful design).

---

## 7. Design a Web Crawler

**Requirements**: Crawl billions of web pages; respect `robots.txt` and politeness (rate limits per domain); avoid re-crawling duplicate content; prioritize important/frequently-changing pages for re-crawling; be resilient to crawler traps (infinite link loops) and malformed content.

**High-level design**: A URL Frontier (a prioritized, persistent queue of URLs to crawl next) feeds many parallel Fetcher workers, which download page content, pass it to a Content Processor (extracts links, deduplicates content, parses/indexes text), which extracts new links and feeds them back into the Frontier (after filtering already-seen/disallowed URLs) — a large-scale producer-consumer loop.

**Deep dive — politeness and per-domain rate limiting**: Crawling too aggressively against any single domain can be indistinguishable from a DoS attack and get the crawler IP-banned, so the Frontier must ensure requests to the same domain are spread out (a per-domain queue/delay, or partitioning fetcher work such that only a limited number of workers/requests are active against any single domain at once) rather than a naive design where many parallel fetchers could all simultaneously hit the same popular domain's many pages at once.

**Deep dive — avoiding duplicate crawling and infinite crawler traps**: Maintain a seen-URL set (a Bloom filter is a natural fit here — see the caching file's cache-penetration question for the same data structure used differently — to cheaply check "have we likely already crawled this URL" without needing to store every full URL in exact-match memory, accepting a small false-positive rate as a deliberate tradeoff for massive space savings at billions-of-URLs scale) and detect/limit content duplication via content hashing (two different URLs with identical or near-identical content shouldn't both be fully processed/indexed). Crawler traps (e.g., a calendar page that generates an infinite sequence of "next day" links) are mitigated with a maximum crawl depth per site and anomaly detection on URL patterns (e.g., a URL parameter that appears to be counting upward indefinitely).

**Tradeoffs**: Crawl freshness (how often to re-crawl a page for updates) vs. crawl breadth (crawling new, never-seen pages) — a priority scoring function (based on estimated page importance/PageRank-like signal, and estimated change frequency from crawl history) balances the Frontier's ordering between these two competing goals, since crawling capacity is always finite relative to the size of the web.

---

## 8. Design a Video Streaming Platform (e.g., YouTube-like upload & playback)

**Requirements**: Users upload videos; videos are processed into multiple resolutions/formats for adaptive streaming; playback must work smoothly across varying network conditions and device types; must scale to serve massive concurrent viewership for popular content.

**High-level design**: Upload → raw video stored in object storage (S3) → an async transcoding pipeline (triggered by the upload event) generates multiple resolution/bitrate variants (encoded via a processing cluster, often a fleet of workers pulling transcoding jobs from a queue, since transcoding is CPU-intensive and highly parallelizable per video) → transcoded output segments stored in object storage → a CDN caches and serves the video segments to viewers, using an adaptive bitrate streaming protocol (HLS/DASH) so the client's video player dynamically switches to a lower/higher bitrate variant based on currently observed network conditions.

**Deep dive — why async transcoding, and handling transcoding failures/retries**: Transcoding a video into multiple formats/resolutions is slow (potentially minutes for a long video) and must not block the upload response — it's queued and processed asynchronously, with the video's status (processing/ready/failed) tracked and surfaced to the uploader, and the actual transcoding job designed to be safely retryable (idempotent — re-running a transcode job for the same video/target-format should simply produce the same output, overwriting or being written to the same target key, if a worker crashes mid-job).

**Deep dive — CDN and adaptive bitrate streaming**: Video is split into short segments (a few seconds each) per resolution/bitrate variant, described by a manifest file the client reads; the client's player continuously monitors its actual download throughput and switches which bitrate variant it requests for the *next* segment accordingly — smoothing playback quality dynamically without needing to restart playback, and letting the CDN cache and serve small, independent segments (which cache far more effectively and flexibly than trying to cache one giant monolithic video file per resolution).

**Tradeoffs**: Transcoding cost/storage (storing many resolution variants of every video multiplies storage cost significantly) vs. playback quality/compatibility across devices/network conditions — many platforms transcode into a limited standard set of resolutions rather than every conceivable combination, and lower-popularity/rarely-watched videos might justify a cheaper transcoding strategy (fewer variants, or transcode-on-first-request rather than eagerly for every upload) than extremely popular content, an explicit cost/value tradeoff worth raising.

---

## 9. Design a Ride-Hailing Service (e.g., Uber-like matching)

**Requirements**: Riders request a ride; the system matches them to a nearby available driver; both parties see real-time location updates during the ride; pricing (including dynamic/surge pricing) is calculated; the system operates at city/regional scale with tight latency requirements for matching.

**High-level design**: Drivers' apps continuously report their current location (via a lightweight, frequent location-update call) to a Location Service, which maintains each driver's current position in a geospatial index (see deep dive). When a rider requests a ride, a Matching Service queries the geospatial index for nearby available drivers, applies matching logic (proximity, driver rating, estimated arrival time), and proposes the ride to a candidate driver (who can accept/decline, cascading to the next candidate if declined) — once accepted, both apps subscribe (via a real-time channel, similar to the chat system's connection/gateway pattern) to live location updates for the duration of the ride.

**Deep dive — geospatial indexing for "find nearby drivers" efficiently**: A naive approach (compute distance from the rider to every driver in the city) doesn't scale. Standard solution: divide the map into a grid (or use a hierarchical spatial indexing scheme like Geohash or Google's S2/Uber's own H3) — each driver's location update is converted to a grid cell ID, and finding "nearby drivers" becomes a fast lookup of drivers registered in the rider's cell and its immediate neighboring cells, rather than a distance calculation against every driver citywide. This grid-cell mapping is typically maintained in a fast in-memory store (Redis geospatial commands are a common practical implementation) given how frequently it must be read (every ride request) and written (every driver location update, which can be extremely high-frequency in aggregate across a whole city).

**Deep dive — handling the "accept/decline cascade" and avoiding double-booking a driver**: When a ride is proposed to a driver, that driver must be temporarily locked/marked unavailable for other ride proposals during their decision window (a short timeout, after which the system automatically cascades to the next candidate driver if there's no response) — this requires an atomic "propose and lock" operation (similar in spirit to the optimistic/pessimistic concurrency discussion in the backend fundamentals file) to guarantee two simultaneous ride requests can't both successfully lock and assign the same driver.

**Tradeoffs**: Real-time location update frequency vs. system load/battery drain (more frequent updates give better matching/tracking accuracy but cost more server load and drain the driver's phone battery faster — most real systems tune this dynamically, e.g., more frequent updates while actively in/near a ride-matching state, less frequent while idle). Matching purely by proximity vs. incorporating ETA (accounting for actual road routes/traffic, not straight-line distance) is a classic simplicity-vs-accuracy tradeoff worth explicitly naming.

---

## 10. Design a Distributed Unique ID Generator (e.g., Twitter's Snowflake)

**Requirements**: Generate unique IDs across many machines/services without coordination on every single ID generation (a fully centralized single-counter service would be both a bottleneck and a SPOF); IDs should ideally be roughly time-sortable (useful for pagination/ordering by creation time without a separate timestamp column).

**High-level design (Snowflake-style)**: Each ID is a single 64-bit integer composed of several bit-packed segments: a timestamp (milliseconds since a custom epoch — providing rough chronological sortability), a machine/worker ID (identifying which node generated this ID, avoiding collisions between different nodes generating IDs at the same millisecond), and a per-millisecond sequence number (allowing a single node to generate multiple unique IDs within the same millisecond, incrementing a local counter that resets each millisecond).

**Deep dive — why this avoids the need for coordination**: Because each node has its own pre-assigned, unique machine ID and generates the timestamp+sequence portion purely from its own local clock and local counter, no two nodes need to communicate with each other (or a central authority) at ID-generation time to guarantee global uniqueness — uniqueness is guaranteed structurally by the combination of (distinct machine ID) + (timestamp) + (local sequence number), making ID generation extremely fast (a purely local, in-memory operation) and horizontally scalable to as many nodes as needed with zero added coordination overhead.

**Deep dive — clock skew/clock going backward, a real operational hazard**: If a node's system clock is corrected backward (e.g., via NTP adjustment after drift, or a VM migration), it could theoretically generate a timestamp segment that's not strictly greater than one it already generated, risking a duplicate ID as the sequence counter also resets — production Snowflake-style implementations detect this explicitly (comparing the current timestamp to the last-used timestamp) and either wait/block briefly until the clock catches back up, or refuse to generate IDs until the clock issue resolves, rather than silently risking a collision.

**Tradeoffs**: Machine-ID bit-width limits the maximum number of concurrent ID-generating nodes (a fixed tradeoff decided upfront when designing the bit layout); sequence-number bit-width limits maximum IDs generatable per node per millisecond before it must wait for the next millisecond — both are capacity planning decisions made explicit in the bit-layout design, worth stating with actual numbers (e.g., a 10-bit machine ID supports 1,024 nodes; a 12-bit sequence supports 4,096 IDs/ms/node) to show quantitative reasoning.

---

## 11. Design a Search Autocomplete / Typeahead System

**Requirements**: As a user types, suggest completions/popular queries in real time (sub-100ms typically expected); suggestions should reflect overall popularity/trending terms; must handle very high read QPS (every keystroke can trigger a request) with a much lower write/update rate (aggregating popularity is not a real-time-per-keystroke concern).

**High-level design**: Historical/aggregated query logs are periodically (e.g., hourly or daily, not per-request) processed by an offline batch job to build/update a Trie (prefix tree) data structure, where each node represents a character and stores the top-K most popular completions for that prefix. This pre-built Trie is loaded into fast, in-memory serving nodes; a user's partial input is looked up directly against the Trie for near-instant top-K suggestion retrieval, with the Trie itself replicated/cached across many read-serving nodes to handle the very high read QPS.

**Deep dive — why building the Trie is a separate offline/batch process, not done in real time per request**: Recomputing "top-K completions for every possible prefix" is an expensive aggregation over the entire query log; doing this live on every keystroke would be far too slow, and recency of trending terms doesn't need to reflect the literal current second — updating the popularity ranking every few minutes to hourly is a perfectly acceptable freshness tradeoff for a feature whose entire value is speed, not perfect real-time accuracy — decoupling the expensive aggregation (offline, batch, infrequent) from the fast lookup (online, per-keystroke, must be sub-100ms) is the core architectural insight of this design.

**Deep dive — sharding the Trie at massive scale**: If the full Trie is too large for a single machine's memory, it can be sharded (e.g., by first character(s) of the prefix, or a hash of the prefix) across multiple nodes, with a routing layer directing each request to the correct shard based on the user's current input prefix — a real tradeoff versus simply keeping the whole Trie in memory on every serving node (simpler, no routing needed, but requires the entire structure to fit in memory on every single node, limiting how large the underlying dataset/vocabulary can practically grow).

**Tradeoffs**: Personalized suggestions (incorporating a specific user's own search history) vs. purely global popularity — personalization adds real complexity (a separate per-user signal that must be merged with the global Trie's suggestions at request time) and is a natural "if we had more time" extension to mention rather than a required baseline, showing awareness of the feature's realistic evolution path without over-engineering the initial design.

---

## 12. Design an E-Commerce Order & Inventory System

**Requirements**: Users browse products and place orders; inventory must never be oversold (can't sell more units than actually in stock); orders involve multiple steps (payment, inventory reservation, shipping) that must be handled consistently even if any individual step fails; must handle flash-sale-level traffic spikes for popular items.

**High-level design**: A Product/Catalog service (read-heavy, heavily cached, can tolerate eventual consistency for things like "product description" but not for real-time stock count) is separate from an Order service and an Inventory service, each owning their own data. Placing an order triggers a multi-step process (Saga pattern, see the message queues file): reserve inventory → process payment → confirm order → trigger fulfillment/shipping — with compensating actions (release the inventory reservation) if any step after it fails.

**Deep dive — preventing overselling under high concurrency (the flash-sale problem)**: This is fundamentally the same read-then-write race condition discussed in the backend fundamentals file, but at extreme concurrency for a single hot item during a flash sale. Solution: use an atomic, database-level conditional decrement (`UPDATE inventory SET stock = stock - 1 WHERE product_id = ? AND stock > 0`, checking the affected-row count to know if it actually succeeded) rather than a naive read-then-write from the application — and at truly extreme concurrency for a single item, consider funneling all reservation attempts for that specific hot item through a single-threaded/serialized queue or an in-memory atomic counter (Redis `DECR`, checked against zero) specifically to avoid database lock contention becoming the bottleneck when many thousands of concurrent requests are all trying to atomically decrement the exact same row simultaneously.

**Deep dive — the Saga/compensating-transaction flow for a multi-step order**: Since inventory reservation, payment processing, and order confirmation may be handled by separate services (each with their own database, precluding a single ACID transaction spanning all of them), each step publishes a success/failure event that drives the next step (orchestrated, typically, by a dedicated Order Orchestrator service for clarity — see the message queues file's choreography-vs-orchestration question); a failure at any step (e.g., payment declined after inventory was already reserved) triggers an explicit compensating action (release the previously-reserved inventory back to available stock) rather than relying on any kind of automatic distributed rollback, which doesn't exist across independent databases.

**Tradeoffs**: Reserving inventory optimistically at "add to cart" time (better user experience — an item in your cart won't sell out from under you) vs. only at actual checkout/payment time (simpler, avoids inventory being tied up by abandoned carts, but risks a worse user experience if an item sells out between adding to cart and completing checkout) — most real e-commerce systems use a time-limited reservation (inventory held for a short window, e.g., 15 minutes, then automatically released if checkout isn't completed) as a deliberate middle-ground tradeoff between these two extremes.
