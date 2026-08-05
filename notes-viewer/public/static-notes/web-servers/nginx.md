## Nginx: A Comprehensive Guide for Backend Engineers

Nginx is a high‑performance, event‑driven web server and reverse proxy designed for low memory usage and extreme concurrency. It excels at serving static files, acting as an HTTP/HTTPS reverse proxy and load balancer, terminating TLS, proxying WebSockets/gRPC, and sitting in front of application servers (Node.js, Python, Ruby, Go, PHP‑FPM).

### Key Characteristics
- **Event-driven, non-blocking I/O**: Scales to hundreds of thousands of connections with low memory.
- **Master/worker architecture**: One master process manages multiple worker processes.
- **Modular configuration**: Clear separation via contexts: `main`, `events`, `http`, `server`, `location`, `upstream`, `stream`.
- **Reverse proxy and load balancer**: `proxy_pass` and `upstream` support multiple algorithms.
- **TLS termination and HTTP/2/3**: Handles certificates, ALPN, HSTS; HTTP/3 requires QUIC build.

### When to Use Nginx
- As a front proxy terminating TLS and forwarding to app servers
- For static content and asset caching
- For load balancing across multiple application instances
- For protocol upgrades: WebSocket, gRPC, HTTP/2, (optionally) HTTP/3/QUIC
- As a sidecar or ingress in container/K8s environments

---

### Installation
- Ubuntu/Debian:
```bash
sudo apt update && sudo apt install -y nginx
```
- RHEL/CentOS/Rocky/Alma:
```bash
sudo dnf install -y nginx
```
- macOS (Homebrew):
```bash
brew install nginx
brew services start nginx
```
- Validate:
```bash
nginx -v
nginx -t
```

Systemd operations:
```bash
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl reload nginx
sudo systemctl status nginx | cat
```

Directory layout (Debian/Ubuntu):
- `nginx.conf` at `/etc/nginx/nginx.conf`
- Site configs under `/etc/nginx/sites-available/` and symlinks to `/etc/nginx/sites-enabled/`
- Global snippets `/etc/nginx/snippets/`
- Logs in `/var/log/nginx/`

---

### Architecture and Core Concepts
- **Master process**: reads config, binds ports, spawns workers, handles reloads.
- **Workers**: handle connections using epoll/kqueue.
- **Contexts**:
  - `main` (global), `events` (worker/connection settings), `http` (HTTP server), `server` (virtual hosts), `location` (path-based routing), `upstream` (LB pools), `stream` (TCP/UDP L4 proxy).
- **Phases**: Nginx processes requests in phases (rewrite, access, content, log). Directives apply in specific phases.

---

### Minimal HTTP Reverse Proxy Example
Create `/etc/nginx/sites-available/app.conf`:
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

---

### Static Files and SPA Fallback
```nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;  # SPA fallback
    }

    location ~* \.(jpg|jpeg|png|gif|svg|css|js|woff2?)$ {
        access_log off;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}
```

---

### Load Balancing
Supported methods: `round_robin` (default), `least_conn`, `ip_hash`, `hash`, `random` (with two least_conn), health checks (basic by default; advanced with NGINX Plus) and circuit breaker patterns via fail_timeout/max_fails.

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

Session affinity (sticky) using `ip_hash`:
```nginx
upstream app_pool { ip_hash; server 10.0.0.11:3000; server 10.0.0.12:3000; }
```

---

### WebSocket and gRPC Proxying
- WebSocket requires `Upgrade` and `Connection` headers.
```nginx
location /ws/ {
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_http_version 1.1;
    proxy_pass http://127.0.0.1:8080;
}
```

- gRPC (HTTP/2) proxy:
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

---

### TLS/SSL Configuration
Basic TLS server with HSTS and modern ciphers:
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

Let's Encrypt (Certbot):
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com --redirect --hsts --agree-tos -m admin@example.com --non-interactive
```

HTTP/3 (QUIC) notes:
- Requires Nginx built with QUIC (nginx 1.25+ with `--with-http_v3_module` and a QUIC-capable TLS library such as BoringSSL or OpenSSL 3 with QUIC).
```nginx
server {
    listen 443 http3 reuseport;
    listen 443 ssl http2;
    # ... ssl_certificate config ...
    add_header Alt-Svc 'h3=":443"; ma=86400';
}
```

---

### Caching
- Static caching via `Cache-Control` headers (from app or Nginx add_header).
- Proxy cache:
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

FastCGI cache (PHP‑FPM):
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

---

### Performance Tuning
In `/etc/nginx/nginx.conf`:
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

Kernel/OS considerations:
- Increase `fs.file-max`, `net.core.somaxconn`, and relevant `ulimit -n` for high connections.
- Use `reuseport` for multi-queue NICs; ensure IRQ affinity and RPS/RFS are tuned.

---

### Security Hardening
- Run as non-root worker (`user www-data;`).
- Limit methods:
```nginx
if ($request_method !~ ^(GET|HEAD|POST)$) { return 405; }
```
- Disable server tokens:
```nginx
server_tokens off;
```
- Rate limiting:
```nginx
limit_req_zone $binary_remote_addr zone=api_burst:10m rate=10r/s;
server { location /api/ { limit_req zone=api_burst burst=20 nodelay; } }
```
- Connection limiting:
```nginx
limit_conn_zone $binary_remote_addr zone=addr:10m;
server { location / { limit_conn addr 20; } }
```
- Size limits and timeouts:
```nginx
client_max_body_size 10m;
client_body_timeout 15s;
send_timeout 30s;
```
- Security headers:
```nginx
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header Referrer-Policy strict-origin-when-cross-origin;
add_header Content-Security-Policy "default-src 'self'";
```

---

### Logging and Observability
- Access/error logs per server or globally.
- Custom `log_format`, including JSON for ingestion into ELK/Datadog:
```nginx
log_format json_combined '{"time":"$time_iso8601","remote_addr":"$remote_addr","request":"$request","status":$status,"bytes_sent":$bytes_sent,"referer":"$http_referer","user_agent":"$http_user_agent","request_time":$request_time,"upstream":"$upstream_addr","upstream_time":"$upstream_response_time"}';

access_log /var/log/nginx/access.json json_combined;
error_log /var/log/nginx/error.log warn;
```
- Metrics:
  - `stub_status`:
```nginx
server {
    listen 127.0.0.1:8080;
    location /nginx_status { stub_status; allow 127.0.0.1; deny all; }
}
```
  - Use an external exporter (e.g., `nginx-prometheus-exporter`).
- Log rotation via `logrotate`.

---

### Zero‑Downtime Reloads and Deployments
- Validate config: `nginx -t`
- Reload without dropping connections: `systemctl reload nginx` or `nginx -s reload`
- Use `proxy_next_upstream`, `health checks`, and `fail_timeout` to survive backend restarts.

---

### Common Troubleshooting
- Port already in use → check `sudo ss -ltnp | grep :80`
- Permission denied on `listen 80` (non-Linux) → need privileges or `authbind`
- 502/504 from upstream → check upstream health/connectivity and `proxy_read_timeout`
- Large file uploads → raise `client_max_body_size`
- WebSocket disconnects → set `proxy_read_timeout` higher and include upgrade headers
- Use logs: `tail -f /var/log/nginx/error.log /var/log/nginx/access.log`

---

### Best Practices Checklist
- Use `worker_processes auto;` and right `worker_connections`
- Keep TLS configs up to date; enable HTTP/2, consider HTTP/3 when stable
- Terminate TLS at Nginx; pass upstream via loopback or private network
- Implement rate limiting and sensible timeouts
- Keep logs structured; ship to central store
- Template configs and validate in CI (`nginx -t`)
- Separate concerns: static vs API, cache where safe

---

### References
- Nginx official docs: `https://nginx.org/en/docs/`
- Hardening guides: `https://mozilla.github.io/server-side-tls/`
- Certbot: `https://certbot.eff.org/`


