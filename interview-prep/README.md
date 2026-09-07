# Interview Prep: DevOps + Backend + SDE + System Design

A comprehensive, self-contained Q&A study set for backend/DevOps engineering interviews — junior through senior — plus general SDE and system design rounds. Every question has a full written answer (not just a one-liner), researched against current industry-standard practice and cross-referenced across files where topics overlap.

**~800+ individual Q&A entries + 18 full system design case-study walkthroughs**, organized into 42 topic files across 4 sections. A second research pass (see "Round 2" below) added tool-specific and language-specific content sourced directly from the most popular interview-question blogs and aggregators (GeeksforGeeks, InterviewBit, KodeKloud, igmGuru, DesignGurus, DataCamp, Tecmint), cross-checked against the existing conceptual coverage to avoid duplication.

## How to use this

Don't try to memorize answers verbatim — read a file, then close it and explain the concept out loud in your own words. The scenario/troubleshooting and case-study files are deliberately written as structured *reasoning* to model (how to approach the question), not scripts to recite. Cross-references between files (e.g., "see the backend fundamentals file's idempotency question") are intentional — many concepts recur across DevOps/backend/system-design because they're the same underlying idea applied at different layers.

---

## 1. DevOps (`devops/`) — ~440 questions

| File | Topic |
|---|---|
| `01-linux-fundamentals.md` | Processes, permissions, filesystem, signals, containers-at-the-kernel-level, troubleshooting CPU/memory/disk |
| `02-networking-dns-http.md` | OSI model, DNS records, HTTP/1.1 vs 2 vs 3, load balancing, TLS, WebSockets, rate limiting |
| `03-git-version-control.md` | merge/rebase, reflog, bisect, history rewriting, branching strategies, PR workflows |
| `04-docker-containers.md` | Dockerfile, layers, multi-stage builds, networking, volumes, image security |
| `05-kubernetes.md` | Architecture, Pods/Deployments/Services/Ingress, probes, RBAC, autoscaling, GitOps |
| `06-ci-cd-pipelines.md` | CI/CD stages, deployment strategies (canary/blue-green/rolling), feature flags, secrets in pipelines |
| `07-aws-cloud.md` | EC2/ECS/EKS/Lambda, VPC, IAM, S3, RDS/DynamoDB, autoscaling, HA architecture |
| `08-azure-and-gcp.md` | Cross-cloud service mapping, GKE Autopilot, Cloud Run, Entra ID, multi-cloud tradeoffs |
| `09-terraform-ansible-iac.md` | State management, modules, drift, Ansible idempotency, Packer, immutable infra |
| `10-monitoring-logging-observability.md` | Metrics/logs/traces, SLI/SLO/error budgets, Prometheus, structured logging, OpenTelemetry |
| `11-security-devsecops.md` | OWASP Top 10, SQLi/XSS/CSRF, JWT pitfalls, secrets management, supply chain security |
| `12-sre-incident-management.md` | Incident response, blameless postmortems, on-call, circuit breakers, DR (RTO/RPO) |
| `13-scripting-automation.md` | Bash safety (`set -euo pipefail`), Python for automation, idempotent scripts |
| `14-scenario-troubleshooting.md` | 15 structured "walk me through debugging X" scenarios (senior/on-call interview staple) |
| `15-jenkins-cicd-tools.md` ✨ | Jenkinsfile, declarative vs scripted, agents, shared libraries, JCasC, real troubleshooting scenarios |
| `16-more-kubernetes-practical.md` ✨ | Static Pods, PV/PVC/StorageClass, CSI, ResourceQuota, CoreDNS, cordon/drain/uncordon |
| `17-more-docker-practical-scenarios.md` ✨ | save/export, exit code 137, build cache internals, HEALTHCHECK, real debugging scenarios |
| `18-aws-devops-tools-real-questions.md` ✨ | CodeBuild/CodeDeploy/CodePipeline/CodeStar, Elastic Beanstalk, Systems Manager, X-Ray vs ADOT |
| `19-more-linux-and-git-practical.md` ✨ | fsck, page faults, `at`/`atq`, plus Git: config levels, commit object anatomy, branch recovery |

## 2. Backend Engineering (`backend-engineer/`) — ~225 questions

| File | Topic |
|---|---|
| `01-backend-fundamentals.md` | Statelessness, blocking vs async I/O, race conditions, CAP theorem, monolith vs microservices |
| `02-api-design-rest-graphql-grpc.md` | REST constraints, idempotency, pagination, GraphQL N+1, gRPC/protobuf, API versioning |
| `03-databases-sql-nosql.md` | ACID, isolation levels, indexing (B-tree, covering, composite), sharding, replication |
| `04-caching-strategies.md` | Cache-aside/write-through, invalidation, stampede/penetration, Redis data structures |
| `05-concurrency-multithreading-async.md` | Mutex/semaphore, deadlock, async/await internals, Promise.all vs allSettled |
| `06-message-queues-event-driven.md` | Delivery semantics, Kafka vs traditional queues, event sourcing, CQRS, Saga pattern |
| `07-microservices-architecture.md` | Service boundaries, API Gateway, service mesh, Bulkhead pattern, strangler fig |
| `08-security-authentication.md` | Session vs JWT, OAuth2/OIDC flows, MFA, token storage, timing attacks, CORS |
| `09-testing-quality.md` | Unit/integration/e2e, mocks vs fakes, flaky tests, contract testing, mutation testing |
| `10-sql-query-patterns-and-questions.md` ✨ | Real SQL: window functions, CTEs, recursive queries, running totals, dedup, sargability |
| `11-nodejs-specific-questions.md` ✨ | Event loop phases, libuv, EventEmitter, streams/Buffer, cluster module, Express middleware |
| `12-python-specific-questions.md` ✨ | GIL, decorators, generators, mutable-default-argument gotcha, pickling security, duck typing |
| `13-java-spring-boot-questions.md` ✨ | JVM/GC, IoC/DI, @SpringBootApplication internals, Spring Data JPA, Actuator, Spring Security |

## 3. System Design (`system-design/`) — ~40 questions + 18 case studies

| File | Topic |
|---|---|
| `01-fundamentals-and-building-blocks.md` | Interview approach, estimation, consistent hashing, fan-out, hot keys |
| `02-scalability-availability-reliability.md` | Availability math, replication topologies, multi-region, graceful degradation |
| `03-data-storage-design.md` | Sharding strategy, CDC, time-series storage, global uniqueness at scale |
| `04-case-studies-walkthroughs.md` | **12 full designs**: URL shortener, rate limiter, chat app, news feed, notifications, distributed cache, web crawler, video streaming, ride-hailing, unique ID generator, typeahead/autocomplete, e-commerce orders |
| `05-more-popular-case-studies.md` ✨ | **6 more full designs**: Google Docs (OT/CRDTs), Dropbox (chunking/dedup), Ticketmaster (seat locking), Stripe-like payments, Airbnb (marketplace), Zoom (SFU vs MCU) |

## 4. SDE General (`sde-general/`) — ~100 questions

| File | Topic |
|---|---|
| `01-dsa-coding-concepts-qna.md` | Complexity analysis, data structure tradeoffs, DP, graph algorithms, interview approach |
| `02-oop-design-patterns.md` | SOLID, composition vs inheritance, Factory/Strategy/Observer/Decorator/Repository patterns |
| `03-behavioral-interview-questions.md` | STAR-structured questions on incidents, ownership, collaboration, on-call maturity |
| `04-object-oriented-design-problems.md` ✨ | LLD/OOD: Parking Lot, Elevator System, Library Management, Vending Machine (State pattern) |

✨ = added in the second research pass, sourced/topic-checked directly against popular interview-question blogs (see Sources below).

---

## Suggested study order

1. **Foundations first**: `devops/01-linux-fundamentals.md` → `02-networking-dns-http.md` → `backend-engineer/01-backend-fundamentals.md`. Everything else builds on these.
2. **Core DevOps stack**: Docker → Kubernetes → CI/CD → your target cloud (AWS or Azure/GCP) → Terraform/Ansible.
3. **Reliability layer**: Monitoring → Security → SRE/incident management → scenario-troubleshooting (this last one is the best pre-interview review for senior/on-call-heavy roles).
4. **Backend depth**: databases → caching → concurrency → message queues → microservices → API design → security/auth → testing.
5. **System design**: fundamentals → scalability → data storage → then work through the 12 case studies one at a time, closing the file and redrawing the architecture from memory before checking yourself.
6. **SDE round prep**: DSA concepts (pair with actual LeetCode practice, this file is conceptual not a problem set) → OOP/patterns → LLD/OOD problems (`04-object-oriented-design-problems.md`) → behavioral (prepare your own real stories against this list, don't memorize the examples).
7. **Language-specific tracks** (pick based on your target stack): SQL query patterns → Node.js, Python, or Java/Spring Boot specifics, in `backend-engineer/10-13`.
8. **Tool-specific DevOps depth**: Jenkins, the "more Kubernetes/Docker/AWS/Linux/Git" files (`devops/15-19`) are best reviewed right before an interview that named a specific tool in the job description — they're denser and more command/scenario-focused than the foundational files.

## Sources consulted (Round 2 research pass)

Question breadth and real-world phrasing for the ✨-marked files were cross-checked against: [GeeksforGeeks](https://www.geeksforgeeks.org/) (Kubernetes, Docker, SQL, Git, Node.js, Jenkins, Python interview-question lists), [KodeKloud's blog](https://kodekloud.com/blog/) (Kubernetes and Docker interview questions, including scenario-based ones), [InterviewBit](https://www.interviewbit.com/) (Git, Python, Spring Boot interview questions), [igmGuru](https://www.igmguru.com/blog/) (AWS DevOps interview questions), [Tecmint](https://www.tecmint.com/) (Linux sysadmin interview questions), and [DesignGurus](https://www.designgurus.io/blog/) (the current list of most-asked FAANG system design questions). Every answer was then authored/verified directly from engineering first principles — questions were sourced from these sites for topic coverage and real interview phrasing, but no answer text was copied from them.

*Last generated: September 2026. Content researched against current (2026) industry sources and written out in full — where public sources didn't provide a complete answer, the answer was authored directly from first-principles engineering knowledge.*
