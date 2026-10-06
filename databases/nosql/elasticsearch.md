# Elasticsearch / OpenSearch Deep Dive

> A search engine is a **derived data store**: it's fed from your system of record and optimized for text relevance, filtering, and aggregations. Treat it as rebuildable.

## Table of Contents
1. [Elasticsearch vs OpenSearch vs Solr](#landscape)
2. [Core Concepts: Index, Document, Mapping, Shard, Replica, Segment](#core)
3. [How Full-Text Search Works: Inverted Index and Analysis](#inverted)
4. [Relevance Scoring: TF-IDF → BM25](#relevance)
5. [Mappings: text vs keyword, Dynamic Mapping Pitfalls](#mappings)
6. [Query DSL Essentials](#queries)
7. [Aggregations](#aggregations)
8. [Write Path: Refresh, Flush, Translog, Merges (Near-Real-Time)](#writepath)
9. [Cluster Architecture, Shard Sizing, and Node Roles](#cluster)
10. [Keeping Search in Sync with the Database](#sync)
11. [Pagination, Autocomplete, Typo Tolerance, Synonyms](#features)
12. [Logs and Observability Use Case (ELK)](#logs)
13. [Vector / Semantic / Hybrid Search](#vector)
14. [Operations and Pitfalls](#ops)
15. [Interview Questions](#qa)

---

## 1. Landscape {#landscape}

- **Elasticsearch** (Elastic, built on Apache Lucene, 2010). Licensing moved to SSPL/Elastic License in 2021, and AGPL was added as an option in 2024.
- **OpenSearch**: the AWS-led fork of ES 7.10 (Apache 2.0), now under the Linux Foundation. Mostly API-compatible for core features, diverging in plugins and newer features.
- **Apache Solr**: also Lucene-based; older, strong in enterprise search.
- Lighter alternatives: **Meilisearch**, **Typesense** (instant search, typo tolerance, simple ops), **Vespa** (large-scale ranking/ML), **Quickwit** (log search on object storage), **ParadeDB** (BM25 inside Postgres), **Algolia** (SaaS).

---

## 2. Core Concepts {#core}

| Concept | Meaning |
|---|---|
| **Index** | A collection of documents with a mapping (≈ table) |
| **Document** | A JSON object with `_id` |
| **Mapping** | Schema: field types and analyzers |
| **Shard** | An index is split into N primary shards, each a full **Lucene index** |
| **Replica** | Copy of a primary shard on another node (HA + read throughput) |
| **Segment** | Immutable mini-index inside a Lucene index; shards consist of many segments merged over time |
| **Node** | One ES process; roles: master-eligible, data (hot/warm/cold/frozen), ingest, coordinating, ml |
| **Cluster** | Nodes sharing a cluster state, managed by an elected master |
| **Alias** | A pointer to one or more indices (zero-downtime reindexing, rollover) |
| **Data stream** | Append-only time-series abstraction over rolling backing indices (logs, metrics) |

Primary shard count is fixed at index creation (changing it requires a reindex or the split/shrink APIs). Replica count is dynamic.

---

## 3. Inverted Index and Analysis {#inverted}

An **inverted index** maps each **term** to a **postings list**: the documents (and positions and frequencies) containing it.
```
"quick"  → [doc1(pos 2), doc3(pos 1)]
"fox"    → [doc1(pos 4), doc2(pos 2)]
```
Plus doc values (columnar storage for sorting/aggregations), stored fields (`_source`), BKD trees (numbers, dates, geo), and HNSW graphs (vectors).

**Analysis** happens at index time and at query time (it must match):
```
"The Quick-Brown Foxes!"
  → char filters (strip HTML, map chars)
  → tokenizer (standard: ["The", "Quick", "Brown", "Foxes"])
  → token filters (lowercase → stop words → stemming): ["quick", "brown", "fox"]
```
Built-in analyzers: `standard`, `simple`, `whitespace`, `keyword` (no tokenization), language analyzers (`english`, `hindi`, …), plus custom ones (`edge_ngram` for autocomplete, `synonym_graph`, `asciifolding`, `icu_*` plugins, phonetic).

Test analyzers with `POST _analyze { "analyzer": "english", "text": "Running foxes" }`.

---

## 4. Relevance: BM25 {#relevance}

Default similarity: **BM25** (Okapi Best Match 25).
- **Term frequency (TF)**: more occurrences in a doc means more relevant, with **saturation** (parameter `k1` ≈ 1.2): the 10th occurrence adds little.
- **Inverse document frequency (IDF)**: rare terms count more ("postgres" > "the").
- **Field-length normalization** (`b` ≈ 0.75): a match in a short field (title) counts more than in a long body.

Tuning relevance: field boosts (`title^3`), `multi_match` types (best_fields, most_fields, cross_fields), `function_score` / `script_score` (boost by popularity, recency decay functions), `rank_feature` fields, **learning to rank** (plugin), and rescoring. Use `"explain": true` to see why a document scored as it did.

IDF is computed **per shard** by default, so small indices with many shards can give odd scores (`search_type=dfs_query_then_fetch` fixes it at a cost).

---

## 5. Mappings {#mappings}

```json
PUT products
{
  "settings": { "number_of_shards": 3, "number_of_replicas": 1,
    "analysis": { "analyzer": { "autocomplete": { "tokenizer": "standard", "filter": ["lowercase", "edge_ngram_2_15"] } },
                  "filter": { "edge_ngram_2_15": { "type": "edge_ngram", "min_gram": 2, "max_gram": 15 } } } },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name":      { "type": "text", "analyzer": "english",
                     "fields": { "raw": { "type": "keyword" }, "ac": { "type": "text", "analyzer": "autocomplete", "search_analyzer": "standard" } } },
      "sku":       { "type": "keyword" },
      "price":     { "type": "scaled_float", "scaling_factor": 100 },
      "category":  { "type": "keyword" },
      "in_stock":  { "type": "boolean" },
      "created_at":{ "type": "date" },
      "location":  { "type": "geo_point" },
      "variants":  { "type": "nested", "properties": { "color": { "type": "keyword" }, "size": { "type": "keyword" } } },
      "embedding": { "type": "dense_vector", "dims": 768, "index": true, "similarity": "cosine" }
    }
  }
}
```
- **`text`**: analyzed, for full-text search, not for sorting or aggregations.
- **`keyword`**: exact value, for filters, sorting, aggregations, and IDs. Use a **multi-field** (`name.raw`) to have both.
- Numbers, dates, booleans, `ip`, `geo_point`/`geo_shape`, `flattened` (whole object as keywords), `nested` (arrays of objects queried independently), `join` (parent/child, slow), `dense_vector`/`sparse_vector`, `search_as_you_type`, `completion`.
- **Arrays of objects without `nested`** are flattened: `variants: [{color: red, size: S}, {color: blue, size: M}]` matches `color=red AND size=M` (wrong!). Use `nested` (with its query and indexing cost) when object boundaries matter.
- **Dynamic mapping pitfalls**: the first document's value type wins (a `"123"` string makes the field `text` + `keyword`). Unbounded dynamic keys cause **mapping explosion** (`index.mapping.total_fields.limit` = 1000). Use `"dynamic": "strict"` or `false` or runtime fields for user-generated keys, and index templates.
- **Mappings can't change type in place.** Create a new index and **reindex**, then swap the **alias** (zero downtime).

---

## 6. Query DSL Essentials {#queries}

```json
GET products/_search
{
  "query": {
    "bool": {
      "must":   [ { "multi_match": { "query": "running shoes", "fields": ["name^3", "description"], "fuzziness": "AUTO" } } ],
      "filter": [ { "term": { "category": "footwear" } },
                  { "range": { "price": { "gte": 1000, "lte": 5000 } } },
                  { "term": { "in_stock": true } } ],
      "should": [ { "term": { "brand": "acme" } } ],
      "must_not": [ { "term": { "discontinued": true } } ]
    }
  },
  "sort": [ "_score", { "created_at": "desc" } ],
  "from": 0, "size": 20,
  "_source": ["name", "price", "sku"],
  "highlight": { "fields": { "name": {} } }
}
```
- **Query context** (`must`, `should`) affects the score. **Filter context** (`filter`, `must_not`) is yes/no, **cached**, and faster. Put non-scoring conditions in `filter`.
- Full-text: `match`, `match_phrase` (word order with `slop`), `multi_match`, `query_string`/`simple_query_string` (user syntax; prefer simple to avoid parse errors).
- Term-level (no analysis): `term`, `terms`, `range`, `exists`, `prefix`, `wildcard` (slow with leading wildcard), `regexp`, `ids`, `fuzzy`.
- `nested`, `has_child`/`has_parent`, `geo_distance`, `knn`.
- Common mistake: a `term` query on a `text` field. `"Running Shoes"` is indexed as `running`, `shoes`, so the term `"Running Shoes"` matches nothing.

---

## 7. Aggregations {#aggregations}

```json
"aggs": {
  "by_category": { "terms": { "field": "category", "size": 20 },
    "aggs": { "avg_price": { "avg": { "field": "price" } } } },
  "price_ranges": { "range": { "field": "price", "ranges": [ {"to": 1000}, {"from": 1000, "to": 5000}, {"from": 5000} ] } },
  "per_day": { "date_histogram": { "field": "created_at", "calendar_interval": "day" } },
  "unique_users": { "cardinality": { "field": "user_id" } },
  "p99_latency": { "percentiles": { "field": "latency_ms", "percents": [50, 95, 99] } }
}
```
- Bucket (terms, histogram, date_histogram, range, filters, composite (paginate all buckets)), metric (avg, sum, min/max, stats, cardinality (HyperLogLog++, approximate), percentiles (TDigest, approximate)), pipeline (derivative, moving_fn, bucket_sort).
- They power **faceted navigation** (counts per filter value) and dashboards (Kibana/OpenSearch Dashboards).
- `terms` aggs on high-cardinality fields across many shards are approximate (`shard_size`, `doc_count_error_upper_bound`).
- They run on **doc values** (columnar). `text` fields have none (fielddata is disabled by default because it's memory hungry).

---

## 8. Write Path: Near-Real-Time {#writepath}

1. A document arrives at the coordinating node, gets routed to the primary shard (`hash(_routing or _id) % num_primary_shards`).
2. The primary writes to an in-memory buffer and appends to the **translog** (WAL; fsync per request by default: `index.translog.durability: request`).
3. The primary replicates to the replica shards, then acks.
4. **Refresh** (default every **1 s**, only for indices searched recently): the buffer becomes a new searchable **segment** (in the filesystem cache, not yet fsynced). This is why ES is **near-real-time**: a document isn't searchable until a refresh. Use `?refresh=wait_for` if a request must see its own write (expensive at scale).
5. **Flush**: Lucene commit (fsync segments) and translog trimming.
6. **Merges**: background merging of small segments into bigger ones. Deleted/updated docs are only physically removed on merge (updates = delete + reindex, since segments are immutable).

Bulk indexing tuning: the `_bulk` API (5–15 MB batches), `refresh_interval: -1` or `30s` during big loads, `number_of_replicas: 0` during the initial load and restored afterward, auto-generated IDs (skip the existence check), and parallel clients.

`GET` by ID is real-time (it reads from the translog or version map); search is not.

Optimistic concurrency: `if_seq_no` + `if_primary_term` on writes. External versioning (`version_type=external`) lets the source DB's version win, which is useful for CDC replays out of order.

---

## 9. Cluster Architecture and Shard Sizing {#cluster}

- **Master-eligible nodes** (3 dedicated for prod): manage cluster state (mappings, shard allocation). Quorum-based election (7.x+, no more `minimum_master_nodes` split-brain config).
- **Data nodes**: hold shards. Tiers: **hot** (SSD, recent writes) → **warm** → **cold** (searchable snapshots) → **frozen** (mostly on object storage). **ILM** (Index Lifecycle Management) / ISM in OpenSearch moves indices through tiers: rollover at 50 GB or 1 day, shrink, force-merge, delete after 30 days.
- **Coordinating-only** nodes for heavy aggregations and fan-out; **ingest nodes** run ingest pipelines (grok, enrich, geoip).
- **Shard sizing**: aim for **10–50 GB per shard** (Elastic guidance); avoid thousands of tiny shards (each shard costs heap and cluster state, the "oversharding" problem). Keep shards per node roughly under 20 per GB of heap as an old rule of thumb. Newer versions are more efficient, but the principle holds.
- Heap: ≤ 50% of RAM and below ~31 GB (compressed OOPs). The rest of RAM is for the **OS filesystem cache**, which Lucene relies on heavily.
- Search fan-out: a query goes to one copy of every shard (query phase: top-N IDs and scores per shard), then the coordinator merges and fetches the documents (fetch phase). More shards means more fan-out overhead.
- **Routing** (`_routing=tenant_id`) sends a tenant's docs to one shard, so its queries hit one shard. Watch for skew.
- Cluster health: green (all shards allocated), yellow (replicas unallocated), red (a primary unallocated, so data is unavailable).

---

## 10. Keeping Search in Sync {#sync}

The system of record (Postgres/MySQL/Mongo) must propagate changes to the search index:
| Approach | Pros | Cons |
|---|---|---|
| **Dual write** in app (DB then ES) | Simple | Inconsistent on partial failure; ordering races |
| **Transactional outbox** → worker → ES | Reliable, ordered per entity | Extra table/worker |
| **CDC** (Debezium → Kafka → ES sink connector / custom consumer) | Decoupled, captures all writes incl. manual SQL | Infra complexity; need to denormalize joined data |
| Periodic batch reindex (by `updated_at`) | Simple | Latency; misses deletes unless soft-deleted |
| Logstash JDBC input | Easy | Polling, same deletion problem |

Use **external versioning** (DB row version or LSN) so out-of-order events don't overwrite newer data. Denormalize at index time (a product doc includes category name and brand name), and when a category is renamed, reindex the affected products (by query or by a fan-out job).

Full reindex strategy: create `products_v2` with the new mapping, bulk load it from the DB (or `_reindex` from v1), replay changes that happened during the load (from Kafka offsets or timestamps), then atomically swap the alias `products` → v2.

---

## 11. Pagination, Autocomplete, Typos, Synonyms {#features}

- **Pagination**: `from + size` is limited to `index.max_result_window` (10,000) because deep pages are expensive (each shard must return `from+size` hits). Use **`search_after`** (keyset with a sort tiebreaker) + **Point in Time (PIT)** for consistent deep pagination. Scroll API is for bulk export (legacy for user pagination).
- **Autocomplete**: `edge_ngram` analyzers, `search_as_you_type` field, `completion` suggester (an FST in memory, very fast prefix), or `match_bool_prefix`.
- **Typo tolerance**: `fuzziness: AUTO` (Levenshtein edit distance 0/1/2 by term length), phonetic analyzers, "did you mean" via the `phrase` suggester.
- **Synonyms**: a `synonym_graph` filter at **search time** (so updates don't require reindexing, and the synonyms API in 8.10+ allows reloading). Multi-word synonyms need the graph filter.
- **Highlighting**: unified highlighter.
- **Multilingual**: per-language fields/analyzers, ICU plugin, language detection in an ingest pipeline.

---

## 12. Logs and Observability (ELK) {#logs}

ELK/Elastic Stack: **Beats/Elastic Agent/Fluent Bit** (ship) → **Logstash** or ingest pipelines (parse/enrich) → **Elasticsearch** (store/search) → **Kibana** (visualize). In OpenSearch the equivalent is OpenSearch + Dashboards + Data Prepper.
- Data streams + ILM rollover; hot-warm-cold architecture.
- Store structured JSON logs (don't parse free text with grok if you can avoid it).
- Cost pressure has pushed many teams toward **Loki** (indexes only labels, stores chunks in object storage), **ClickHouse**-based logging (SigNoz, Uptrace, Quickwit), or vendor SaaS.

---

## 13. Vector, Semantic, and Hybrid Search {#vector}

- `dense_vector` with HNSW (`knn` search), quantization (int8, int4, BBQ binary quantization) to cut memory.
- Generate embeddings with an inference pipeline (ELSER sparse model, external embedding models) at ingest and query time.
- **Hybrid search**: combine BM25 lexical results with kNN semantic results using **Reciprocal Rank Fusion (RRF)** (the `retriever` API in 8.14+) or weighted score combination. It usually beats either alone.
- Semantic reranking with cross-encoders on the top-k.
- OpenSearch: the k-NN plugin (Faiss/nmslib/Lucene engines) plus a neural search plugin.

---

## 14. Operations and Pitfalls {#ops}

- **Mapping explosion** from dynamic fields → strict mappings / flattened type.
- **Oversharding** → fewer, larger shards; rollover by size.
- **Heap pressure / circuit breakers** (`CircuitBreakingException`): big aggregations, fielddata on text fields, too many shards.
- **Deep pagination** → search_after.
- **Using ES as the primary database**: no transactions, eventual visibility, historical data-loss bugs under partitions (Jepsen tests in the 1.x era). Keep the source of truth elsewhere, and snapshot regularly (`_snapshot` to S3).
- **Expensive queries**: leading wildcards, regex, scripts in queries, huge `terms` lists. `search.allow_expensive_queries` can disable them.
- Security: enable TLS and auth (free in the basic license since 6.8/7.1). Never expose port 9200 to the internet (massive data leaks from open ES clusters are a recurring headline).
- Upgrades: rolling upgrades within major versions; snapshot before upgrades.
- Monitoring: cluster health, JVM heap/GC, search/index latency and rate, thread pool rejections (`write`, `search` queues), disk watermarks (85% low / 90% high / 95% flood stage → indices made read-only!), segment counts, merges.

---

## 15. Interview Questions {#qa}

1. How does an inverted index work? Why is search fast?
2. Explain BM25's components.
3. `text` vs `keyword`: when do you use each, and why does a `term` query on a `text` field fail?
4. Why is Elasticsearch "near real-time"? What do refresh and flush do?
5. How do you keep Elasticsearch in sync with Postgres reliably?
6. How do you change a field's mapping in production without downtime?
7. How would you size shards for 2 TB/day of logs with 30 days retention?
8. How do you implement deep pagination?
9. What's hybrid search and how is RRF used?
10. Why shouldn't Elasticsearch be your primary database?
