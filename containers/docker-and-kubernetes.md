# Docker and Kubernetes

Docker packages an application with its dependencies into a portable, reproducible
unit. Kubernetes runs many of those units, reliably, across many machines. They solve
different problems and are usually learned together, but they're not the same layer —
Docker builds and runs a single container; Kubernetes schedules and manages containers
across a cluster.

## TL;DR
- A container is **not** a lightweight VM. It's a normal process on the host, isolated
  using Linux **namespaces** (what it can see) and **cgroups** (what it can use). There
  is no hypervisor, no separate kernel — this is why containers start in milliseconds
  and VMs take seconds to minutes.
- Images are built in **layers**; Docker caches layers, so ordering your Dockerfile
  from least-to-most-frequently-changing instructions makes rebuilds fast.
- **Multi-stage builds** compile in one stage and copy only the final artifact into a
  slim runtime stage — production images shouldn't ship compilers, build tools, or dev
  dependencies.
- **Docker Compose** is for local multi-container development on one machine. It is
  not an orchestrator for production — it doesn't scale across hosts or self-heal.
- **Kubernetes** exists because production needs more than "run this container":
  scheduling across many hosts, restarting crashed containers, rolling updates without
  downtime, and scaling based on load.
- Core K8s objects: **Pod** (smallest deployable unit, usually one container),
  **Deployment** (manages replica Pods + rolling updates), **Service** (stable network
  identity for a set of Pods), **ConfigMap/Secret** (configuration and sensitive
  config), **Ingress** (external HTTP routing), **Namespace** (logical isolation).
- K8s Secrets are **base64-encoded, not encrypted**, by default — this trips up nearly
  everyone the first time. Base64 is an encoding, not encryption; anyone with API
  access (or etcd access) can trivially decode it.

## Docker fundamentals

### What a container actually is

The single most common misconception: "a container is a lightweight virtual machine."
It is not. Understanding the actual mechanism matters for reasoning about security,
performance, and what containers can and can't isolate.

A **virtual machine** virtualizes hardware. A hypervisor (e.g. KVM, Hyper-V) emulates a
full machine, and each VM runs its own complete operating system kernel on top of that
emulated hardware. That's why VMs take real memory/disk overhead and take seconds to
boot — you're booting an entire OS.

A **container** is just a regular process running on the host's existing kernel, made
to *look* isolated using two Linux kernel features:

- **Namespaces** — control what a process can *see*. Each namespace type isolates one
  resource:
  - `pid` namespace — the container sees its own process tree (its process looks like
    PID 1), can't see host processes.
  - `net` namespace — its own network stack, interfaces, routing table, ports.
  - `mnt` namespace — its own filesystem mount points (this is what makes the
    container's filesystem look like a fresh OS image even though it's really just a
    directory tree on the host).
  - `uts` namespace — its own hostname.
  - `ipc`, `user` namespaces — isolate inter-process communication and user/group ID
    mappings.
- **cgroups** (control groups) — control what a process can *use*: CPU shares, memory
  limits, I/O bandwidth, number of processes. This is what "resource limits" in Docker
  and Kubernetes actually configure under the hood.

So a container is: one process (or a small process tree), given its own namespaced view
of PIDs/network/filesystem, with cgroup limits capping its resource usage, all still
running directly on the **host's single shared kernel**. No hypervisor, no second
kernel, no emulated hardware.

**Why this distinction matters in practice**:
- **Startup time** — a container starts as fast as launching a process (milliseconds)
  because there's no OS to boot. A VM boots a kernel (seconds to tens of seconds).
- **Density** — you can run far more containers than VMs on the same hardware, because
  you're not paying for N duplicate kernels' worth of memory overhead.
- **Isolation strength** — this is the trade-off. Because containers share the host
  kernel, a kernel-level vulnerability or misconfiguration (e.g. running as root inside
  a container with excess capabilities) can potentially escape to the host or affect
  other containers in a way that's structurally impossible with a proper VM boundary.
  Containers are good process isolation, not the same security boundary as a hypervisor.
  This is exactly why running containers as non-root and dropping Linux capabilities
  matters (see Common Pitfalls).

### Images vs containers

An **image** is a read-only template: a filesystem snapshot plus metadata (entrypoint,
env vars, exposed ports). A **container** is a running (or stopped) *instance* of an
image — the image plus a thin writable layer on top for any runtime filesystem changes.

Analogy: image is a class, container is an instance of that class. You can run many
containers from the same image; each gets its own writable layer and process, but they
all share the same underlying read-only image layers on disk.

```bash
docker build -t myapp:1.0 .        # builds an image from a Dockerfile
docker run -d --name myapp-1 myapp:1.0   # creates + starts a container from that image
docker run -d --name myapp-2 myapp:1.0   # another independent container, same image
docker ps                           # list running containers
docker images                       # list images
```

### Layers and the build cache

A Docker image is built as a stack of **layers**, one per instruction in the Dockerfile
(roughly — `RUN`, `COPY`, `ADD` each create a layer; some instructions like `ENV` and
`LABEL` are metadata-only). Layers are content-addressed and cached: if an instruction
and everything before it are unchanged since the last build, Docker reuses the cached
layer instead of re-executing it.

This is why **instruction order matters**. Put things that change rarely (installing
system packages, installing dependencies) *before* things that change often (copying
your application source code) — otherwise every source code change invalidates the
cache for everything after it, and you re-run `npm install` or `pip install` on every
single build even though your dependencies didn't change.

```dockerfile
# Bad: any source change invalidates the dependency install cache
COPY . .
RUN npm install

# Good: dependency install is cached unless package.json actually changes
COPY package.json package-lock.json ./
RUN npm install
COPY . .
```

### A real Dockerfile, explained

A Node.js API example:

```dockerfile
# Base image: pin a specific version, not `latest` — reproducibility matters
FROM node:20-alpine AS base
# alpine = minimal Linux distro, much smaller than the default Debian-based image

WORKDIR /app
# All subsequent instructions run relative to /app inside the image

COPY package.json package-lock.json ./
# Copy only dependency manifests first — maximizes cache hits (see above)

RUN npm ci --omit=dev
# `npm ci` (not `npm install`) for reproducible installs from the lockfile;
# --omit=dev skips devDependencies since this is a production image

COPY . .
# Now copy the rest of the source — this layer changes on every code change,
# but everything above it is still cached

ENV NODE_ENV=production
# Environment variable baked into the image; overridable at `docker run` time

EXPOSE 3000
# Documentation only — does NOT actually publish the port; that happens at
# `docker run -p` or in docker-compose's `ports:`

USER node
# Run as the non-root `node` user baked into the official image, not root
# (see Common Pitfalls — this matters for real security reasons)

CMD ["node", "server.js"]
# The default command when the container starts (overridable at `docker run`)
```

Key instruction reference:

| Instruction | Purpose |
|---|---|
| `FROM` | Base image to build on top of |
| `WORKDIR` | Sets the working directory for subsequent instructions |
| `COPY` | Copies files from build context into the image |
| `ADD` | Like `COPY` but also handles remote URLs and auto-extracts archives — prefer `COPY` unless you specifically need those extras |
| `RUN` | Executes a command at *build* time, creates a new layer |
| `CMD` | Default command at *container start* time; overridable via `docker run <image> <cmd>` |
| `ENTRYPOINT` | Like `CMD` but harder to override — commonly paired with `CMD` supplying default args to it |
| `ENV` | Sets an environment variable, persists into the running container |
| `EXPOSE` | Documents which port the app listens on — informational, doesn't publish it |
| `USER` | Sets which user subsequent instructions and the container process run as |
| `ARG` | Build-time-only variable (not present in the final container, unlike `ENV`) |

### Multi-stage builds

A single-stage build that compiles code inevitably ships the compiler, build tools, and
often the full source tree and dev dependencies inside your production image — bloating
image size (slower pulls, slower deploys, bigger attack surface) for no runtime benefit.

**Multi-stage builds** solve this: use one stage to build the artifact, then copy just
the built output into a fresh, minimal runtime stage. Everything from the build stage
that isn't explicitly copied over is discarded.

```dockerfile
# ---- Build stage ----
FROM node:20 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
# Produces /app/dist — everything else (node_modules, source, build tools)
# stays in this stage and never reaches the final image

# ---- Production stage ----
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json /app/package-lock.json ./
RUN npm ci --omit=dev
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

A compiled-language example makes the size win even more dramatic — a Go binary needs
essentially no runtime at all:

```dockerfile
# ---- Build stage ----
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app ./cmd/server

# ---- Production stage ----
FROM scratch
# `scratch` is a literally empty base image — no shell, no libc, nothing.
# Works because CGO_ENABLED=0 produces a fully static binary.
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

That last example can produce a final image measured in single-digit megabytes, versus
potentially 1GB+ for a naive single-stage build that includes the full Go toolchain.

## Docker Compose for local development

Compose defines and runs multiple containers together on a single host, with shared
networking and declared dependencies — the standard way to run "my app + its database +
its cache" locally without manually wiring up `docker run` commands and networks by
hand.

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "3000:3000"          # host:container
    environment:
      - DATABASE_URL=postgres://user:pass@postgres:5432/appdb
      - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - .:/app                # bind mount for live code reload in dev
      - /app/node_modules      # anonymous volume: don't overwrite container's node_modules

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=appdb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data   # named volume: survives container restarts
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

```bash
docker compose up -d        # start everything in the background
docker compose logs -f app  # tail logs for one service
docker compose down         # stop and remove containers (keeps named volumes)
docker compose down -v      # also remove named volumes (wipes the DB data)
```

Notable mechanics:
- Services can reach each other by **service name** as a hostname (`postgres`,
  `redis`) — Compose sets up a private network and DNS automatically.
- `depends_on` controls start *order*, not full readiness by itself — pair it with
  `condition: service_healthy` and a `healthcheck` (as above) if the dependent service
  needs the DB to actually be accepting connections, not just started.
- Named volumes (`pgdata:`) persist data across `down`/`up`; bind mounts (`.:/app`) sync
  a host directory into the container, mainly useful for live-reload dev workflows.

⚠️ Compose is a **local development / single-host** tool. It has no concept of
scheduling across multiple machines, no self-healing across hosts, no rolling updates
across a fleet. `docker compose up` on one server is not a production deployment
strategy for anything that needs to survive that server dying.

## Why orchestration is needed beyond plain Docker

Running one container on one host with `docker run` doesn't answer questions that
production systems actually face:

- **What happens when the host dies?** Plain Docker: nothing, the container's gone. You
  need something watching across multiple hosts that can reschedule it elsewhere.
- **What happens when the container crashes?** Plain Docker: `--restart=always` helps
  on a single host, but doesn't help if the whole host is unreachable.
- **How do you scale to 10 replicas across 5 machines, load-balanced?** Plain Docker
  has no concept of "cluster" — you'd be manually SSHing into machines and running
  `docker run` on each, and manually wiring up a load balancer.
- **How do you roll out a new version without downtime**, and automatically roll back
  if the new version is broken? Plain Docker has no built-in rollout strategy.
- **How do you scale automatically based on load**, and how does a new pod discover
  where the database or other services live?

**Kubernetes** (and alternatives like Nomad, or managed equivalents like ECS) exist to
answer these: schedule containers across a fleet of machines, keep the desired number
of replicas running (self-healing), roll out changes gradually with automatic rollback
on failure, and scale based on demand.

## Kubernetes core concepts

### Pods — why not just containers directly

Kubernetes's smallest deployable unit is not a container — it's a **Pod**, a wrapper
around one or more containers that are always scheduled together, on the same node,
sharing the same network namespace (same IP, can reach each other via `localhost`) and
optionally shared volumes.

Most Pods run exactly one container — the extra wrapper exists for the cases where
tightly-coupled containers genuinely need to co-locate: a **sidecar** container (e.g. a
log shipper reading the main container's logs from a shared volume) or an **init
container** (runs to completion before the main container starts, e.g. running a DB
migration). Kubernetes schedules and scales at the Pod level, not the individual
container level — you don't independently scale one container inside a multi-container
Pod.

### Deployments — rolling updates, replica sets

You almost never create bare Pods directly in production — Pods are ephemeral and
don't self-heal on their own. A **Deployment** is a controller that manages a set of
identical Pods (via an intermediate object called a **ReplicaSet**) and handles:

- **Maintaining replica count** — if a Pod dies or a node fails, the Deployment's
  controller notices the actual state doesn't match the desired `replicas` count and
  schedules a replacement.
- **Rolling updates** — when you change the Pod template (e.g. a new image tag), the
  Deployment gradually replaces old Pods with new ones (default: a few at a time,
  configurable via `maxSurge`/`maxUnavailable`), rather than killing everything at once.
- **Rollback** — `kubectl rollout undo deployment/myapp` reverts to the previous
  ReplicaSet if the new version is bad.

### Services — ClusterIP, NodePort, LoadBalancer

Pods are ephemeral — they get new IPs whenever they're rescheduled. A **Service**
provides a stable network identity (a fixed virtual IP and DNS name) in front of a set
of Pods selected by label, with the Service load-balancing traffic across whichever
Pods currently match.

| Type | What it does | When to use |
|---|---|---|
| `ClusterIP` (default) | A stable virtual IP reachable only from *inside* the cluster | Internal service-to-service traffic — the default and most common case |
| `NodePort` | Opens a static port (30000-32767) on *every* node's IP, forwarding to the Service | Quick/dev-grade external access; rarely used directly in production |
| `LoadBalancer` | Provisions an actual external load balancer from the cloud provider (AWS ELB, GCP LB, etc.), pointing at the Service | Production external-facing traffic on a cloud platform |
| `ExternalName` | Maps the Service name to an external DNS name (no proxying) | Referring to something outside the cluster by an in-cluster name |

`ClusterIP` is the building block; `NodePort` and `LoadBalancer` are supersets of it
(each also gets a ClusterIP). In practice, most production HTTP traffic reaches the
cluster via an **Ingress** (below) sitting in front of `ClusterIP` services, rather than
via `LoadBalancer` Services directly per app.

### ConfigMaps and Secrets

Both decouple configuration from the container image, so the same image can run in dev,
staging, and prod with different config injected at deploy time.

- **ConfigMap** — non-sensitive configuration: feature flags, URLs, log levels.
- **Secret** — for sensitive values: passwords, API keys, TLS certs. Structurally
  almost identical to a ConfigMap, but ⚠️ **only base64-encoded, not encrypted, by
  default** — see Common Pitfalls, this is a genuinely common source of real security
  incidents when people assume "Secret" implies encryption at rest.

Both can be mounted into a Pod as environment variables or as files via a volume.

### Ingress controllers

A Service (even `LoadBalancer`) gives you one entry point per service — fine for one
app, unwieldy for many (a separate cloud load balancer per service gets expensive and
hard to manage). **Ingress** is a Kubernetes object that describes HTTP(S) routing
rules — host/path-based routing to different backend Services — enforced by an
**Ingress controller** (NGINX Ingress, Traefik, or a cloud-native one like AWS ALB
Ingress Controller) that actually implements the routing, usually behind a single
external load balancer.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: users-service
                port:
                  number: 80
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: orders-service
                port:
                  number: 80
  tls:
    - hosts: ["api.example.com"]
      secretName: api-example-com-tls
```

### Namespaces

Logical partitions within one cluster — a way to separate environments (`dev`,
`staging`, `prod`) or teams within a shared cluster, scoping resource names, RBAC
permissions, and resource quotas. Object names only need to be unique within a
namespace, not cluster-wide.

```bash
kubectl create namespace staging
kubectl apply -f deployment.yaml -n staging
kubectl get pods -n staging
```

## A real, minimal but complete example

A Deployment + Service for a simple web app:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3                      # run 3 identical Pods
  selector:
    matchLabels:
      app: myapp                   # must match template's labels below
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1             # at most 1 Pod down during a rollout
      maxSurge: 1                   # at most 1 extra Pod above `replicas` during rollout
  template:
    metadata:
      labels:
        app: myapp                  # Pods get this label; Service selects on it
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:1.4.2
          ports:
            - containerPort: 3000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: database-url
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: myapp-config
                  key: log-level
          resources:
            requests:                 # what scheduling guarantees this Pod
              cpu: "250m"              # 0.25 CPU core
              memory: "256Mi"
            limits:                   # hard cap enforced at runtime
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: ClusterIP
  selector:
    app: myapp                       # routes traffic to Pods with this label
  ports:
    - protocol: TCP
      port: 80                       # port the Service exposes
      targetPort: 3000               # port the container actually listens on
```

Line-by-line, what matters:
- `replicas: 3` + `selector`/`template.labels` matching is what lets the Deployment
  find and manage "its" Pods — the selector is the link between the Deployment and any
  Pod carrying that label, regardless of which ReplicaSet created it.
- `resources.requests` is what the **scheduler** uses to decide which node has room for
  this Pod; `resources.limits` is what the **kubelet/runtime** enforces at runtime (a
  container hitting its memory limit gets OOM-killed; hitting its CPU limit gets
  throttled, not killed).
- `readinessProbe` controls whether the Pod receives traffic from its Service (fails →
  removed from the Service's endpoints, but the Pod isn't killed) — used for "not ready
  yet" (e.g. still warming a cache) or "temporarily overloaded."
- `livenessProbe` controls whether Kubernetes considers the container dead and
  restarts it — used for "this process is stuck/deadlocked and a restart will fix it."
  Conflating these two is a very common mistake (see Common Pitfalls).
- The Service's `selector: app: myapp` is what connects it to the Deployment's Pods —
  note the Service doesn't reference the Deployment at all, only the label, which is
  also why Services keep working seamlessly across rolling updates (old and new Pods
  both carry the label during the transition).
- `targetPort` vs `port`: `port` is what other things call the Service on;
  `targetPort` is what the container inside the Pod is actually listening on — they
  don't have to match.

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get deployments
kubectl get pods -l app=myapp
kubectl rollout status deployment/myapp
kubectl set image deployment/myapp myapp=myregistry/myapp:1.5.0   # triggers rolling update
kubectl rollout undo deployment/myapp                              # rollback
```

## 🟢 Beginner

- Install Docker Desktop (or Docker Engine on Linux), write a Dockerfile for a small
  app, `docker build` and `docker run` it, understand `docker ps`/`docker logs`/`docker
  exec -it <container> sh`.
- Understand the image/container/layer distinction above, and why `.dockerignore` (like
  `.gitignore`, excludes files from the build context — commonly `node_modules/`,
  `.git/`, `.env`) speeds up builds and avoids leaking secrets into the image.
- Practice: containerize a simple app (any language), publish it to a registry
  (`docker push`), pull and run it on a different machine to confirm it "just works."

## 🟡 Intermediate

- Docker Compose for local multi-service dev (app + DB + cache), including
  healthchecks and `depends_on` ordering, named volumes vs bind mounts.
- Basic Kubernetes objects: Pods, Deployments, Services, ConfigMaps, Secrets,
  Namespaces — understand what each is for and write the YAML for a simple
  app-plus-database setup.
- `kubectl` fluency: `get`, `describe`, `logs`, `exec`, `apply`, `delete`,
  `rollout status`/`rollout undo`, `port-forward` for local debugging against a
  cluster.
- Understand `requests` vs `limits` and readiness vs liveness probes (both covered in
  the full example above) — these show up in essentially every real Deployment.

## 🔴 Advanced

- **Helm charts** — Kubernetes YAML gets repetitive fast across environments (dev vs
  staging vs prod Deployments that are 95% identical). Helm templates manifests with
  variables (`values.yaml`), and packages them as versioned, installable/upgradable
  "charts" — the closest thing Kubernetes has to a package manager.
  ```yaml
  # values.yaml
  replicaCount: 3
  image:
    repository: myregistry/myapp
    tag: "1.4.2"
  resources:
    requests: { cpu: 250m, memory: 256Mi }
    limits: { cpu: 500m, memory: 512Mi }
  ```
  ```yaml
  # templates/deployment.yaml (excerpt)
  spec:
    replicas: {{ .Values.replicaCount }}
    template:
      spec:
        containers:
          - name: myapp
            image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
            resources:
              {{- toYaml .Values.resources | nindent 14 }}
  ```
  ```bash
  helm install myapp ./mychart -f values-prod.yaml
  helm upgrade myapp ./mychart -f values-prod.yaml
  ```

- **Horizontal Pod Autoscaling (HPA)** — automatically adjusts `replicas` based on
  observed metrics (commonly CPU/memory, or custom metrics like queue depth via
  something like KEDA):
  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: myapp-hpa
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: myapp
    minReplicas: 3
    maxReplicas: 20
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 70
  ```
  Note this only works meaningfully if `resources.requests` is set (utilization % is
  computed relative to the request) — another reason requests/limits aren't optional
  in a serious setup.

- **Readiness vs liveness probes, deeper** — a wrong liveness probe is actively
  dangerous: if it fails during a legitimate slow startup or a temporary GC pause, K8s
  kills and restarts a perfectly healthy Pod, which can cascade (more Pods restart
  simultaneously → more load on survivors → more failures). Prefer generous
  `initialDelaySeconds`/`failureThreshold` on liveness, and use readiness (which only
  pulls traffic, doesn't kill anything) for anything more sensitive to transient states.
  A **startupProbe** exists specifically for slow-starting apps, to delay when
  liveness checks even begin.

- **StatefulSets** — Deployments assume Pods are interchangeable (any replica can be
  killed/replaced freely, none has a distinguishing identity). Stateful workloads
  (databases, distributed queues like Kafka) need stable, sticky identity: a
  predictable Pod name (`myapp-0`, `myapp-1`, ...), stable network identity per replica,
  and a dedicated persistent volume that follows that specific replica across
  rescheduling. `StatefulSet` provides exactly this, plus ordered, sequential
  scaling/rollout (important for e.g. a database primary/replica set where order
  matters). Rule of thumb: stateless app → Deployment; anything with its own durable,
  per-instance data and identity → StatefulSet (or, often better in practice, an
  operator-managed external database rather than running your primary datastore inside
  K8s at all).

- **Resource requests/limits and scheduling** — worth re-emphasizing at the advanced
  level: the scheduler bin-packs Pods onto nodes based on `requests`, not actual usage.
  Set requests too low and the scheduler over-packs a node, causing real contention
  under load even though "on paper" there was room. Set them too high and you waste
  cluster capacity. This tuning is an ongoing, real operational task, not a
  set-once-and-forget config.

## Common pitfalls

⚠️ **Containers running as root** — the Docker default. If an attacker achieves code
execution inside a root container, they have root *inside* the container's namespace,
which meaningfully increases the blast radius of any container escape or
misconfiguration versus a non-root process. Always set a `USER` in the Dockerfile (or
`securityContext.runAsNonRoot: true` in the Pod spec) unless you have a specific,
understood reason not to.

⚠️ **No resource limits set ("noisy neighbor")** — a Pod with no `resources.limits` can
consume unbounded CPU/memory on its node, starving every other Pod scheduled there.
This is one of the most common causes of "unrelated service degraded for no reason" in
shared clusters. Always set both requests and limits in production.

⚠️ **Secrets stored as plain ConfigMaps, or trusting Secret encryption that isn't
there** — two related mistakes. First, don't put passwords/API keys in a ConfigMap out
of laziness — use an actual `Secret` object. Second, and more subtly: **Kubernetes
Secrets are base64-encoded, not encrypted, by default.** Anyone with `kubectl get
secret -o yaml` access, or direct access to the underlying `etcd` datastore, can trivially
recover the plaintext — base64 is an encoding for safely representing binary data as
text, not a security mechanism. Production clusters need **encryption at rest for
etcd** (a K8s-level config, not automatic) and/or an external secrets manager (AWS
Secrets Manager, HashiCorp Vault, Sealed Secrets) integrated via something like the
External Secrets Operator, plus tight RBAC on who can `get`/`list` Secrets at all.

⚠️ **Image bloat from skipping multi-stage builds** — shipping build tools, dev
dependencies, and full source history in a production image slows every pull and
deploy, and needlessly increases attack surface (more installed packages = more
potential CVEs). Multi-stage builds (above) are close to free once set up — there's
rarely a good reason to skip them for anything beyond a throwaway script.

⚠️ **Treating Pods as pets, not cattle** — Pods are ephemeral by design: they get
rescheduled, replaced, and given new IPs constantly, including for entirely routine
reasons (node maintenance, autoscaling, a rolling update). Don't SSH into a specific Pod
to hand-fix something, don't rely on a Pod's IP being stable, don't store anything you
care about in a Pod's local filesystem without a PersistentVolume backing it. Design
for "any Pod can die at any moment and be replaced" as the baseline assumption, not an
edge case.

⚠️ **`latest` tag in production** — `image: myapp:latest` is not reproducible; a
redeploy or a new node pulling the image later can silently get a different build than
what you tested. Pin explicit, immutable tags (ideally including a digest) for anything
beyond local experimentation.

⚠️ **Missing `.dockerignore`** — without one, `docker build`'s context includes
`.git/`, `node_modules/`, and potentially `.env` files with real secrets, bloating
build time and risking secrets baked into image layers (which persist even if a later
layer "deletes" the file — layers are immutable and additive).

## Real-world context

Docker (2013) popularized containers for application delivery, though the underlying
kernel primitives (namespaces, cgroups) predate it by years (LXC used them earlier).
Kubernetes (open-sourced by Google in 2014, based on their internal Borg system) became
the dominant orchestrator, though alternatives exist: Docker Swarm (simpler, largely
faded from mainstream use), HashiCorp Nomad (simpler scheduler, not container-specific),
and fully managed offerings (AWS ECS/EKS, GCP GKE, Azure AKS) that run Kubernetes (EKS,
GKE, AKS) or a proprietary scheduler (ECS) so you don't operate the control plane
yourself. For most teams today, "using Kubernetes" in practice means a managed offering,
not self-hosting the control plane — self-hosting K8s is itself a significant
operational undertaking usually only justified at large scale or with specific
compliance/infra requirements.

## Further reading
- [Docker documentation](https://docs.docker.com/)
- [Kubernetes documentation](https://kubernetes.io/docs/home/)
- [Kubernetes Patterns](https://k8spatterns.io/) — Bilgin Ibryam & Roland Huß, common
  design patterns for K8s-native applications
- [The Illustrated Children's Guide to Kubernetes](https://www.cncf.io/phippy/the-childrens-illustrated-guide-to-kubernetes/) —
  genuinely useful as a quick mental model, not just a gimmick
- [`system-design/microservices-vs-monolith.md`](../system-design/microservices-vs-monolith.md) —
  containers and K8s are the usual deployment substrate for a microservices architecture
