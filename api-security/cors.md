# CORS (Cross-Origin Resource Sharing)

CORS is the browser mechanism that lets a server say "requests from this other origin are allowed to read my responses." It's one of the most commonly misunderstood parts of web security — mostly because the error message you see in devtools ("No 'Access-Control-Allow-Origin' header is present") describes a browser-side block, not a server-side bug, and that trips people up constantly. This guide covers what it is, why it exists, exactly when preflight fires, the credentials+wildcard trap, and a concrete debugging walkthrough.

## TL;DR
- CORS **relaxes** the Same-Origin Policy (SOP) — it does not add security by itself; it's an opt-in *sharing* mechanism, not a firewall.
- **CORS is enforced by the browser, not the server.** The server just sends headers; the browser decides whether to let JS read the response. Tools like `curl` and Postman ignore CORS entirely — that's why "it works in Postman but not the browser" is not a paradox.
- **Simple requests** (GET/HEAD/POST with only safelisted headers and content-types) go straight through — no preflight.
- **Preflighted requests** (PUT/DELETE/PATCH, custom headers, `application/json`, etc.) trigger a browser-sent `OPTIONS` request first, to ask permission before sending the real one.
- `Access-Control-Allow-Origin: *` **cannot** be combined with `Access-Control-Allow-Credentials: true` — the spec forbids it, and browsers will reject the response if a server tries.
- The request always leaves the browser (the server *does* receive it) — CORS blocks the *response* from being handed to JavaScript, not the request from being sent. This is a critical, frequently-missed nuance, especially for non-idempotent (state-changing) simple requests.

## Table of contents
1. [What CORS is and why it exists](#what-is-cors)
2. [The Same-Origin Policy: the foundation](#same-origin-policy)
3. [What triggers CORS (and what doesn't)](#what-triggers-cors)
4. [How CORS works: the request/response flow](#how-cors-works)
5. [Simple requests](#simple-requests)
6. [Preflighted requests](#preflighted-requests)
7. [Credentials (cookies and auth) — and the wildcard trap](#credentials)
8. [Redirects in CORS](#redirects)
9. [Third-party cookies and CORS](#third-party-cookies)
10. [HTTP headers reference](#headers)
11. [Examples and code](#examples)
12. [Debugging a CORS error: step by step](#errors)
13. [Server-side configuration by framework](#server-config)
14. [Browser compatibility and edge cases](#compatibility)
15. [Security implications and best practices](#security)
16. [Advanced topics](#advanced)
17. [FAQs and myths](#faqs)
18. [Further reading](#resources)

<a name="what-is-cors"></a>
## 1. What CORS is and why it exists

**CORS (Cross-Origin Resource Sharing)** is an HTTP-header-based mechanism that lets a server tell browsers which other origins are allowed to read its responses via JavaScript.

### Why it exists: browsers default to blocking cross-origin reads

Browsers enforce the **Same-Origin Policy** by default: a script running on `https://a.com` cannot read the response of a request to `https://b.com` — even if the request itself succeeds. This exists to stop a malicious page from silently reading your data on sites you're logged into (your bank, your email) using your own browser session and cookies.

The problem: the modern web is built on cross-origin API calls by design — a React app on `app.example.com` calling `api.example.com`, a third-party widget calling its own backend, a public API meant to be consumed by many different frontends. SOP alone would break all of that. CORS is how a server *opts in*: it explicitly tells the browser "responses from me can be read by scripts running on this specific other origin."

**Analogy**: SOP is a locked door — the browser's default stance is "no cross-origin reads, full stop." CORS is the key a server can hand out: "resource requests from `https://foo.example` are fine, let them through." Without that key, the door stays locked no matter how the request itself went.

### Key definitions
- **Origin** = scheme + domain + port. Two URLs are the *same* origin only if all three match exactly.
  - `https://example.com` and `https://example.com/about` → same origin (path doesn't matter)
  - `http://example.com` vs `https://example.com` → different (scheme differs)
  - `https://example.com` vs `https://sub.example.com` → different (subdomain differs)
  - `https://example.com` vs `https://example.com:8080` → different (port differs)
- **Cross-origin request**: any request where the page's origin differs from the resource's origin in scheme, domain, or port.
- **Same-Origin Policy (SOP)**: the browser's default security boundary — scripts can only freely read responses from their own origin.

CORS doesn't replace SOP; it's a controlled, server-authorized exception to it.

### Why it matters practically
- **Security**: without SOP (and CORS as its escape hatch), any malicious site could read your logged-in session data from any other site just by having your browser make the request.
- **Historical context**: before Fetch/XHR, HTML forms could already submit cross-origin (that's how `<form action="https://other.com">` has always worked) — but the response wasn't readable by the page's JS. CORS extends controlled cross-origin access to modern APIs (Fetch, XHR) while keeping the "response is unreadable by default" guarantee unless explicitly allowed.
- **Real-world use**: any SPA calling an API on a different subdomain or domain needs CORS configured correctly on the API side.

Per the spec: "CORS is an HTTP-header based mechanism that allows a server to indicate any origins other than its own from which a browser should permit loading resources."

<a name="same-origin-policy"></a>
## 2. The Same-Origin Policy: the foundation

SOP is the browser's core security model: a page can only freely interact with resources from its own origin.

### What SOP blocks vs. allows
SOP blocks scripts from **reading** cross-origin response content. But several cross-origin *embeddings* are allowed without any CORS headers at all — they've been legal since before CORS existed:
- `<img src="...">` — cross-origin images render fine (you just can't read their pixel data into canvas without CORS).
- `<script src="...">` — cross-origin scripts execute, but you can't inspect their source or get detailed error info without CORS.
- `<link rel="stylesheet" href="...">` — cross-origin stylesheets apply.
- `<iframe>` — cross-origin iframes render (with their own separate SOP boundary for anything inside).

### SOP in action
If a page at `https://domain-a.com` calls `fetch("https://domain-b.com/data.json")` and `domain-b.com` sends no CORS headers, the request still goes out and the server still processes it — but the browser refuses to hand the response body back to the calling JavaScript. You'll see a network entry with a `200` status in devtools' Network tab, but the `fetch()` promise rejects with a CORS error.

<a name="what-triggers-cors"></a>
## 3. What triggers CORS (and what doesn't)

Not every cross-origin interaction goes through CORS at all.

**Always subject to CORS** (browser enforces the allow-origin check before letting JS read the result):
- `fetch()` / `XMLHttpRequest` reading a cross-origin response.
- Web fonts (`@font-face`) loaded cross-origin — servers must explicitly allow this.
- WebGL textures sourced cross-origin.
- Drawing cross-origin images/video frames to a `<canvas>` via `drawImage()` (needed to read pixel data back out).
- CSS Shapes computed from an external image.

**Not subject to CORS** (no browser-side JS is trying to read the cross-origin response content):
- `<img>`, `<video>`, `<audio>` used for plain display — rendering isn't "reading," so SOP doesn't block it.
- `<form>` submissions — these predate CORS/Fetch; the browser navigates or POSTs, but the page's own JS was never going to read the response body directly anyway.
- Simple navigation (links, redirects) — again, no script reading the response.
- WebSockets — they use their own origin-checking mechanism (the `Origin` header is sent, and it's up to the server to check it), not the CORS/preflight system.

| Request type | Triggers CORS? | Why |
|---|---|---|
| `fetch`/XHR reading data | Yes | SOP blocks reading a cross-origin response by default |
| `<img src="cross-origin">` for display | No | Display-only; yes if you then try to read pixels via canvas |
| `@font-face` | Yes | Fonts must be explicitly allowed cross-origin |
| CSS `background-image` | No | Display-only |
| WebSocket | No | Separate origin-check protocol, not HTTP CORS |
| `<form>` POST | No (for the response) | Predates CORS; response isn't readable by page JS anyway |

<a name="how-cors-works"></a>
## 4. How CORS works: the request/response flow

CORS is negotiated purely through HTTP headers, on a per-request basis.

### High-level flow
1. The browser attaches an `Origin` header to the outgoing cross-origin request (e.g., `Origin: https://domain-a.com`) — the browser sets this automatically; JS cannot override it.
2. The server's response includes (or omits) `Access-Control-Allow-Origin` and related headers.
3. If the response's `Access-Control-Allow-Origin` matches the request's `Origin` (or is `*`, when credentials aren't involved), the browser lets JS read the response. Otherwise, the promise/callback gets a CORS error and the body is inaccessible.

For requests that could have side effects the server hasn't explicitly cleared (custom headers, non-simple methods, non-simple content types), the browser first sends an automatic `OPTIONS` **preflight** request to ask permission, before ever sending the real one.

```
Simple request:
  Client --Origin header--> Server
  Server --Access-Control-Allow-Origin--> Client (JS can read if it matches)

Preflighted request:
  Client --OPTIONS + Access-Control-Request-Method/-Headers--> Server
  Server --Access-Control-Allow-Methods/-Headers--> Client
  (if approved) Client --actual request--> Server --response--> Client
```

"When the browser makes a request it adds an `Origin` header. If it's a mismatch, the browser prevents the response data from being shared with the page's JavaScript" — this is the one-sentence mental model to keep: **the request happens regardless; CORS decides whether the response is readable.**

<a name="simple-requests"></a>
## 5. Simple requests

Simple requests skip preflight because the spec considers them "safe" in the sense that they're indistinguishable from what an HTML `<form>` could always do pre-CORS — so gating them behind an extra permission check wouldn't add real protection.

### Conditions for a "simple" request — all must hold
- **Method**: `GET`, `HEAD`, or `POST` only.
- **Headers**: only the safelisted ones — `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` (restricted, see below), `Range` (with some restriction).
- **`Content-Type`**: only `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`. (Notably: `application/json` is **not** simple — sending JSON with `Content-Type: application/json` always triggers preflight.)
- No event listeners registered on `XMLHttpRequest.upload`.
- No `ReadableStream` used in the request body.

Safari/WebKit adds an extra restriction: it rejects "nonstandard" `Accept` header values that other browsers tolerate as simple.

⚠️ **Gotcha**: "simple" does not mean "safe" in the security sense — a simple `POST` can still change server-side state (this is exactly what an HTML form could always do). CORS's simple/preflight split is about *browser compatibility with pre-existing behavior*, not about protecting against state changes. Don't assume a simple request is somehow less dangerous; your server still needs CSRF protection independent of CORS (see the Security section).

### Example
```js
// JavaScript on https://foo.example
fetch("https://bar.other/data.json")
  .then(response => response.json())
  .then(data => console.log(data));
```

Request (browser sets `Origin` automatically):
```http
GET /data.json HTTP/1.1
Host: bar.other
Origin: https://foo.example
```

Response (server must include this for the browser to expose the body to JS):
```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://foo.example
Content-Type: application/json
```

If `Access-Control-Allow-Origin` doesn't match (or is missing), the browser still received a `200` — but blocks the JS from reading it.

<a name="preflighted-requests"></a>
## 6. Preflighted requests

For requests the browser can't assume are pre-CORS-safe — non-simple methods, custom headers, JSON bodies — the browser first sends an `OPTIONS` request asking "is this actually okay?" before sending the real request.

### What triggers a preflight
- Any method other than `GET`, `HEAD`, `POST` (so: `PUT`, `DELETE`, `PATCH`, etc.).
- Any header outside the safelist (a custom `X-Auth-Token`, for instance).
- `Content-Type` outside the three simple values — most notably, **`application/json` always preflights**.
- Credentialed requests can add additional preflight requirements depending on other headers involved.

### Preflight flow
1. Browser sends `OPTIONS` with:
   - `Access-Control-Request-Method`: the method the real request will use (e.g., `POST`)
   - `Access-Control-Request-Headers`: the custom headers the real request will send
2. Server responds (typically with no body, `204`) with:
   - `Access-Control-Allow-Methods`: the methods it permits
   - `Access-Control-Allow-Headers`: the headers it permits
   - `Access-Control-Max-Age`: how long the browser may cache this preflight result (seconds)
   - `Access-Control-Allow-Origin`: the origin(s) allowed
3. If the browser is satisfied the real request is covered, it sends the actual request — this is a *separate* HTTP request/response pair, with its own `Access-Control-Allow-Origin` check on the real response too.

```js
fetch("https://bar.other/doc", {
  method: "POST",
  headers: { "Content-Type": "application/xml", "X-PINGOTHER": "pingpong" },
  body: "<data>Stuff</data>"
});
```

Preflight request (automatic, browser-generated):
```http
OPTIONS /doc HTTP/1.1
Host: bar.other
Origin: https://foo.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type, x-pingother
```

Preflight response:
```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://foo.example
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: X-PINGOTHER, Content-Type
Access-Control-Max-Age: 86400
```

Then, and only then, the actual `POST /doc` request follows.

"Certain HTTP requests like PUT... need to be preflighted... the browser automatically knows when to preflight" — you never write preflight logic client-side; it's entirely browser-driven based on the request shape. Server-side, you must handle `OPTIONS` explicitly (most frameworks/middleware do this for you).

⚠️ **Gotcha**: preflight responses are cached per the `Access-Control-Max-Age` header, but browsers cap this differently — Chrome caps at 2 hours regardless of a larger value; if the header is absent, the default cache window is very short (as little as 5 seconds in some browsers), so identical requests in quick succession can re-trigger preflight repeatedly if you don't set `Max-Age`.

<a name="credentials"></a>
## 7. Credentials (cookies and auth) — and the wildcard trap

By default, cross-origin requests **do not** send cookies or HTTP auth headers, and the browser won't expose `Set-Cookie` from the response either. To opt in:

- **Client**: `fetch(url, { credentials: "include" })` or `xhr.withCredentials = true`.
- **Server**: respond with `Access-Control-Allow-Credentials: true`.

```js
fetch("https://bar.other/credentialed", { credentials: "include" });
```

```http
GET /credentialed HTTP/1.1
Host: bar.other
Origin: https://foo.example
Cookie: session=abc123
```

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://foo.example
Access-Control-Allow-Credentials: true
Set-Cookie: new=def456
```

If `Access-Control-Allow-Credentials` is missing (or `false`), the browser discards the response even if `Allow-Origin` matched — the JS never sees it.

### The wildcard + credentials trap — read this twice

**`Access-Control-Allow-Origin: *` combined with `Access-Control-Allow-Credentials: true` is invalid, and browsers reject it.** This is not a soft warning — modern browsers will refuse to expose the response to JS if the server sends this combination, precisely because it would be catastrophic if it worked: `*` means "any site on the internet," and combined with "also send the user's cookies," that's an open door for any malicious page to make an authenticated request on the victim's behalf and read the response.

```http
# INVALID — browser will block this even though both headers are individually well-formed
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

**The fix**: when credentials are involved, the server must echo back the *specific* requesting origin (never `*`), typically read dynamically from the incoming `Origin` header and validated against an allowlist:

```javascript
// Correct pattern: reflect a validated origin, never a bare wildcard, when credentials matter
const allowedOrigins = ['https://app.example.com', 'https://admin.example.com'];

app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (allowedOrigins.includes(origin)) {
    res.header('Access-Control-Allow-Origin', origin);
    res.header('Access-Control-Allow-Credentials', 'true');
    res.header('Vary', 'Origin'); // caches must not serve this response to a different Origin
  }
  next();
});
```

⚠️ Do **not** implement "validated origin" as "reflect whatever `Origin` header the request sent, no matter what." That defeats the purpose — it's functionally identical to a wildcard, just spelled differently, and is a real misconfiguration seen in production. Check the incoming origin against an actual allowlist before echoing it back.

### Preflight requests never carry credentials
Preflight (`OPTIONS`) requests never include cookies or auth headers, regardless of the `credentials` setting on the real request — this is by spec design, since the preflight is just a permission check, not the real call. The server's `OPTIONS` handler therefore can't (and shouldn't try to) authenticate the caller; it should just answer the CORS policy question. The *actual* request that follows is where credentials get sent and where your normal auth middleware applies.

"The most interesting capability... is the ability to make 'credentialed' requests that are aware of HTTP cookies" — but note some auth/security products have historically sent client certificates on preflights against spec; this was inconsistent across browsers and has mostly been resolved, but is a reminder that preflight behavior isn't always perfectly uniform across implementations.

<a name="redirects"></a>
## 8. Redirects in CORS

Redirects (3xx responses) interact awkwardly with preflighted requests specifically.

- **Historical issue**: older browsers refused to follow a redirect after a preflighted request at all, failing with an error instead.
- **Current spec**: following redirects post-preflight is now allowed, but browser support/behavior has varied, so don't assume it's universally smooth.

**Workarounds**:
- Avoid pairing a redirect with an endpoint that also requires preflight — if you control the server, resolve the redirect target directly instead of chaining through it.
- Make the request "simple" (no custom headers, simple content-type) so it skips preflight and redirect-following is less fraught.
- Do an initial simple `GET` to resolve the final URL client-side, then make the real (possibly preflighted) request directly to that URL.

Typical error you'll see: *"The request was redirected to https://example.com/foo, which is disallowed for cross-origin requests that require preflight."*

If an `Authorization` header on your request is what's forcing the preflight in the first place, you need server-side control over the redirect chain to fix this cleanly — client-side workarounds are limited.

<a name="third-party-cookies"></a>
## 9. Third-party cookies and CORS

CORS and cookie policy are two separate, independently-enforced browser mechanisms — getting CORS headers right doesn't automatically mean cookies will flow.

A cookie set by `domain-b.com` while your page is on `domain-a.com` is a **third-party cookie**, and modern browsers restrict these by default unless:
- The cookie is set with `SameSite=None; Secure`, explicitly opting into cross-site use, **and**
- The user (or browser default policy) hasn't blocked third-party cookies outright — and increasingly, browsers block them by default regardless (Safari's ITP, Chrome's phased third-party cookie deprecation).

Even with perfect `Access-Control-Allow-Origin` / `Access-Control-Allow-Credentials` configuration, if the browser's cookie policy blocks the cookie, credentials simply won't be attached to the request — and this failure mode looks different from a CORS error in devtools (the request may go out fine, just without the cookie), which is a common source of confusion when debugging "CORS is configured right but auth still isn't working."

**Analogy**: CORS opens the door for the response data; cookie policy is a separate lock on the cookie jar specifically — opening one doesn't open the other.

Per spec: "cookies set in CORS responses are subject to normal third-party cookie policies" — CORS headers don't override or bypass cookie-specific browser restrictions.

<a name="headers"></a>
## 10. HTTP headers reference

### Response headers (server sets these)

| Header | Syntax | Example | Purpose |
|---|---|---|---|
| `Access-Control-Allow-Origin` | `<origin> \| *` | `Access-Control-Allow-Origin: https://foo.example` | Which origin(s) may read the response. `*` only valid without credentials. Add `Vary: Origin` if the value is set dynamically per request. |
| `Access-Control-Expose-Headers` | `<header>[, <header>]*` | `Access-Control-Expose-Headers: X-Custom, Content-Range` | Allows JS to read non-standard response headers (by default JS can only read a small safelist: `Cache-Control`, `Content-Language`, `Content-Type`, `Expires`, `Last-Modified`, `Pragma`). |
| `Access-Control-Max-Age` | `<seconds>` | `Access-Control-Max-Age: 600` | How long the browser may cache a preflight result. Browser-imposed caps apply (e.g., Chrome: 2 hours max). |
| `Access-Control-Allow-Credentials` | `true` | `Access-Control-Allow-Credentials: true` | Permits credentials (cookies/auth) on the actual request. Cannot pair with `Allow-Origin: *`. |
| `Access-Control-Allow-Methods` | `<method>[, <method>]*` | `Access-Control-Allow-Methods: GET, POST, OPTIONS` | Methods allowed for the real request (returned on preflight). |
| `Access-Control-Allow-Headers` | `<header>[, <header>]*` | `Access-Control-Allow-Headers: X-Token, Content-Type` | Custom headers allowed for the real request (returned on preflight). |

With credentials involved, `Allow-Origin` must be an exact origin — never `*` — and the same applies in practice to being careful with `Allow-Headers`/`Allow-Methods` wildcards, since sloppy over-permissive configuration here is the most common real-world CORS security mistake.

### Request headers (browser sets these automatically — you never set these in JS)

| Header | Syntax | Example | When |
|---|---|---|---|
| `Origin` | `<origin>` | `Origin: https://foo.example` | Every cross-origin request |
| `Access-Control-Request-Method` | `<method>` | `Access-Control-Request-Method: DELETE` | Preflight only |
| `Access-Control-Request-Headers` | `<header>[, <header>]*` | `Access-Control-Request-Headers: Authorization` | Preflight only |

<a name="examples"></a>
## 11. Examples and code

### Simple GET
```js
fetch('https://api.example.com/data')
  .then(res => res.json())
  .then(console.log)
  .catch(err => console.error('CORS error:', err));
```
```js
// Server (Express)
app.get('/data', (req, res) => {
  res.header('Access-Control-Allow-Origin', '*');
  res.json({ message: 'Hello' });
});
```

### Preflighted POST with a custom header
```js
fetch('https://api.example.com/update', {
  method: 'POST',
  headers: { 'X-Auth': 'secret', 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'Bob' })
});
```
```js
// Server must handle OPTIONS explicitly
app.options('/update', (req, res) => {
  res.header('Access-Control-Allow-Origin', 'https://client.example.com');
  res.header('Access-Control-Allow-Methods', 'POST');
  res.header('Access-Control-Allow-Headers', 'X-Auth, Content-Type');
  res.sendStatus(204);
});

app.post('/update', (req, res) => {
  res.header('Access-Control-Allow-Origin', 'https://client.example.com');
  res.json({ success: true });
});
```

### With credentials
```js
fetch('https://api.example.com/protected', { credentials: 'include' });
```
```js
app.get('/protected', (req, res) => {
  res.header('Access-Control-Allow-Origin', 'https://client.example.com'); // never '*' here
  res.header('Access-Control-Allow-Credentials', 'true');
  res.json({ data: 'Secret' });
});
```

<a name="errors"></a>
## 12. Debugging a CORS error: step by step

CORS errors are deliberately vague in JavaScript — you get something like `TypeError: Failed to fetch` or a console message about a missing header, with no further detail exposed to script (this is intentional, to stop attackers from probing server config via error messages). All the real diagnostic information is in the Network tab, not the JS exception.

**Walkthrough:**

1. **Open DevTools → Network tab, reproduce the request.** Click the failed request. Note it likely shows a status (`200`, `404`, etc.) even though the fetch failed — that already tells you the request *reached* the server; the browser is blocking the *response*, not the request.

2. **Check whether a preflight (`OPTIONS`) request happened.** If your request has a custom header, non-simple `Content-Type` (like `application/json`), or a non-simple method, look for a separate `OPTIONS` entry immediately before the real request. If the `OPTIONS` request itself failed or is missing expected headers, that's your problem — the browser never even attempted the real request.

3. **Read the exact console error text** — it tells you which check failed:
   - *"No 'Access-Control-Allow-Origin' header is present"* → server isn't sending the header at all for this origin. Fix server config.
   - *"...Allow-Origin' header has a value 'https://x.com' that is not equal to the supplied origin"* → server is sending a hardcoded/wrong origin instead of reflecting/allowlisting the caller's actual origin.
   - *"Method ... is not allowed"* → check the preflight response's `Access-Control-Allow-Methods`.
   - *"Request header field X-Custom is not allowed"* → add it to `Access-Control-Allow-Headers` on the server.
   - *"...has been blocked by CORS policy: The value of the 'Access-Control-Allow-Origin' header in the response must not be the wildcard '*' when the request's credentials mode is 'include'"* → the wildcard+credentials trap above. Fix: send back the specific validated origin, not `*`.

4. **Inspect the actual response headers on the real (non-OPTIONS) request too** — a correct preflight doesn't guarantee the real response also has correct headers; some setups pass preflight but the actual route handler forgets to set `Access-Control-Allow-Origin`.

5. **Rule out cookie policy as a separate cause.** If headers all look right but an authenticated request still behaves as logged-out, check whether it's actually a third-party cookie being blocked (see section 9) rather than a CORS header problem — these look similar but have different fixes.

6. **Test with `curl` or Postman to isolate browser vs. server.** These tools ignore CORS entirely, so if the request/response works there, the server logic itself is fine — the issue is purely in the CORS headers the server is (or isn't) sending, not in the underlying endpoint logic.
   ```bash
   curl -i -X OPTIONS https://api.example.com/update \
     -H "Origin: https://app.example.com" \
     -H "Access-Control-Request-Method: POST" \
     -H "Access-Control-Request-Headers: content-type"
   ```
   Inspect the response headers directly — this shows you exactly what the server sends for a preflight, without any browser interpretation in the way.

7. **Common root causes, roughly in order of frequency**:
   - Server doesn't set `Access-Control-Allow-Origin` for this origin at all (misconfigured or forgot the middleware).
   - Server hardcodes one origin, but the app is calling from a different one (e.g., `localhost:3000` in dev vs. the configured production origin).
   - `Access-Control-Allow-Headers` missing a custom header the client sends.
   - Wildcard + credentials conflict.
   - `OPTIONS` handler not implemented at all (server returns 404/405 for the preflight itself).
   - CDN/proxy in front of the server stripping or not forwarding CORS headers.

**Analogy**: a CORS error is a bouncer saying "no entry" without explaining why — the guest list (the actual headers) is in the Network tab, not in the vague message you get in the console.

<a name="server-config"></a>
## 13. Server-side configuration by framework

### Express.js (Node)
```bash
npm i cors
```
```javascript
const cors = require('cors');
app.use(cors({
  origin: 'https://client.example.com',  // string, array, or validator function
  methods: ['GET', 'POST'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 600
}));
```
For dynamic allowlisting: `origin: (origin, cb) => cb(null, allowedOrigins.includes(origin))`.

### Apache (.htaccess)
```apache
Header set Access-Control-Allow-Origin "*"
```

### Nginx
```nginx
add_header 'Access-Control-Allow-Origin' '*';
```

### Python (Flask)
```python
from flask_cors import CORS
CORS(app, origins='https://client.example.com')
```

### Go
```go
func handler(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Access-Control-Allow-Origin", "*")
}
```

Manual header-setting always works as a fallback, but middleware handles preflight `OPTIONS` responses and edge cases (like the `Vary: Origin` header) correctly with far less code — "in Express, this can be achieved with a single line of middleware."

<a name="compatibility"></a>
## 14. Browser compatibility and edge cases

CORS is universally supported in modern browsers (Chrome 4+, Firefox 3.5+, Safari 4+, Edge — all current). Legacy IE had partial support (XHR only, from IE10).

**Edge cases**:
- Safari/WebKit blacklists certain "nonstandard" `Accept` header values from qualifying as simple requests.
- Web Workers inherit the origin of the page that spawned them.
- Service Workers can intercept fetch events, but CORS rules still apply to what they fetch.
- Non-browser tools (`curl`, Postman, server-to-server calls) don't enforce CORS at all — it is purely a browser-side protection, never a server-side one. A server with no CORS headers is just as reachable by a script running outside a browser as one with permissive headers.

<a name="security"></a>
## 15. Security implications and best practices

### Risks
- **Overly permissive `Access-Control-Allow-Origin: *`** (or worse, "reflect whatever `Origin` was sent") on an endpoint that also allows credentials, or that returns sensitive data, effectively exposes that data to any site on the internet that can trick a user's browser into making the request.
- **CORS does not prevent CSRF.** These are separate concerns: CORS controls whether a cross-origin script can *read* a response; it does nothing to stop a cross-origin *form* or simple request from being *sent* and having side effects (a simple `POST` doesn't need CORS approval to fire — see section 5). Use CSRF tokens or `SameSite` cookies for that; don't rely on CORS as CSRF protection.
- **Over-permissive method/header allowlists** widen the attack surface unnecessarily — allowing every method and header "just in case" defeats the purpose of the preflight check.

### Best practices
1. Use specific origins, never a bare `*`, especially once credentials or sensitive data are involved.
2. Validate the incoming `Origin` against a real allowlist before reflecting it back — never blindly echo it.
3. Send `Vary: Origin` whenever the `Allow-Origin` value is set dynamically, so caches (CDNs, browser cache) don't serve one origin's CORS response to a different origin.
4. Limit `Access-Control-Allow-Methods` and `-Headers` to exactly what's needed.
5. Serve everything over HTTPS — mixing CORS with plaintext HTTP adds an interception risk on top of the CORS config itself.
6. Implement the `OPTIONS` preflight handler correctly and test it directly (see debugging section).
7. Test across multiple browsers — Safari's stricter simple-request rules and third-party cookie defaults can behave differently.
8. Document the exact CORS requirements for any public API — consumers need to know which origins, methods, and headers are supported.
9. Avoid enabling `credentials: true` unless the endpoint genuinely needs cookies/auth headers cross-origin — it's an easy thing to leave on by default and forget.
10. Monitor logs for repeated CORS-rejected requests — it can reveal both misconfiguration and probing/attack attempts.

Per spec: "CORS failures result in errors, but for security reasons, specifics about the error are not available to JavaScript" — this is deliberate, not a bug in your tooling.

<a name="advanced"></a>
## 16. Advanced topics

- **CORS and WebSockets**: WebSockets don't use the CORS/preflight system at all — they have their own origin-checking convention (server reads the `Origin` header on the upgrade request and decides whether to accept the connection).
- **CORS proxies**: a workaround (e.g., `cors-anywhere`) for consuming an API you don't control and that lacks CORS headers — routes the request through an intermediary that adds the headers. Fine for prototyping, a security and reliability liability in production (you're trusting a third party with the traffic).
- **JSONP**: the pre-CORS workaround (loading data via a `<script>` tag, since scripts aren't SOP-restricted) — inherently insecure (arbitrary script execution from the response) and obsolete now that CORS exists.
- **Server-Sent Events (SSE)**: uses the same CORS rules as Fetch.
- **Fetch `mode` options**: `'cors'` (default cross-origin behavior, subject to everything above), `'no-cors'` (allows the request to fire but yields an **opaque** response — status and body unreadable — useful only for fire-and-forget side effects), `'same-origin'` (fails outright if the URL isn't same-origin).
- **CORS in microservices**: each service behind a gateway needs its own correct CORS config (or the gateway centralizes it) — a common mistake is configuring CORS only on the "main" API and forgetting a newly added microservice.
- **Performance**: preflight requests add a full extra round trip before the real request can even start; set `Access-Control-Max-Age` generously (bounded by browser caps) to avoid re-preflighting identical requests repeatedly.
- **Alternatives predating/adjacent to CORS**: `document.domain` for same-parent-domain subdomain sharing (deprecated, being removed from browsers), `postMessage` for controlled cross-origin communication between windows/iframes (a different mechanism entirely, not HTTP-header-based).

"Although the POST method can modify server data, it does not always trigger a preflight request" — worth re-emphasizing since it directly undercuts the intuition that "preflight = the browser protecting me from dangerous requests." It doesn't; see the CSRF note above.

<a name="faqs"></a>
## 17. FAQs and myths

**Q: Why doesn't JS get error details on a CORS failure?**
A: Deliberate — detailed errors would let an attacker's script probe server configuration (which origins are allowed, etc.) by reading failure messages.

**Q: Does CORS run on the server?**
A: No — the server just sends headers describing its policy. Enforcement (blocking the response from JS) happens entirely in the browser. The server has no way to "block" the request from arriving; it can only refuse to answer or answer without CORS headers, at which point the browser (not the server) prevents the frontend code from reading the response.

**Q: Can I disable CORS in my browser?**
A: Yes, via flags (`--disable-web-security` in Chrome) — strictly a local development workaround, never something to tell end users to do, and never a substitute for fixing the server.

**Myth: "CORS protects my API from attacks."**
Truth: CORS controls which *browser-based JavaScript* can read responses. It does nothing against non-browser clients, and doesn't stop the request from being sent in the first place for simple requests. Pair it with real server-side auth and CSRF protection.

**Q: What if `Origin` is `null`?**
A: Happens for `file://` URLs, sandboxed iframes, and some redirect chains. Servers can choose to allow it, but doing so broadly is risky since many different untrusted contexts can present `Origin: null`.

<a name="resources"></a>
## 18. Further reading
- MDN: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- Fetch spec: https://fetch.spec.whatwg.org/
- CORS explainer: https://willitcors.com/
- Tools: Chrome DevTools Network tab, Postman/curl for isolating browser vs. server behavior, `https://test-cors.org/` for live testing
