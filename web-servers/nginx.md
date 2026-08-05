# Nginx

A high-performance, event-driven web server and reverse proxy built for low memory usage and extreme concurrency. It serves static files, terminates TLS, proxies HTTP/WebSockets/gRPC, load-balances across app servers (Node.js, Python, Ruby, Go, PHP-FPM), and commonly sits as an ingress/sidecar in container and Kubernetes environments.

## TL;DR
- Event-driven, non-blocking I/O — one master process + a small number of worker processes handle huge connection counts with low memory, unlike Apache's process/thread-per-connection model.
- Config is a nested block structure: `main` → `events` → `http` → `server` → `location`, plus `upstream` for load-balancing pools and `stream` for raw TCP/UDP.
- Core jobs: reverse proxy, load balancer, TLS termination, static file server, cache layer, protocol upgrader (WebSocket/gRPC/HTTP2/HTTP3).
- No `.htaccess` — all config is centralized, which is faster and more auditable but less flexible per-directory than Apache.
- Reload config without dropping connections: `nginx -t && nginx -s reload`.

## Where nginx fits: reverse proxy vs load balancer vs API gateway

These three terms get used interchangeably but describe different responsibilities. Nginx can play any of these roles — sometimes all three in the same config — which is exactly why they get conflated.

- **Reverse proxy**: sits in front of one or more backend servers and forwards client requests to them, returning the response as if it served it itself. The core mechanism is just `proxy_pass`. Nginx-as-reverse-proxy is about *where the request goes* — hiding backend topology, terminating TLS, handling protocol upgrades.
- **Load balancer**: a reverse proxy with more than one backend and a policy for picking which one gets each request (`round_robin`, `least_conn`, `ip_hash`, etc.), typically with health checks to route around dead nodes. Every load balancer is a reverse proxy; not every reverse proxy is load-balancing (a single-backend `proxy_pass` isn't).
- **API gateway**: a reverse proxy/load balancer with an application-aware layer on top — authentication/authorization, per-client rate limiting, request/response transformation, routing by API version or tenant, aggregating multiple backend calls into one response, protocol translation (REST-to-gRPC), and centralized API analytics. Nginx *can* do a lot of this (rate limiting, auth via `auth_request`, header rewriting) but dedicated gateways (Kong, Envoy, AWS API Gateway, Apigee) build the application/business logic in as first-class features rather than assembled from lower-level directives.

Rule of thumb: if the concern is "get this request to a healthy backend efficiently," you're doing reverse-proxying/load-balancing — nginx is a natural fit and is what most people mean by "nginx in front of my app." If the concern is "enforce API contracts, quotas, and auth policy across many services owned by different teams," you're in API-gateway territory — nginx can approximate it for simple cases, but a purpose-built gateway (or nginx plus a module like `njs`/OpenResty, or NGINX Plus) pays off once the policy surface grows. In practice, many architectures layer them: `client → API gateway (authn, quotas, routing) → nginx (TLS termination, LB, caching) → app servers`.

## Installation

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y nginx

# RHEL/CentOS/Rocky/Alma
sudo dnf install -y nginx

# macOS (Homebrew)
brew install nginx
brew services start nginx
```

Validate and control via systemd:
```bash
nginx -v
nginx -t                          # test config syntax

sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl reload nginx
sudo systemctl status nginx | cat
```

Directory layout (Debian/Ubuntu):
- `nginx.conf` at `/etc/nginx/nginx.conf`
- Site configs under `/etc/nginx/sites-available/`, symlinked into `/etc/nginx/sites-enabled/`
- Global snippets in `/etc/nginx/snippets/`
- Logs in `/var/log/nginx/`

## Architecture and core concepts

- **Master process** — reads config, binds ports, spawns workers, handles reloads.
- **Worker processes** — handle connections using epoll (Linux) / kqueue (BSD/macOS), each capable of managing thousands of simultaneous connections via non-blocking I/O.
- **Contexts** (config blocks, nested):
  - `main` — global settings (user, worker count, PID file).
  - `events` — worker/connection tuning (`worker_connections`, `multi_accept`).
  - `http` — HTTP server settings shared across virtual hosts.
  - `server` — a virtual host (one `server_name` + `listen` combination).
  - `location` — path-based routing within a server block.
  - `upstream` — named pool of backend servers for load balancing.
  - `stream` — raw TCP/UDP (Layer 4) proxying, outside the `http` context.
- **Phases** — nginx processes each request through ordered phases (rewrite, access, content, log); directives only apply within their phase, which is why some directive combinations behave unexpectedly (e.g. `if` inside `location` interacting oddly with `try_files`).

## 🟢 Beginner: basic reverse proxy and static files

Minimal reverse proxy — `/etc/nginx/sites-available/app.conf`:
```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://127.0.0.1:3000;
    }
}
```
Enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

The `proxy_set_header` lines matter more than they look: without them, the backend sees every request as coming from `127.0.0.1` with `Host: 127.0.0.1`, breaking anything that depends on the real client IP or the original hostname (redirects, logging, rate limiting by IP).

Static files with SPA fallback and cache headers for assets:
```nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;  # SPA fallback: unknown paths go to index.html
    }

    location ~* \.(jpg|jpeg|png|gif|svg|css|js|woff2?)$ {
        access_log off;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}
```
`try_files` checks each argument in order and serves the first that exists on disk — the SPA fallback pattern (`$uri $uri/ /index.html`) is what lets client-side routers (React Router, Vue Router) handle deep links without 404s.

## 🟡 Intermediate: load balancing, protocol upgrades, TLS, caching

### Load balancing

Built-in algorithms: `round_robin` (default, no directive needed), `least_conn`, `ip_hash`, `hash` (with a key), and `random` (optionally with two-choice least-conn). Health checking is passive by default (`max_fails`/`fail_timeout`); active health checks require NGINX Plus.

```nginx
upstream app_pool {
    least_conn;
    server 10.0.0.11:3000 max_fails=3 fail_timeout=10s;
    server 10.0.0.12:3000 max_fails=3 fail_timeout=10s;
}

server {
    listen 80;
    server_name api.example.com;
    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://app_pool;
    }
}
```

Session affinity (sticky sessions) via `ip_hash` — same client IP always lands on the same backend:
```nginx
upstream app_pool { ip_hash; server 10.0.0.11:3000; server 10.0.0.12:3000; }
```

### WebSocket and gRPC proxying

WebSocket requires forwarding the `Upgrade`/`Connection` handshake headers explicitly — nginx doesn't do this automatically:
```nginx
location /ws/ {
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_http_version 1.1;
    proxy_pass http://127.0.0.1:8080;
}
```

gRPC rides on HTTP/2, so the listener needs `http2` and the `grpc_pass` directive instead of `proxy_pass`:
```nginx
upstream grpc_backend { server 127.0.0.1:50051; }

server {
    listen 443 ssl http2;
    server_name grpc.example.com;
    ssl_certificate /etc/letsencrypt/live/grpc.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/grpc.example.com/privkey.pem;

    location / {
        grpc_pass grpc://grpc_backend;
    }
}
```

### TLS/SSL

Basic TLS server with HSTS, modern ciphers, and HTTP→HTTPS redirect:
```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header Referrer-Policy no-referrer-when-downgrade;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

Let's Encrypt via Certbot:
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com --redirect --hsts --agree-tos -m admin@example.com --non-interactive
```

HTTP/3 (QUIC) requires nginx built with QUIC support (1.25+, `--with-http_v3_module`, plus a QUIC-capable TLS library like BoringSSL or OpenSSL 3 with QUIC):
```nginx
server {
    listen 443 http3 reuseport;
    listen 443 ssl http2;
    # ... ssl_certificate config ...
    add_header Alt-Svc 'h3=":443"; ma=86400';
}
```

### Caching

Proxy cache for API responses:
```nginx
proxy_cache_path /var/cache/nginx keys_zone=api_cache:100m max_size=10g inactive=60m use_temp_path=off;

server {
    location /api/ {
        proxy_cache api_cache;
        proxy_cache_valid 200 301 302 10m;
        proxy_cache_valid any 1m;
        add_header X-Cache-Status $upstream_cache_status;
        proxy_pass http://127.0.0.1:3000;
    }
}
```

FastCGI cache for PHP-FPM:
```nginx
fastcgi_cache_path /var/cache/nginx/fastcgi levels=1:2 keys_zone=fcgicache:100m inactive=60m;
location ~ \.php$ {
    include fastcgi_params;
    fastcgi_pass unix:/run/php/php8.2-fpm.sock;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_cache fcgicache;
    fastcgi_cache_valid 200 10m;
}
```

`$upstream_cache_status` (HIT/MISS/BYPASS/EXPIRED) exposed as a response header is the fastest way to debug whether caching is actually working.

## 🔴 Advanced: performance tuning, security, observability

### Performance tuning

`/etc/nginx/nginx.conf`:
```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 4096;  # raise based on ulimit -n
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 4096;

    gzip on;
    gzip_comp_level 5;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript application/xml+rss;

    # If compiled with Brotli
    # brotli on; brotli_comp_level 5; brotli_types text/plain text/css application/json application/javascript;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

Kernel/OS tuning for high connection counts: raise `fs.file-max`, `net.core.somaxconn`, and the relevant `ulimit -n`. Use `reuseport` on multi-queue NICs and check IRQ affinity / RPS / RFS tuning for very high throughput.

### Security hardening

```nginx
# Run as non-root worker (set in main context)
# user www-data;

# Limit HTTP methods
if ($request_method !~ ^(GET|HEAD|POST)$) { return 405; }

# Hide version info
server_tokens off;
```

Rate and connection limiting:
```nginx
limit_req_zone $binary_remote_addr zone=api_burst:10m rate=10r/s;
server { location /api/ { limit_req zone=api_burst burst=20 nodelay; } }

limit_conn_zone $binary_remote_addr zone=addr:10m;
server { location / { limit_conn addr 20; } }
```

Size limits and timeouts (protects against slow-loris style attacks and oversized uploads):
```nginx
client_max_body_size 10m;
client_body_timeout 15s;
send_timeout 30s;
```

Security headers:
```nginx
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header Referrer-Policy strict-origin-when-cross-origin;
add_header Content-Security-Policy "default-src 'self'";
```

### Logging and observability

Structured JSON logging for ingestion into ELK/Datadog:
```nginx
log_format json_combined '{"time":"$time_iso8601","remote_addr":"$remote_addr","request":"$request","status":$status,"bytes_sent":$bytes_sent,"referer":"$http_referer","user_agent":"$http_user_agent","request_time":$request_time,"upstream":"$upstream_addr","upstream_time":"$upstream_response_time"}';

access_log /var/log/nginx/access.json json_combined;
error_log /var/log/nginx/error.log warn;
```

Built-in metrics via `stub_status` (pair with `nginx-prometheus-exporter` for real monitoring):
```nginx
server {
    listen 127.0.0.1:8080;
    location /nginx_status { stub_status; allow 127.0.0.1; deny all; }
}
```

Rotate logs with `logrotate`.

### Zero-downtime reloads

```bash
nginx -t                          # validate first, always
sudo systemctl reload nginx       # or: nginx -s reload — reloads without dropping connections
```
Combine with `proxy_next_upstream`, health checks (`max_fails`/`fail_timeout`), to survive backend restarts gracefully.

## Common troubleshooting

| Symptom | Cause / fix |
|---|---|
| Port already in use | Check `sudo ss -ltnp \| grep :80` for a conflicting process |
| Permission denied on `listen 80` | Need root/CAP_NET_BIND_SERVICE, or use `authbind` |
| 502/504 from upstream | Check upstream health/connectivity and `proxy_read_timeout` |
| Large file uploads fail | Raise `client_max_body_size` |
| WebSocket disconnects | Raise `proxy_read_timeout`; confirm `Upgrade`/`Connection` headers are forwarded |
| Nothing helps | `tail -f /var/log/nginx/error.log /var/log/nginx/access.log` |

## Best practices checklist

- Set `worker_processes auto;` and size `worker_connections` to your `ulimit -n`.
- Keep TLS config current; enable HTTP/2, consider HTTP/3 once stable for your stack.
- Terminate TLS at nginx; talk to upstreams over loopback or a private network.
- Implement rate limiting and sane timeouts on every public-facing `location`.
- Ship structured logs to a central store.
- Template configs and validate them in CI (`nginx -t`).
- Separate static and API concerns; cache aggressively where it's safe to.

## Further reading
- Nginx official docs: `https://nginx.org/en/docs/`
- Mozilla server-side TLS guide: `https://mozilla.github.io/server-side-tls/`
- Certbot: `https://certbot.eff.org/`
