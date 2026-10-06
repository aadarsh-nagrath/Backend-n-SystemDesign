# Twelve-Factor Apps, Configuration, Secrets, Containers, CI/CD, and Deployment Strategies

> Deep DevOps Q&A lives in [`interview-prep/devops/`](../interview-prep/devops/) (Docker, Kubernetes, CI/CD, Terraform). This note covers what a **backend engineer** must know to build services that are easy to deploy, configure, scale, and operate.

## Table of Contents
1. [The Twelve-Factor App (and Beyond)](#twelve)
2. [Configuration Management](#config)
3. [Secrets Management](#secrets)
4. [Feature Flags](#flags)
5. [Containerizing a Backend Service Properly](#containers)
6. [Kubernetes Essentials for Backend Engineers](#k8s)
7. [Graceful Startup and Shutdown](#graceful)
8. [CI/CD Pipeline for a Service](#cicd)
9. [Deployment Strategies](#deploy)
10. [Database Migrations in the Pipeline](#migrations)
11. [Environments, Preview Environments, and Parity](#envs)
12. [Infrastructure as Code and GitOps](#iac)
13. [Production Readiness Checklist](#readiness)
14. [Interview Questions](#qa)

---

## 1. The Twelve-Factor App {#twelve}

Heroku's methodology (2011, Adam Wiggins). It's still the baseline for cloud-native services:
| # | Factor | Meaning | Common violation |
|---|---|---|---|
| I | **Codebase** | One codebase in VCS, many deploys | Different code per environment |
| II | **Dependencies** | Explicitly declare and isolate (lockfiles, containers) | Relying on system-installed tools |
| III | **Config** | Store config in the **environment**, separate from code | Hardcoded URLs or credentials, `if env == "prod"` in code |
| IV | **Backing services** | Treat DBs, caches, and queues as attached resources via URLs | Code assuming local DB |
| V | **Build, release, run** | Strictly separate stages: build artifact → release (artifact + config) → run | Patching servers in place |
| VI | **Processes** | Stateless, share-nothing processes; state in backing services | In-memory sessions, local file uploads |
| VII | **Port binding** | Export services via port binding (self-contained server) | Deploying into an app server |
| VIII | **Concurrency** | Scale out via the process model (more processes/containers) | Only vertical scaling |
| IX | **Disposability** | Fast startup, **graceful shutdown**, robust against sudden death | 5-minute boot, no SIGTERM handling |
| X | **Dev/prod parity** | Keep environments similar (same backing service types) | SQLite in dev, Postgres in prod |
| XI | **Logs** | Treat logs as **event streams** to stdout; the platform routes them | Writing log files with rotation inside the app |
| XII | **Admin processes** | Run one-off admin tasks (migrations, scripts) as one-off processes with the same code and config | SSHing into prod to run ad-hoc scripts |

Modern additions (Kevin Hoffman's *Beyond the Twelve-Factor App*): **API first**, **telemetry** (metrics, traces, health), **authentication/authorization** everywhere, and graceful degradation.

---

## 2. Configuration {#config}

- **Separate config from code**: anything that varies between deploys (endpoints, feature toggles, pool sizes, log levels) comes from the environment.
- Sources (in precedence order, typically): defaults in code → config file per environment → environment variables → runtime overrides (flags service). Libraries: Spring Boot externalized config, Viper (Go), pydantic-settings, dotenv (local only), node-config/convict.
- **Validate config at startup and fail fast** (missing DB URL → crash immediately with a clear error, not at the first request at 3 AM). Use typed config objects.
- Kubernetes: ConfigMaps (non-secret) mounted as env vars or files. File mounts update live (env vars don't), so the app must reload or restart.
- Dynamic config (change without redeploy): feature flag services, AWS AppConfig, Consul KV, etcd. Watch for consistency (all instances converge), validation, and rollback.
- Keep config **declarative and versioned** (git), with review for production changes. Many outages are bad config pushes (e.g., the 2024 CrowdStrike incident was a content/config update), so roll config out progressively like code.

---

## 3. Secrets Management {#secrets}

Never:
- Commit secrets to git (use pre-commit scanning: gitleaks, trufflehog, GitHub secret scanning/push protection).
- Bake secrets into container images.
- Log secrets (redact them in logging middleware).
- Share one god-credential across services.

Do:
- Store them in a **secrets manager**: HashiCorp **Vault**, AWS Secrets Manager / SSM Parameter Store, GCP Secret Manager, Azure Key Vault, Doppler, 1Password Secrets Automation, Infisical.
- Inject at runtime: env vars (simple, but visible to the process and its children, and may leak via crash dumps), mounted files (tmpfs), or SDK fetch at startup. In Kubernetes, use the **External Secrets Operator**, Secrets Store CSI driver, Vault Agent injector, or sealed-secrets/SOPS for GitOps. (Native K8s Secrets are only base64 encoded, so enable encryption at rest in etcd and restrict RBAC.)
- **Prefer short-lived, dynamic credentials**: Vault dynamic DB credentials (a unique user per lease, auto-revoked), cloud IAM roles for workloads (IRSA / EKS Pod Identity, GKE Workload Identity, Azure Workload Identity), and RDS IAM auth tokens, instead of static passwords. **Workload identity** (SPIFFE/SPIRE) for service-to-service.
- **Rotate** regularly and automatically, and support dual credentials during rotation (accept old + new).
- Least privilege per service, plus audit logs of secret access.
- Encryption keys: KMS with envelope encryption, and never manage raw master keys in the app.

---

## 4. Feature Flags {#flags}

Decouple **deploy** (code in production) from **release** (feature visible to users).
Types:
| Type | Lifetime | Example |
|---|---|---|
| Release toggle | Days–weeks | Ship incomplete feature dark; enable gradually |
| Experiment toggle | Weeks | A/B tests |
| Ops toggle / kill switch | Long-lived | Disable expensive recommendations under load |
| Permission toggle | Long-lived | Premium features, beta users |

Practices:
- Progressive rollout by percentage, user segment, region, or internal users first, with **sticky bucketing** (hash of user ID so a user consistently sees the same variant).
- Evaluate server-side for security-sensitive features. Client SDKs cache rules and stream updates (LaunchDarkly, Unleash, Flagsmith, GrowthBook, Split, ConfigCat, OpenFeature as the vendor-neutral API standard, AWS AppConfig).
- **Clean up flags** after rollout (flag debt makes code paths combinatorial). Track flag age and owners.
- Test both paths, and default to safe behavior if the flag service is unreachable.
- Flags + trunk-based development enable continuous deployment without long-lived branches.

---

## 5. Containerizing Properly {#containers}

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.23 AS build                     # 1. build stage with toolchain
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download     # cached dependency layer
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot   # 2. minimal runtime: no shell, no package manager
COPY --from=build /out/app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]                               # exec form → app is PID 1 and receives SIGTERM
```
Checklist:
- **Multi-stage builds**: small runtime images (distroless, alpine, slim, chainguard), so faster pulls and a smaller attack surface.
- **Layer ordering for cache**: copy dependency manifests and install dependencies before copying source.
- **Run as non-root**, read-only root filesystem where possible, drop capabilities.
- **Exec-form ENTRYPOINT** so signals reach the app. If you need a shell wrapper, use `exec` in it, or an init like `tini`/`dumb-init` to reap zombies and forward signals.
- `.dockerignore` (exclude `.git`, `node_modules`, secrets, test data).
- Pin base image versions/digests, rebuild regularly for security patches, scan images (Trivy, Grype, Docker Scout), sign them (cosign/Sigstore), and generate an SBOM (syft).
- One process per container, logs to stdout/stderr, config via env/files, healthcheck endpoints.
- Container-aware runtimes: set memory and CPU limits awareness (JVM `-XX:MaxRAMPercentage=75`, which modern JVMs read from cgroups; Node `--max-old-space-size`; Go `GOMEMLIMIT` and `GOMAXPROCS` via automaxprocs, since Go before 1.25 doesn't respect CPU quotas by default).

---

## 6. Kubernetes Essentials {#k8s}

What a backend engineer should be able to read and write:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders-api }
spec:
  replicas: 3
  strategy: { type: RollingUpdate, rollingUpdate: { maxSurge: 25%, maxUnavailable: 0 } }
  selector: { matchLabels: { app: orders-api } }
  template:
    metadata: { labels: { app: orders-api } }
    spec:
      terminationGracePeriodSeconds: 40
      containers:
        - name: api
          image: registry.example.com/orders-api:1.42.0@sha256:...
          ports: [{ containerPort: 8080 }]
          env:
            - { name: DB_URL, valueFrom: { secretKeyRef: { name: orders-db, key: url } } }
          resources:
            requests: { cpu: "250m", memory: "256Mi" }     # scheduling guarantee
            limits:   { memory: "512Mi" }                  # OOMKill above this (CPU limits often omitted to avoid throttling)
          readinessProbe: { httpGet: { path: /readyz, port: 8080 }, periodSeconds: 5 }
          livenessProbe:  { httpGet: { path: /livez, port: 8080 }, periodSeconds: 10, failureThreshold: 3 }
          startupProbe:   { httpGet: { path: /livez, port: 8080 }, failureThreshold: 30, periodSeconds: 2 }
          lifecycle: { preStop: { exec: { command: ["sleep", "10"] } } }
      topologySpreadConstraints:
        - { maxSkew: 1, topologyKey: topology.kubernetes.io/zone, whenUnsatisfiable: ScheduleAnyway, labelSelector: { matchLabels: { app: orders-api } } }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: orders-api }
spec: { minAvailable: 2, selector: { matchLabels: { app: orders-api } } }
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: orders-api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: orders-api }
  minReplicas: 3
  maxReplicas: 30
  metrics: [{ type: Resource, resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } } }]
```
Key concepts: Pods, Deployments (stateless), StatefulSets (stable identity/storage), Jobs/CronJobs, Services (ClusterIP/NodePort/LoadBalancer, headless), Ingress/Gateway API, ConfigMaps/Secrets, resource requests and limits (QoS classes), probes, HPA/VPA/KEDA (event-driven autoscaling on queue lag), PodDisruptionBudgets, affinity/topology spread, namespaces, RBAC, NetworkPolicies.

Common backend pitfalls in K8s: no readiness probe (traffic before warm-up), liveness probe checking the DB (restart storms), no preStop delay (502s during rollout), memory limit below real peak (OOMKilled loops), CPU limits causing throttling latency spikes, DNS `ndots` overhead, and connection pools × replicas exceeding DB limits as HPA scales out.

---

## 7. Graceful Startup and Shutdown {#graceful}

**Startup**:
1. Load and validate config. Fail fast.
2. Initialize connection pools (optionally warm them), caches, and clients.
3. Run startup checks (can connect to the DB?) and report them via the readiness probe, not liveness.
4. Start the HTTP server. Mark **ready** only when warm (JIT warm-up for JVM: some teams send synthetic requests before readiness).

**Shutdown** on SIGTERM:
1. Mark **not ready** (fail readiness), so the LB/endpoints stop sending new traffic. Wait a few seconds for propagation (preStop sleep).
2. Stop accepting new connections. Close idle keep-alive connections (`Connection: close`, HTTP/2 GOAWAY).
3. **Finish in-flight requests** within a deadline (shorter than `terminationGracePeriodSeconds`).
4. Stop background consumers: stop polling, finish or requeue current jobs, commit offsets.
5. Flush telemetry (logs, traces, metrics), close DB pools.
6. Exit 0. If the deadline is exceeded, the orchestrator sends SIGKILL.

```go
srv := &http.Server{Addr: ":8080", Handler: mux}
go srv.ListenAndServe()
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
<-ctx.Done(); stop()
ready.Store(false)                         // readiness now fails
time.Sleep(5 * time.Second)                // let endpoints propagate
shutdownCtx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
defer cancel()
_ = srv.Shutdown(shutdownCtx)              // waits for in-flight requests
consumer.Close(); db.Close(); tracerProvider.Shutdown(shutdownCtx)
```

---

## 8. CI/CD Pipeline {#cicd}

Typical pipeline for a service (GitHub Actions, GitLab CI, Jenkins, Buildkite, CircleCI):
```
PR opened:
  lint + format check → unit tests → build → integration tests (Testcontainers) → contract tests
  → security scans (SAST: Semgrep/CodeQL; dependency: Dependabot/Renovate/Snyk/OSV; secrets scan; IaC scan)
  → build image → scan image → preview environment (optional) → code review approval
Merge to main:
  build once (immutable artifact tagged with git SHA) → push to registry → sign + SBOM
  → deploy to staging (run migrations) → smoke/e2e tests → deploy to production progressively (canary)
  → automated verification (metrics/SLO checks) → full rollout or automatic rollback
```
Principles:
- **Build once, promote the same artifact** across environments (never rebuild per environment).
- Fast feedback: keep PR pipelines under ~10 minutes (parallelize, cache dependencies, use test impact analysis).
- **Trunk-based development** with short-lived branches + feature flags lets you deploy many times a day.
- Deployment frequency, lead time, change failure rate, and time to restore are the **DORA metrics**.
- Supply-chain security: pinned actions/dependencies, provenance (SLSA levels), signed artifacts, least-privilege CI credentials (OIDC to cloud, no long-lived keys).
- Rollback must be one click (or automatic). Practice it.

---

## 9. Deployment Strategies {#deploy}

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Recreate** | Stop all old, start all new | Simple; no version mixing | Downtime |
| **Rolling update** | Replace instances gradually (maxSurge/maxUnavailable) | No downtime; default in K8s | Old and new versions run simultaneously (must be compatible); slow rollback |
| **Blue-green** | Two full environments; switch traffic at once (LB/DNS) | Instant switch & rollback; test green before switch | 2× capacity during deploy; DB schema must work for both |
| **Canary** | Send a small % of traffic (1% → 5% → 25% → 100%) to the new version, analyze metrics at each step | Limits blast radius; real-traffic validation | Needs traffic splitting + good metrics; slower |
| **Progressive delivery** (automated canary analysis) | Canary + automated SLO comparison + auto-rollback (Argo Rollouts, Flagger, Spinnaker/Kayenta, AWS CodeDeploy) | Safe at scale | Tooling investment |
| **Shadow / traffic mirroring** | Copy production traffic to the new version; discard its responses | Zero user impact testing | Side effects must be disabled/stubbed; double load |
| **Feature-flag release** | Deploy dark, release via flags | Decouples deploy/release, fine-grained targeting | Flag debt |
| **A/B testing** | Variants for experiments, by user segment | Business learning | Not a safety mechanism per se |
| **Cell/region waves** | Deploy cell-by-cell or region-by-region with bake time | Contains bad deploys | Slower global rollout |

**Version compatibility** during rolling/canary/blue-green deploys: the old and new versions coexist, so APIs, message formats, cache formats, and **database schemas** must be backward and forward compatible for at least one version (expand/contract).

---

## 10. Database Migrations in the Pipeline {#migrations}

- Run migrations as a **separate step** (Kubernetes Job, init step in the pipeline) **before** deploying code that needs them, never concurrently from every app instance at startup (races; use a migration lock if you must).
- Migrations must be compatible with the currently running code: **expand → deploy → contract** (see [`databases/fundamentals/06-schema-design-and-migrations.md`](../databases/fundamentals/06-schema-design-and-migrations.md)).
- Long-running data backfills run as background jobs, not inside the deploy.
- Lint migrations in CI (squawk, strong_migrations) for locking hazards.

---

## 11. Environments and Parity {#envs}

- Typical: local → CI → dev/integration → staging (production-like) → production. Fewer environments with better automation often beats many drifting ones.
- **Preview (ephemeral) environments** per PR (Vercel/Netlify for frontends; Kubernetes namespaces with Argo CD ApplicationSets, Okteto, Signadot, Uffizzi for backends), seeded with test data.
- **Local development**: Docker Compose for backing services, Testcontainers in tests, Tilt/Skaffold/Telepresence for K8s inner loops, dev containers.
- Parity: same DB engine and major version, same message broker, and the same config mechanism as prod. Use production-like data volumes for performance testing (anonymized copies).

---

## 12. Infrastructure as Code and GitOps {#iac}

- **IaC**: Terraform/OpenTofu, Pulumi (real languages), AWS CDK, CloudFormation, Crossplane (K8s-native). Benefits: reviewable, reproducible, versioned infrastructure. State management, drift detection, modules, and policy as code (OPA/Conftest, Sentinel, Checkov) are the operational concerns. See [`interview-prep/devops/09-terraform-ansible-iac.md`](../interview-prep/devops/09-terraform-ansible-iac.md).
- **GitOps** (Argo CD, Flux): the desired cluster state lives in git, and a controller continuously reconciles the cluster to match. Deploys happen via PRs that change the image tag, giving an audit trail, easy rollback (git revert), and drift correction.
- Helm charts / Kustomize overlays for K8s manifests per environment.

---

## 13. Production Readiness Checklist {#readiness}

Before a service takes production traffic:
- [ ] **Ownership**: owning team, on-call rotation, entry in the service catalog, runbooks.
- [ ] **Observability**: structured logs with request IDs, RED metrics, traces, dashboards, SLOs with alerts on burn rate (see [`observability/`](../observability/)).
- [ ] **Health**: liveness, readiness, and startup probes; graceful shutdown.
- [ ] **Resilience**: timeouts on all outbound calls, retries with backoff and budgets, circuit breakers, bulkheads, fallbacks for non-critical dependencies.
- [ ] **Capacity**: load tested to a known limit; autoscaling configured; resource requests and limits set; DB connection math done.
- [ ] **Availability**: ≥ 2–3 replicas across AZs; PDB; no single points of failure.
- [ ] **Data**: backups and tested restores; migrations reviewed; PII classified and protected; retention policies.
- [ ] **Security**: authN/authZ, secrets in a manager, least privilege IAM, dependency and image scanning, TLS everywhere, threat model reviewed.
- [ ] **Deploys**: automated CI/CD, canary or progressive rollout, one-click rollback, feature flags for risky changes.
- [ ] **Dependencies**: documented upstreams and downstreams, rate limits known, contracts tested.
- [ ] **Docs**: API docs (OpenAPI), architecture diagram, ADRs (Architecture Decision Records) for key choices.

---

## 14. Interview Questions {#qa}

1. Explain the twelve-factor app. Which factors do teams most often violate?
2. How should a service handle configuration and secrets in Kubernetes?
3. Why are short-lived credentials better than static passwords? How do you get them?
4. What are feature flags used for, and what are their risks?
5. What makes a good production Dockerfile?
6. Explain liveness vs readiness vs startup probes. What happens if liveness checks the database?
7. Walk through graceful shutdown of an HTTP service in Kubernetes.
8. Compare rolling, blue-green, and canary deployments.
9. How do you deploy a database schema change safely alongside a code change?
10. What would you check in a production readiness review?
