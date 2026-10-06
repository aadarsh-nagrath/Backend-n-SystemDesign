# Backend and System Design

A deep, structured knowledge base for becoming a strong backend engineer: from TCP packets and B+trees to consensus algorithms, event-driven architectures, and production operations. The goal is **depth**: every topic is covered down to the mechanisms, the trade-offs, the failure modes, and the questions interviewers and production incidents will ask.

> 📖 Browse the notes in the built-in viewer (`notes-viewer/`, deployed to GitHub Pages), or read the Markdown directly on GitHub.

---

## 🗺️ Map of the Repository

| Area | Folder | What's inside |
|---|---|---|
| **Databases** | [`databases/`](databases/README.md) | Relational theory, SQL, transactions & isolation, storage engines, query optimization, schema & migrations, pooling, replication/backup. Then deep dives into **PostgreSQL** (11 notes) and **MySQL/InnoDB** (9 notes), **NoSQL** (MongoDB, Cassandra, DynamoDB, Elasticsearch), and **analytics** (OLAP, CDC, ClickHouse) |
| **Database concepts (original notes)** | [`database-concepts/`](database-concepts/), [`scaling-db/`](scaling-db/) | ACID, ORMs, indexing, sharding, CAP |
| **Networking** | [`networking/`](networking/README.md) | TCP/IP & sockets, DNS, HTTP/1.1→2→3 & QUIC, load balancers/proxies/gateways/service mesh, real-time (SSE, WebSockets, WebRTC) |
| **APIs** | [`api/`](api/) | REST, [GraphQL](api/graphql.md), gRPC, JSON:API, [API design best practices](api/api-design-best-practices.md) (errors, idempotency, pagination, versioning, webhooks, OpenAPI) |
| **Messaging & async** | [`messaging/`](messaging/README.md) | Queues vs logs, delivery semantics, **Kafka**, RabbitMQ/SQS/SNS/NATS, outbox/sagas/CQRS/event sourcing, background jobs & workflows (Temporal) |
| **Distributed systems** | [`distributed-systems/`](distributed-systems/README.md) | Failures & clocks, Raft/Paxos & coordination, consistency models & CRDTs, core algorithms (consistent hashing, Bloom, HLL, Snowflake IDs, rate limiting, geo), resilience patterns |
| **Caching** | [`caching/`](caching/) | Client-side, CDN, server-side, Redis |
| **Architecture** | [`architecture/`](architecture/README.md) | Monolith vs modular monolith vs microservices, hexagonal/clean/DDD, twelve-factor, config & secrets, containers, Kubernetes basics, CI/CD & deployment strategies |
| **Concurrency** | [`concurrency/`](concurrency/concurrency-and-async-io.md) | Threads vs async vs virtual threads, epoll/io_uring, event loops, runtimes (Go, JVM, Python, Node, Rust, BEAM), races/deadlocks, memory models |
| **Storage & files** | [`storage/`](storage/object-storage-and-file-handling.md) | Object storage (S3), presigned & multipart uploads, secure file serving, processing pipelines, video streaming basics |
| **Observability** | [`observability/`](observability/observability-logs-metrics-traces-slos.md) | Structured logs, metrics & cardinality, tracing & OpenTelemetry, SLOs/error budgets, burn-rate alerting, incident response |
| **Testing** | [`testing/`](testing/backend-testing-strategy.md) | Test strategy, doubles, Testcontainers, contract testing (Pact), property-based & mutation testing, load testing, flaky tests |
| **Authentication** | [`authentication/`](authentication/) | Auth fundamentals, JWT, OAuth 2.0/OIDC, advanced auth, security best practices, testing & monitoring |
| **API & web security** | [`api-security/`](api-security/), [`web-security/`](web-security/) | API security, CORS, CSP/OWASP, HTTPS, SSL/TLS, hashing (MD5, SHA, bcrypt/scrypt) |
| **Web servers** | [`web-servers/`](web-servers/) | Nginx, Apache, comparison |
| **System design** | [`system-design/`](system-design/), [`system-desing-websocket-focused.md`](system-desing-websocket-focused.md), [`os-sysdesign-ipc.md`](os-sysdesign-ipc.md) | 40 core concepts with examples, curated resources, implementations (consistent hashing, LB algorithms, rate limiters in Java & Python), high-level architectures, OS/IPC & design patterns |
| **Version control** | [`git/`](git/) | Git & GitHub |
| **Roadmap** | [`Roadmap/`](Roadmap/) | Backend roadmap PDF, internet fundamentals, sources |
| **Interview prep** | [`interview-prep/`](interview-prep/README.md) | ~800 answered Q&As (DevOps, backend, SDE, system design) + 18 case-study walkthroughs |

---

## 🎯 Learning Path

The order goes from foundations to production mastery. Each step links to the notes to read.

### Stage 1: Foundations (weeks 1–4)
1. **How the internet works**: [`networking/01`](networking/01-tcp-ip-udp-and-sockets.md) TCP/IP → [`02`](networking/02-dns.md) DNS → [`03`](networking/03-http-evolution-1-1-2-3-and-quic.md) HTTP → [`api-security/ssl-tls.md`](api-security/ssl-tls.md)
2. **Git**: [`git/`](git/)
3. **APIs**: [`api/restapi.md`](api/restapi.md) → [`api/api-design-best-practices.md`](api/api-design-best-practices.md)
4. **Relational databases & SQL**: [`databases/fundamentals/01`](databases/fundamentals/01-relational-model-and-normalization.md) → [`02`](databases/fundamentals/02-sql-deep-dive.md) → [`03`](databases/fundamentals/03-transactions-isolation-concurrency.md)
5. **Authentication**: [`authentication/1-authentication.md`](authentication/1-authentication.md) → JWT → OAuth
6. **Web security basics**: [`web-security/intro.md`](web-security/intro.md), hashing, [`api-security/cors.md`](api-security/cors.md)

### Stage 2: Building real services (weeks 5–10)
1. **Database internals & performance**: [`databases/fundamentals/04`](databases/fundamentals/04-storage-engine-internals.md) → [`05`](databases/fundamentals/05-query-planning-and-optimization.md) → [`postgresql/`](databases/postgresql/README.md)
2. **Schema design, migrations, pooling**: [`databases/fundamentals/06`](databases/fundamentals/06-schema-design-and-migrations.md), [`07`](databases/fundamentals/07-connections-pooling-and-app-integration.md)
3. **Caching**: [`caching/`](caching/) (client-side, CDN, server-side, Redis)
4. **Concurrency & async I/O**: [`concurrency/`](concurrency/concurrency-and-async-io.md)
5. **Code architecture**: [`architecture/02`](architecture/02-clean-hexagonal-ddd.md)
6. **Testing**: [`testing/`](testing/backend-testing-strategy.md)
7. **Files & object storage**: [`storage/`](storage/object-storage-and-file-handling.md)
8. **More API styles**: [`api/graphql.md`](api/graphql.md), [`api/grpc.md`](api/grpc.md)

### Stage 3: Scale & distribution (weeks 11–16)
1. **Messaging & events**: [`messaging/`](messaging/README.md) (fundamentals → Kafka → patterns → jobs)
2. **Distributed systems theory**: [`distributed-systems/`](distributed-systems/README.md)
3. **Replication, sharding, CAP**: [`databases/fundamentals/08`](databases/fundamentals/08-replication-ha-backup-recovery.md), [`scaling-db/`](scaling-db/)
4. **MySQL & NoSQL**: [`databases/mysql/`](databases/mysql/README.md), [`databases/nosql/`](databases/nosql/README.md)
5. **Load balancing, gateways, real-time**: [`networking/04`](networking/04-load-balancers-proxies-and-gateways.md), [`05`](networking/05-real-time-communication.md)
6. **Microservices vs monolith**: [`architecture/01`](architecture/01-monolith-modular-monolith-microservices.md)

### Stage 4: Production engineering (weeks 17–20)
1. **Resilience**: [`distributed-systems/05`](distributed-systems/05-resilience-patterns.md)
2. **Observability & SLOs**: [`observability/`](observability/observability-logs-metrics-traces-slos.md)
3. **Deployment & operations**: [`architecture/03`](architecture/03-twelve-factor-config-and-deployment.md), [`web-servers/`](web-servers/)
4. **Analytics & data pipelines**: [`databases/analytics/`](databases/analytics/olap-warehousing-and-data-pipelines.md)
5. **Security hardening**: [`api-security/`](api-security/), [`authentication/Authentication Security Best Practices.md`](authentication/Authentication%20Security%20Best%20Practices.md)

### Stage 5: System design mastery (ongoing)
1. [`system-design/notes/concepts.md`](system-design/notes/concepts.md): the 40 core concepts
2. [`system-design/awesome-notes/`](system-design/awesome-notes/README.md): curated resources and implementations
3. [`interview-prep/system-design/`](interview-prep/system-design/): case studies (URL shortener, chat, feed, notifications, ride-hailing, …)
4. Practice: design one system per week, write it down, and compare it with reference designs

---

## ✅ Coverage Tracker

### Covered in depth
- [x] Networking: TCP/IP, UDP, sockets, DNS, HTTP/1.1/2/3, QUIC, TLS, load balancing, proxies, API gateways, service mesh, real-time protocols
- [x] APIs: REST, GraphQL, gRPC, JSON:API, API design (errors, idempotency, pagination, versioning, webhooks, OpenAPI)
- [x] Authentication & authorization: sessions, JWT, OAuth2/OIDC, MFA, security practices
- [x] Web/API security: OWASP, CORS, CSP, HTTPS/TLS, password hashing
- [x] Databases: relational theory, SQL, transactions/isolation/MVCC, storage engines, indexing, query optimization, schema design, zero-downtime migrations, pooling, replication, backups/PITR, choosing a DB
- [x] PostgreSQL: architecture, MVCC/VACUUM, indexes, EXPLAIN, JSONB/FTS, locking, partitioning, replication/HA, tuning, security/RLS, extensions/ops
- [x] MySQL: architecture, InnoDB internals, indexes/EXPLAIN, locking, replication/HA/Vitess, tuning, backup/online DDL, gotchas, vs PostgreSQL
- [x] NoSQL: MongoDB, Cassandra/ScyllaDB, DynamoDB, Elasticsearch/OpenSearch, Redis
- [x] Analytics: OLAP, warehousing, dimensional modeling, ETL/ELT, CDC, streaming, lakehouse, ClickHouse
- [x] Caching: client, CDN, server-side, Redis
- [x] Messaging: queues/logs, delivery semantics, Kafka, RabbitMQ, SQS/SNS/EventBridge, NATS, outbox, sagas, CQRS, event sourcing, jobs & workflows
- [x] Distributed systems: failures, clocks, consensus (Raft/Paxos), coordination & locks, consistency models, CRDTs, probabilistic structures, ID generation, rate limiting, geo-indexing, resilience patterns
- [x] Architecture: monolith/microservices, DDD, hexagonal/clean, twelve-factor, config/secrets, feature flags, containers, Kubernetes basics, CI/CD, deployment strategies
- [x] Concurrency: threads/async/virtual threads, I/O models, event loops, language runtimes, synchronization, memory models
- [x] Storage: object storage, uploads, secure file serving, media pipelines, video streaming
- [x] Observability: logs, metrics, traces, OpenTelemetry, SLOs, alerting, incident response
- [x] Testing: unit/integration/contract/E2E, Testcontainers, property-based, mutation, load testing, flaky tests
- [x] Interview prep: DevOps, backend, SDE, system design (~800 Q&As)

### Next topics to add (gaps still open)
- [ ] **Language/framework deep dives** (one per stack you use): Node.js (Express/Fastify/NestJS), Python (FastAPI/Django), Java (Spring Boot), Go (net/http, Gin/Echo). The interview prep has Q&A for Node, Python, and Java
- [ ] **Search engineering beyond Elasticsearch basics**: relevance tuning, learning-to-rank, typeahead systems
- [ ] **Security deep dives**: secrets & KMS internals, supply-chain security (SBOM, SLSA), threat modeling (STRIDE), OWASP ASVS checklist, multi-tenant isolation reviews
- [ ] **Cloud architecture**: AWS core services for backend engineers (VPC, IAM, ALB/NLB, RDS/Aurora, DynamoDB, SQS/SNS, Lambda, ECS/EKS), well-architected framework, cost optimization (FinOps)
- [ ] **Performance engineering**: profiling per runtime (pprof, async-profiler, py-spy, clinic.js), flame graphs walkthroughs, GC tuning (JVM G1/ZGC, Go GC)
- [ ] **Data privacy & compliance for engineers**: GDPR/DPDP/HIPAA/PCI-DSS implications for data modeling, retention, encryption, audit
- [ ] **AI/LLM backend engineering**: RAG architecture, vector search in production, LLM API integration (streaming, retries, cost control, caching, evals), agent/tool backends
- [ ] **Email & notifications infrastructure**: deliverability (SPF/DKIM/DMARC), providers, templating, multi-channel notification system design
- [ ] **Payments engineering**: ledgers, idempotency, reconciliation, PSP integration, refunds/chargebacks
- [ ] **Multi-region architecture**: active-active patterns, data residency, global traffic management
- [ ] **Hands-on projects**: build-your-own exercises (key-value store with WAL, Raft from the MIT labs, rate limiter service, URL shortener with full observability)
- [ ] Fill [`system-design/mynotes/readme.md`](system-design/mynotes/readme.md) with personal case-study write-ups

---

## 🛠️ Tools & Technologies Map

| Category | Tools |
|---|---|
| Languages/runtimes | Go, Java/Kotlin (JVM), Python, Node.js/TypeScript, Rust, C#/.NET, Elixir |
| Databases | PostgreSQL, MySQL, Redis/Valkey, MongoDB, Cassandra/ScyllaDB, DynamoDB, Elasticsearch/OpenSearch, ClickHouse |
| Messaging | Kafka, RabbitMQ, SQS/SNS, NATS, Redis Streams, Temporal |
| Infra | Docker, Kubernetes, Terraform/OpenTofu, Helm, Argo CD |
| Proxies/LB | Nginx, Envoy, HAProxy, Traefik, cloud LBs, Istio/Linkerd |
| Observability | OpenTelemetry, Prometheus, Grafana, Loki, Tempo/Jaeger, Sentry |
| Testing | Testcontainers, Pact, k6, Hypothesis/fast-check/jqwik, WireMock |
| API tooling | OpenAPI, Postman/Bruno, buf (protobuf), GraphQL Codegen |

---

## 📖 Essential Books & Resources

**Books**
- *Designing Data-Intensive Applications*, Martin Kleppmann (the single most important book for this repo)
- *Database Internals*, Alex Petrov
- *System Design Interview* Vols 1 & 2, Alex Xu
- *Understanding Distributed Systems*, Roberto Vitillo
- *Release It!*, Michael Nygard
- *Site Reliability Engineering* & *The Site Reliability Workbook* (Google, free online)
- *Building Microservices* & *Monolith to Microservices*, Sam Newman
- *Fundamentals of Software Architecture*, Richards & Ford
- *High Performance MySQL*; *The Art of PostgreSQL*
- *Clean Code*, *The Pragmatic Programmer*

**Courses & sites**
- [Backend Developer Roadmap](https://roadmap.sh/backend)
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- MIT 6.5840 Distributed Systems (Raft labs), CMU 15-445 Database Systems (Andy Pavlo)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/), [OWASP API Security Top 10](https://owasp.org/API-Security/)
- Engineering blogs: Cloudflare, Netflix, Uber, Discord, Stripe, Shopify, GitHub, AWS Builders' Library
- [jepsen.io](https://jepsen.io) analyses, Marc Brooker's blog, Martin Kleppmann's lectures

---

## 🖥️ Notes Viewer

`notes-viewer/` is a small Express + static site that renders every Markdown note in a searchable tree.
```bash
cd notes-viewer
npm install
npm start            # local server at http://localhost:3000 (reads notes live)
npm run build        # builds the static site into notes-viewer/public (deployed via GitHub Actions)
```

## 🤝 Contributing

This repository is for learning and sharing knowledge. You're welcome to:
- Add new topics (see "Next topics to add")
- Improve existing content, fix errors, add diagrams or clarifications
- Suggest resources

Note conventions: one topic per file, a table of contents at the top, **why → how → trade-offs → pitfalls → interview questions**, and cross-links to related notes.
