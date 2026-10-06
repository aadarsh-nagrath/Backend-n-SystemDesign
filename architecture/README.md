# Software Architecture for Backend Systems

These notes cover how to structure systems at three scales: the codebase (layers, hexagonal, DDD), the system (monolith vs microservices), and the delivery pipeline (config, containers, deployments).

| # | Note | Topics |
|---|---|---|
| 01 | [Monolith, Modular Monolith & Microservices](01-monolith-modular-monolith-microservices.md) | Trade-offs, decision framework, service boundaries, database per service, sync vs async communication, distributed monolith anti-patterns, strangler fig migration, serverless, Conway's Law & team topologies |
| 02 | [Clean/Hexagonal Architecture & DDD](02-clean-hexagonal-ddd.md) | Layered vs hexagonal vs clean vs vertical slices, bounded contexts & context maps, entities/value objects/aggregates/repositories/domain events, aggregate rules, validation & error strategy, DI, SOLID, reference project structure |
| 03 | [Twelve-Factor, Config, Secrets & Deployment](03-twelve-factor-config-and-deployment.md) | 12-factor, config & secrets management, feature flags, production Dockerfiles, Kubernetes essentials, graceful startup/shutdown, CI/CD, deployment strategies (rolling/blue-green/canary/shadow), migrations in pipelines, GitOps, production readiness checklist |

Related:
- Design patterns (GoF): [`os-sysdesign-ipc.md`](../os-sysdesign-ipc.md) §2, [`interview-prep/sde-general/02-oop-design-patterns.md`](../interview-prep/sde-general/02-oop-design-patterns.md)
- Event-driven architecture: [`messaging/04-event-driven-architecture-patterns.md`](../messaging/04-event-driven-architecture-patterns.md)
- Resilience: [`distributed-systems/05-resilience-patterns.md`](../distributed-systems/05-resilience-patterns.md)
- Gateways & service mesh: [`networking/04-load-balancers-proxies-and-gateways.md`](../networking/04-load-balancers-proxies-and-gateways.md)
- High-level case studies: [`system-desing-websocket-focused.md`](../system-desing-websocket-focused.md), [`interview-prep/system-design/`](../interview-prep/system-design/)

## Recommended reading
- *Fundamentals of Software Architecture* (Richards & Ford), *Software Architecture: The Hard Parts*
- *Building Microservices*, 2nd ed., and *Monolith to Microservices* (Sam Newman)
- *Domain-Driven Design* (Evans), *Implementing DDD* / *DDD Distilled* (Vernon), *Learning DDD* (Khononov)
- *Clean Architecture* (Robert C. Martin), *Architecture Patterns with Python* ("Cosmic Python", free online)
- *Team Topologies* (Skelton & Pais)
- *Release It!*, 2nd ed. (Nygard): stability patterns
- martinfowler.com articles: MonolithFirst, StranglerFigApplication, BoundedContext, Microservices
- Architecture Decision Records (adr.github.io): write one for every significant choice
