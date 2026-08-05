## Apache HTTP Server (httpd): A Comprehensive Guide for Backend Engineers

Apache HTTP Server (httpd) is a highly extensible, module‑based web server widely used for hosting static and dynamic websites and as a reverse proxy/load balancer. It is process/thread‑based (via MPMs) and excels in flexibility and rich module ecosystem.

### Key Characteristics
- **MPMs (Multi-Processing Modules)**: `event` (asynchronous keep-alive handling), `worker` (threaded), `prefork` (process‑per‑request; legacy, use for non‑thread‑safe modules like old mod_php).
- **Rich module ecosystem**: `mod_ssl`, `mod_http2`, `mod_proxy_*`, `mod_brotli`, `mod_deflate`, `mod_security2`, `mod_status`, `mod_rewrite`, `mod_headers`, `mod_proxy_balancer`, `mod_proxy_wstunnel`, `mod_proxy_uwsgi`, `mod_proxy_fcgi`, etc.
- **Flexible configuration**: `VirtualHost`, per‑directory settings, `.htaccess` (can be disabled for performance).

### When to Use Apache
- Need advanced per‑directory access control and htaccess compatibility
- Hosting legacy PHP apps (via PHP‑FPM or, historically, mod_php)
- Complex rewrite/redirect logic compatible with `.htaccess` files
- Reverse proxy/load balancing with built‑in balancer manager

---

### Installation
- Ubuntu/Debian:
```bash
sudo apt update && sudo apt install -y apache2
```
- RHEL/CentOS/Rocky/Alma:
```bash
sudo dnf install -y httpd
```
- macOS (Homebrew):
```bash
brew install httpd
brew services start httpd
```
- Validate:
```bash
apachectl -v
apachectl configtest
```

Systemd operations:
```bash
sudo systemctl enable apache2    # Debian/Ubuntu
sudo systemctl start apache2
sudo systemctl reload apache2

sudo systemctl enable httpd      # RHEL family
sudo systemctl start httpd
sudo systemctl reload httpd
```

Directory layout (Debian/Ubuntu):
- Main config `/etc/apache2/apache2.conf`
- VHosts: `/etc/apache2/sites-available/`, symlink to `/etc/apache2/sites-enabled/`
- Modules: `/etc/apache2/mods-available/` and `mods-enabled/` (use `a2enmod`/`a2dismod`)
- Logs: `/var/log/apache2/`

Directory layout (RHEL):
- Main config `/etc/httpd/conf/httpd.conf`
- Extra conf `/etc/httpd/conf.d/*.conf`
- Logs `/var/log/httpd/`

---

### MPM Selection and Tuning
Check active MPM:
```bash
apachectl -V | grep -i mpm | cat
```
Set MPM (Debian example):
```bash
sudo a2dismod mpm_prefork mpm_worker
sudo a2enmod mpm_event
sudo systemctl reload apache2
```

Event MPM tuning (`/etc/apache2/mods-available/mpm_event.conf`):
```apache
<IfModule mpm_event_module>
  StartServers             2
  ServerLimit              32
  ThreadsPerChild          25
  MaxRequestWorkers        800
  MinSpareThreads          25
  MaxSpareThreads          75
  ThreadLimit              64
  MaxConnectionsPerChild   10000
</IfModule>
```
Adjust to your CPU/RAM profile and connection patterns.

---

### Basic Virtual Host
Create `/etc/apache2/sites-available/example.conf`:
```apache
<VirtualHost *:80>
  ServerName example.com
  ServerAlias www.example.com
  DocumentRoot /var/www/example

  <Directory /var/www/example>
    Options FollowSymLinks
    AllowOverride None
    Require all granted
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/example_error.log
  CustomLog ${APACHE_LOG_DIR}/example_access.log combined
</VirtualHost>
```
Enable and reload:
```bash
sudo a2ensite example
sudo apachectl configtest && sudo systemctl reload apache2
```

---

### Reverse Proxy
Enable proxy modules:
```bash
sudo a2enmod proxy proxy_http headers rewrite
```

Proxy config:
```apache
<VirtualHost *:80>
  ServerName api.example.com

  ProxyPreserveHost On
  ProxyRequests Off
  RequestHeader set X-Forwarded-Proto expr=%{REQUEST_SCHEME}
  RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"

  ProxyPass        / http://127.0.0.1:3000/
  ProxyPassReverse / http://127.0.0.1:3000/

  ErrorLog ${APACHE_LOG_DIR}/api_error.log
  CustomLog ${APACHE_LOG_DIR}/api_access.log combined
</VirtualHost>
```

WebSocket proxy (wss/ws):
```bash
sudo a2enmod proxy_wstunnel
```
```apache
ProxyPass "/ws/"  "ws://127.0.0.1:8080/"
ProxyPassReverse "/ws/"  "ws://127.0.0.1:8080/"
```

gRPC proxy:
```bash
sudo a2enmod proxy_http2 proxy_h2 proxy_fcgi
```
```apache
<VirtualHost *:443>
  ServerName grpc.example.com
  Protocols h2 http/1.1
  SSLEngine on
  SSLCertificateFile /etc/letsencrypt/live/grpc.example.com/fullchain.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/grpc.example.com/privkey.pem

  ProxyPass        "/"  "grpc://127.0.0.1:50051/"
  ProxyPassReverse "/"  "grpc://127.0.0.1:50051/"
</VirtualHost>
```

---

### Load Balancing
Enable balancer modules:
```bash
sudo a2enmod proxy_balancer lbmethod_byrequests lbmethod_bytraffic lbmethod_bybusyness
```

Configuration:
```apache
<Proxy "balancer://appcluster">
  BalancerMember "http://10.0.0.11:3000" retry=5
  BalancerMember "http://10.0.0.12:3000" retry=5
  ProxySet lbmethod=byrequests timeout=5
</Proxy>

<VirtualHost *:80>
  ServerName api.example.com
  ProxyPreserveHost On
  ProxyPass        "/"  "balancer://appcluster/"
  ProxyPassReverse "/"  "balancer://appcluster/"

  # Optional: Balancer Manager (protect in production!)
  <Location "/balancer-manager">
    SetHandler balancer-manager
    Require ip 127.0.0.1
  </Location>
</VirtualHost>
```

Session stickiness:
```apache
ProxyPass "/" "balancer://appcluster/" stickysession=JSESSIONID|sid
```

---

### TLS/SSL and HTTP/2
Enable modules:
```bash
sudo a2enmod ssl headers http2
```

VirtualHost with TLS + HSTS + HTTP/2:
```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
  ServerName example.com
  Protocols h2 http/1.1
  SSLEngine on
  SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
  Header set X-Content-Type-Options "nosniff"
  Header set X-Frame-Options "DENY"
  Header set Referrer-Policy "no-referrer-when-downgrade"

  ProxyPass        "/"  "http://127.0.0.1:3000/"
  ProxyPassReverse "/"  "http://127.0.0.1:3000/"
</VirtualHost>
</IfModule>

<VirtualHost *:80>
  ServerName example.com
  Redirect permanent "/" "https://example.com/"
</VirtualHost>
```

Let's Encrypt (Certbot):
```bash
sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache -d example.com -d www.example.com --redirect --hsts --agree-tos -m admin@example.com --non-interactive
```

HTTP/3 notes:
- Apache httpd does not natively support HTTP/3 as of most stable distros; deploy HTTP/3 via a front proxy (e.g., Nginx with QUIC, Caddy) or use experimental builds.

---

### PHP and Application Backends
- Prefer PHP‑FPM with `mod_proxy_fcgi`:
```bash
sudo a2enmod proxy_fcgi setenvif
sudo a2enconf php8.2-fpm   # Debian-based
```
```apache
<FilesMatch "\.php$">
  SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
</FilesMatch>
```

Python (uWSGI/WSGI) and others:
- `mod_proxy_uwsgi` for uWSGI, or reverse proxy to Gunicorn/Uvicorn via HTTP.

---

### Compression and Caching
- Gzip (mod_deflate):
```bash
sudo a2enmod deflate
```
```apache
AddOutputFilterByType DEFLATE text/plain text/css application/json application/javascript application/xml
```

- Brotli (mod_brotli):
```bash
sudo a2enmod brotli
```
```apache
AddOutputFilterByType BROTLI_COMPRESS text/plain text/css application/json application/javascript application/xml
BrotliCompressionQuality 5
```

- Static caching headers:
```apache
<LocationMatch "\.(jpg|jpeg|png|gif|svg|css|js|woff2?)$">
  Header set Cache-Control "public, max-age=31536000, immutable"
</LocationMatch>
```

---

### Security Hardening
- Hide server tokens:
```apache
ServerTokens Prod
ServerSignature Off
```
- Restrict methods:
```apache
<Location "/">
  <LimitExcept GET POST HEAD>
    Require all denied
  </LimitExcept>
```
- Security headers:
```apache
Header set X-Frame-Options "DENY"
Header set X-Content-Type-Options "nosniff"
Header set Referrer-Policy "strict-origin-when-cross-origin"
Header set Content-Security-Policy "default-src 'self'"
```
- Rate limiting and WAF:
  - Use `mod_evasive` for simple DoS mitigation.
  - Use `ModSecurity` (`mod_security2`) with OWASP CRS for WAF.
- Disable `.htaccess` where possible for performance and central policy:
```apache
AllowOverride None
```
- Directory permissions and `Options`:
```apache
<Directory />
  AllowOverride None
  Options -Indexes -Includes -ExecCGI
  Require all denied
</Directory>
```

---

### Logging and Observability
- Custom log formats (JSON):
```apache
LogFormat '{"time":"%t","remote_addr":"%a","host":"%v","request":"%r","status":%>s,"bytes":%B,"referer":"%{Referer}i","ua":"%{User-Agent}i","reqtime":%D}' json
CustomLog ${APACHE_LOG_DIR}/access.json json
ErrorLogFormat "[%{cu}t] [%-m:%l] [pid %P:tid %T] %7F: %E: [client %a] %M"
```
- Status module:
```bash
sudo a2enmod status
```
```apache
<Location "/server-status">
  SetHandler server-status
  Require ip 127.0.0.1
</Location>
```
- Log rotation via `logrotate` or `rotatelogs`:
```apache
CustomLog "|/usr/sbin/rotatelogs /var/log/apache2/access.%Y%m%d.log 86400" combined
```

---

### Zero‑Downtime Reloads and Control
```bash
apachectl configtest
apachectl graceful     # reload without dropping connections
systemctl reload apache2
```

---

### Troubleshooting
- 403/404: verify `DocumentRoot`, `<Directory>` permissions, and `Require` directives
- 502/504: upstream issues; adjust `ProxyTimeout`, check backend health
- SSL errors: certificate/key paths, `SSLCertificateFile` vs chain order
- Port conflicts: `sudo ss -ltnp | grep :80`
- High CPU/RAM: tune MPM `MaxRequestWorkers`, enable compression wisely, disable unused modules
- Logs: `tail -f /var/log/apache2/error.log /var/log/apache2/access.log`

---

### Best Practices Checklist
- Prefer `event` MPM; avoid `prefork` unless strictly required
- Use PHP‑FPM or app servers; avoid in‑process interpreters when not necessary
- Keep modules minimal; enable only what you need
- Centralize configuration; avoid `.htaccess` in production
- Use HTTP/2 with TLS; keep cipher suites modern; enable HSTS
- Monitor with `mod_status` and centralized logging

---

### References
- Apache httpd docs: `https://httpd.apache.org/docs/`
- Security: `https://httpd.apache.org/docs/2.4/misc/security_tips.html`
- ModSecurity + CRS: `https://coreruleset.org/`


