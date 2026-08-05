# Nginx vs Apache (httpd)

A decision-focused comparison across architecture, performance, configuration, and operations. For config examples and deep dives on either server, see [`nginx.md`](./nginx.md) and [`apache-http-server.md`](./apache-http-server.md) — this file stays a comparison, not a duplicate.

## TL;DR
- **Nginx**: event-driven, non-blocking I/O — one master + a handful of workers handle huge connection counts with low memory. Best for high concurrency, static assets, reverse proxy/load balancing, TLS termination, container/K8s ingress, WebSocket/gRPC.
- **Apache**: process/thread-based via MPMs — more flexible per-directory config (`.htaccess`), a much larger module ecosystem, and a built-in balancer-manager UI. Best for legacy LAMP stacks, complex `.htaccess`-driven rewrite logic, and deep `mod_*` needs.
- **Use both**: nginx at the edge (TLS, caching, WAF, static) in front of Apache (legacy app, complex rewrites) is a common, sane pattern — not an either/or in practice.

## Architecture

| | Nginx | Apache |
|---|---|---|
| Concurrency model | Event-driven, non-blocking I/O (epoll/kqueue) | Process/thread-based via MPMs (`event`, `worker`, `prefork`) |
| Memory per connection | Very low — workers multiplex thousands of connections each | Higher — `prefork` is one process per connection; `event`/`worker` narrow the gap but don't close it |
| Best under high concurrency | Excellent — scales to hundreds of thousands of idle/keep-alive connections | Good with `event` MPM; weak with legacy `prefork` |
| Blocking I/O handling | Never blocks a worker on I/O | Can block a thread/process depending on MPM and module (notably `prefork` + non-thread-safe modules) |

The architectural difference is the root cause of most other rows in this table: nginx's non-blocking model means a slow client or slow upstream doesn't tie up a full process/thread the way it can under Apache's older MPMs.

## Performance and resource usage

| | Nginx | Apache |
|---|---|---|
| Static files | Generally faster, lower CPU/memory | Good, but typically behind nginx |
| Dynamic apps (via FastCGI/proxy) | Comparable when proxying to the same app server | Comparable, especially with `event` MPM |
| Many idle/keep-alive connections | Excels — this is the classic C10K scenario nginx was built for | `event` MPM mitigates the old `prefork` weakness but stays heavier per connection |

## Reverse proxy and load balancing

Both support reverse proxying, load balancing, and health checks.

| | Nginx | Apache |
|---|---|---|
| Config style | Simple, declarative (`proxy_pass`, `upstream`) | More verbose (`mod_proxy_balancer`, `<Proxy>` blocks) |
| WebSocket/gRPC | First-class (`proxy_set_header Upgrade`, `grpc_pass`) | Supported via `proxy_wstunnel`, `proxy_http2` |
| L4 (TCP/UDP) proxying | Built-in via `stream` context | Not a native fit — Apache is primarily L7 |
| Active health checks | NGINX Plus only (open-source nginx is passive-only via `max_fails`) | Built into `mod_proxy_balancer` |
| Admin UI for balancer state | None built-in (open-source) | `balancer-manager` web UI |

## TLS / HTTP2 / HTTP3

| | Nginx | Apache |
|---|---|---|
| TLS termination | Excellent, mature | Excellent, mature |
| HTTP/2 | Stable | Stable via `mod_http2` |
| HTTP/3 (QUIC) | Available with a QUIC-capable build (1.25+, BoringSSL/OpenSSL 3) | Not in mainstream stable packages — needs a front proxy |

## Configuration and extensibility

| | Nginx | Apache |
|---|---|---|
| Config model | Centralized blocks (`http`/`server`/`location`/`upstream`); no `.htaccess` | Centralized `VirtualHost` + optional per-directory `.htaccess` overrides |
| Per-directory overrides | Not supported (by design) | `.htaccess` — flexible but a per-request filesystem cost if left enabled broadly |
| Module ecosystem | Smaller core, very stable | Very large: `mod_security2`, `mod_evasive`, `mod_rewrite`, `mod_proxy_*`, and more |
| Dynamic modules | Limited in open-source nginx (recompile or use `njs`/OpenResty for scripting) | `LoadModule` at runtime — easy to add/remove |

## Security

| | Nginx | Apache |
|---|---|---|
| Rate/connection limiting | Built-in (`limit_req`, `limit_conn`) | Needs `mod_evasive` or external tooling |
| WAF | External (ModSecurity port exists but is less common) or NGINX App Protect | First-class `ModSecurity` + OWASP CRS integration |
| Attack surface | Smaller when modules are kept minimal | Larger by default given the bigger module ecosystem — mitigate by disabling unused modules |
| Common misconfiguration risk | Lower — no `.htaccess` sprawl | `.htaccess` can be a foot-gun if left enabled without care (`AllowOverride None` is the fix) |

## Observability

Both expose access/error logs with custom formats (including JSON) and have Prometheus exporters available.

| | Nginx | Apache |
|---|---|---|
| Built-in status endpoint | `stub_status` (connection counts only) | `mod_status` (per-worker/request detail, more granular) |
| Structured logging | `log_format` directive | `LogFormat` + `ErrorLogFormat` |

## Admin and operations

| | Nginx | Apache |
|---|---|---|
| Config test | `nginx -t` | `apachectl configtest` |
| Zero-downtime reload | `nginx -s reload` / `systemctl reload nginx` | `apachectl graceful` / `systemctl reload apache2` |
| Common as K8s ingress | Very common (nginx-ingress is a default choice) | Less common |

## Common deployment patterns

**1. Nginx front, Apache app** — nginx handles TLS, caching, WAF, static assets; Apache runs the legacy app or PHP-FPM with complex rewrites behind it:
```
Internet → Nginx (TLS, caching, WAF, static) → Apache (PHP-FPM/legacy rewrites) → App
```

**2. Nginx only** — nginx as the full edge: TLS, reverse proxy, load balancing straight to app servers:
```
Internet → Nginx (TLS, reverse proxy, LB) → App servers (Node/Go/Python)
```

**3. Apache only** — Apache handles TLS termination and proxying/balancing directly:
```
Internet → Apache (TLS, proxy/balancer) → App servers (PHP-FPM, Python, etc.)
```

## Equivalent minimal reverse proxy

Nginx:
```nginx
server {
  listen 80;
  server_name api.example.com;
  location / {
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://127.0.0.1:3000;
  }
}
```

Apache:
```apache
<VirtualHost *:80>
  ServerName api.example.com
  ProxyPreserveHost On
  ProxyPass        "/"  "http://127.0.0.1:3000/"
  ProxyPassReverse "/"  "http://127.0.0.1:3000/"
</VirtualHost>
```

## Quick decision matrix

| Scenario | Pick |
|---|---|
| Greenfield microservices / high-concurrency APIs | Nginx |
| Legacy LAMP stack, heavy `.htaccess` use | Apache (nginx in front for TLS/caching if needed) |
| Kubernetes ingress | Nginx |
| Need built-in balancer-manager UI | Apache |
| HTTP/3 adoption today | Nginx (or another QUIC-capable proxy) in front of whichever serves the app |
| Deep WAF requirement with mature tooling | Apache + ModSecurity, or nginx + NGINX App Protect |

## Practical recommendations

- For greenfield microservices and high-concurrency APIs, prefer nginx at the edge.
- For legacy LAMP or heavy `.htaccess` use, Apache remains strong — put nginx in front for TLS/caching if you want the best of both.
- For HTTP/3 adoption today, place nginx (or another QUIC-capable proxy) in front of Apache.
- Keep configurations centralized, version-controlled, and validated in CI regardless of which you choose.

## Further reading
- Nginx docs: `https://nginx.org/en/docs/`
- Apache httpd docs: `https://httpd.apache.org/docs/`
- Mozilla server-side TLS guide: `https://mozilla.github.io/server-side-tls/`
- See also: [`nginx.md`](./nginx.md) for reverse-proxy/gateway/load-balancer terminology and full nginx config depth; [`apache-http-server.md`](./apache-http-server.md) for MPM tuning and module details.
