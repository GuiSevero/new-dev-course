# Module 06 — Client–Server Communication & HTTP

**Time:** ~6 hours
**Prereq:** [Module 01](01-how-computers-run-code.md), [Module 02](02-programming-fundamentals.md)

---

## Why this module exists

Everything you build from here on is two programs talking to each other over a network they don't control. HTTP is the language they speak. Most "weird" bugs in web development — CORS errors, cookies that vanish, caches serving stale data, requests that arrive twice — are HTTP behaving exactly as specified, in a way you didn't know about.

This module is heavy on *looking at real traffic*. Do the labs; reading about HTTP is much less useful than watching it.

---

## 6.1 The client–server model

- A **server** is a program that starts, binds to a port, and waits. It doesn't initiate anything.
- A **client** is a program that initiates a request and waits for a response.
- "Client" and "server" are **roles, not machines.** Your API is a server to the browser and a client to Postgres.

**Key properties of the web's version of this model:**

- **Request/response.** One request, one response. The server cannot spontaneously push you something over plain HTTP. (That's what WebSockets and Server-Sent Events are for — [MDN: WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API), [SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events).)
- **Stateless.** The server remembers nothing between requests. Every request must carry everything needed to handle it. This is *why* cookies and tokens exist — they re-establish identity on each request.
- **The network is unreliable and slow.** Requests get lost, arrive twice, arrive out of order, or hang. Latency between continents is ~150ms *at the speed of light*, before any processing. Design for this: timeouts, retries, idempotency.

---

## 6.2 What happens beneath HTTP

Layers again (Module 01).

**DNS** — `taskflow.app` → `203.0.113.42`. Your machine asks a resolver, which asks the root servers, then `.app`'s servers, then the domain's own nameservers. Results are cached at every level, which is why DNS changes "take time to propagate."

```bash
dig taskflow.app +short
nslookup github.com
```

**TCP** — a reliable, ordered byte stream on top of unreliable packets. Established with a three-way handshake (SYN → SYN-ACK → ACK), which costs one round-trip *before any data moves*. TCP handles retransmitting lost packets and reassembling out-of-order ones.

**TLS** — encryption and identity. Another 1–2 round trips. The server presents a certificate signed by a Certificate Authority your OS trusts, proving it really is `taskflow.app`. Without TLS, anyone on the path (your café's wifi, an ISP) can read and modify traffic. **Use HTTPS everywhere; [Let's Encrypt](https://letsencrypt.org/) made it free.**

**HTTP** — finally, the request.

**Why you care:** each connection costs several round trips before the first byte of your data. That's the entire reason for HTTP keep-alive, connection pooling, HTTP/2 multiplexing, and CDNs. When someone says "reduce the number of requests," this is why.

📖 [High Performance Browser Networking](https://hpbn.co/) ch. 1–4 — the definitive free reference
🎨 [How HTTPS works](https://howhttps.works/) — the comic version
🎮 [Cloudflare Learning Center](https://www.cloudflare.com/learning/) — short, accurate explainers for every term in this module

---

## 6.3 Anatomy of an HTTP message

An HTTP request is **plain text** (in HTTP/1.1 — HTTP/2 and 3 are binary, but semantically identical).

```http
POST /api/tasks HTTP/1.1
Host: api.taskflow.app
Content-Type: application/json
Accept: application/json
Authorization: Bearer eyJhbGciOi...
Cookie: session=abc123
Content-Length: 46

{"title":"Learn HTTP","listId":"7f3a-..."}
```

Four parts: **request line** (method, path, version), **headers** (key: value metadata), **blank line**, **body** (optional).

The response has the same shape:

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/tasks/9c2f-...
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/

{"id":"9c2f-...","title":"Learn HTTP","status":"todo"}
```

**See it for real** — this is the single most valuable exercise in the module:

```bash
curl -v https://api.github.com/zen
```

`-v` prints the whole conversation: DNS, TLS handshake, request headers (`>`), response headers (`<`), body. Read every line and look up anything you don't recognize.

📖 [MDN: HTTP Messages](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Messages) · [MDN: HTTP Headers reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers)

---

## 6.4 The verbs

The method tells the server what *kind* of operation this is. Two properties matter enormously:

- **Safe** — doesn't change anything. Can be prefetched, retried, crawled freely.
- **Idempotent** — doing it twice has the same effect as doing it once. Safe to retry after a timeout.

| Method | Purpose | Safe | Idempotent | Body |
|---|---|:---:|:---:|:---:|
| **GET** | Retrieve a resource | ✅ | ✅ | no |
| **POST** | Create a resource / non-idempotent action | ❌ | ❌ | yes |
| **PUT** | Replace a resource entirely | ❌ | ✅ | yes |
| **PATCH** | Partially update a resource | ❌ | ❌* | yes |
| **DELETE** | Remove a resource | ❌ | ✅ | no |
| **HEAD** | Like GET, headers only, no body | ✅ | ✅ | no |
| **OPTIONS** | What can I do with this resource? Used by CORS preflight | ✅ | ✅ | no |

\* PATCH *can* be idempotent depending on the patch format; `{"status":"done"}` is, `{"increment": 1}` isn't.

**Why idempotency is not academic.** A user taps "Create task." The response is lost to a flaky mobile network. The client retries. With `POST`, you now have two tasks. With `PUT /tasks/<client-generated-id>`, you have one. Real systems solve this with client-generated UUIDs or an `Idempotency-Key` header — [Stripe's design](https://docs.stripe.com/api/idempotent_requests) is the canonical example.

**`HEAD`** is genuinely useful: check whether a large file exists or has changed (via `ETag`/`Last-Modified`) without downloading it.

```bash
curl -I https://developer.mozilla.org      # -I sends HEAD
```

**PUT vs PATCH.** `PUT /tasks/1` with `{"title":"x"}` means *the task is now exactly this* — omitted fields get wiped. `PATCH` means *change only what I sent*. Getting this wrong is a classic data-loss bug.

📖 [MDN: HTTP request methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods)

---

## 6.5 Status codes

The first digit is the category. Learn the categories, then ~15 specific codes.

| Range | Meaning | Who's at fault |
|---|---|---|
| **1xx** | Informational | — |
| **2xx** | Success | — |
| **3xx** | Redirection | — |
| **4xx** | Client error | The caller |
| **5xx** | Server error | You |

The ones you'll use:

- **200 OK** — success with a body
- **201 Created** — a resource was created (include a `Location` header)
- **204 No Content** — success, nothing to return (typical for DELETE)
- **301 / 308** — moved permanently (browsers cache this *hard*; be careful)
- **302 / 307** — moved temporarily
- **304 Not Modified** — your cached copy is still good; here's no body, save the bandwidth
- **400 Bad Request** — malformed or failed validation
- **401 Unauthorized** — you are not authenticated (misnamed; it means "unauthenticated")
- **403 Forbidden** — we know who you are; you're not allowed
- **404 Not Found** — no such resource
- **409 Conflict** — collides with current state (duplicate email, edit conflict)
- **422 Unprocessable Content** — syntactically fine, semantically invalid
- **429 Too Many Requests** — rate limited (send a `Retry-After` header)
- **500 Internal Server Error** — you crashed
- **502 / 503 / 504** — bad gateway / unavailable / gateway timeout — usually infrastructure

**401 vs 403** is the distinction beginners always miss: *who are you?* vs *you can't do that*. Module 09 formalizes it as authentication vs authorization.

📖 [MDN: HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status) · 😺 [http.cat](https://http.cat/) — surprisingly effective mnemonics

---

## 6.6 Headers you must know

| Header | Direction | What it does |
|---|---|---|
| `Content-Type` | both | Format of the body: `application/json`, `text/html`, `multipart/form-data` |
| `Accept` | request | What formats the client can handle |
| `Authorization` | request | Credentials, e.g. `Bearer <token>` |
| `Cookie` / `Set-Cookie` | req / res | See below |
| `Cache-Control` | both | How long, and by whom, this may be cached |
| `ETag` / `If-None-Match` | res / req | Version fingerprint → enables 304 responses |
| `Location` | response | Where the created/moved resource is |
| `User-Agent` | request | What client this is |
| `Access-Control-Allow-Origin` | response | CORS — see below |
| `Retry-After` | response | How long to wait before retrying (with 429/503) |

### CORS — the error you will absolutely hit

The browser enforces the **same-origin policy**: JavaScript on `https://app.taskflow.dev` may not read responses from `https://api.taskflow.dev` unless that server explicitly opts in. An *origin* is scheme + host + port — all three must match.

This exists so that a malicious page can't quietly read your bank's API using your logged-in cookies.

**CORS** (Cross-Origin Resource Sharing) is the opt-in mechanism: the server sends `Access-Control-Allow-Origin` headers saying who may read it. For "non-simple" requests (custom headers, methods beyond GET/POST/HEAD), the browser first sends a **preflight** `OPTIONS` request asking permission, *then* the real one.

Four things to internalize now, so future-you doesn't lose an afternoon:

1. **CORS is enforced by the browser, not the server.** `curl` ignores it entirely. If curl works and the browser doesn't, it's CORS.
2. **A CORS error is not a server error.** The request often reached your server and succeeded; the browser refused to hand the response to your JavaScript.
3. **Cookies across origins need `credentials: "include"` on the client *and* `Access-Control-Allow-Credentials: true` plus a specific (not `*`) origin on the server.** Module 09 depends on this.
4. Fixing CORS means configuring the **server**, never the client.

📖 [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS) — read it properly once, and you'll never fear the error again.

### Caching

```http
Cache-Control: public, max-age=31536000, immutable   # a hashed JS bundle: cache forever
Cache-Control: no-store                              # never cache: anything personal
Cache-Control: private, max-age=0, must-revalidate   # check with the server every time
```

`ETag` lets the server say "same as before" with a **304** and no body — big savings on unchanged data.

> Caching is famously one of the [two hard problems](https://martinfowler.com/bliki/TwoHardThings.html) in computer science. The practical rule: cache aggressively for content-addressed static assets (filenames containing a hash), and `no-store` for anything user-specific.

📖 [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching)

---

## 6.7 Cookies

A cookie is a small key-value pair the server asks the browser to store and send back on every subsequent request to that domain. It is HTTP's answer to statelessness.

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800
```

Every attribute matters, and each one prevents a specific attack:

| Attribute | Effect | Why |
|---|---|---|
| `HttpOnly` | JavaScript **cannot** read this cookie | An XSS bug can no longer steal the session |
| `Secure` | Sent only over HTTPS | Can't be sniffed on plain HTTP |
| `SameSite=Lax` | Not sent on most cross-site requests | Blocks most CSRF attacks |
| `SameSite=Strict` | Never sent cross-site | Safest; breaks "click a link from email and be logged in" |
| `SameSite=None` | Always sent — **requires `Secure`** | Needed for genuine cross-site APIs |
| `Path` / `Domain` | Scope of the cookie | Least privilege |
| `Max-Age` / `Expires` | Lifetime; without one it dies when the browser closes | Limits the damage window |

**Cookies vs `localStorage` for auth tokens** — the argument you'll see everywhere:

- `localStorage` is readable by any JavaScript on your page. One XSS vulnerability, or one compromised npm dependency, and your tokens are exfiltrated.
- An `HttpOnly` cookie is not readable by JavaScript at all. It is automatically sent by the browser (which is why you then need CSRF protection — hence `SameSite`).

**Use `HttpOnly` cookies for session tokens.** This is the current consensus and it's what Better Auth does by default in Module 09.

📖 [MDN: Using HTTP cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies) · [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)

---

## 6.8 REST API design

REST is an architectural style [described by Roy Fielding in 2000](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm). In practice, "REST API" means: resources identified by URLs, manipulated with HTTP verbs, exchanging JSON.

**Resources are nouns. Verbs are HTTP methods.**

```
GET    /tasks              list tasks
POST   /tasks              create a task
GET    /tasks/:id          fetch one
PATCH  /tasks/:id          partially update
DELETE /tasks/:id          delete
GET    /lists/:id/tasks    tasks belonging to a list  (nested resource)
```

```
❌ POST /createTask        the verb is in the URL — that's RPC, not REST
❌ GET  /tasks/delete/1    a GET that mutates. Prefetchers and crawlers will ruin your day.
❌ POST /getTasks
```

**Design conventions worth adopting:**

- Plural nouns for collections (`/tasks`, not `/task`)
- Filtering, sorting, pagination via query string: `GET /tasks?status=done&sort=-createdAt&limit=20&cursor=abc`
- Return the created resource from `POST`, with a `Location` header and `201`
- Nest at most one level deep; beyond that, use query params
- **Version your API** (`/api/v1/...`) before you need to, not after
- Errors should be consistent, machine-readable, and never leak internals:

```json
{ "error": { "code": "VALIDATION_FAILED",
             "message": "Title is required",
             "details": [{ "field": "title", "issue": "required" }] } }
```
  Consider the [RFC 9457 Problem Details](https://datatracker.ietf.org/doc/html/rfc9457) format so you don't have to invent one.

**Pagination.** Never return an unbounded list. Offset pagination (`?page=3`) is simple but drifts when data changes underneath. Cursor pagination (`?cursor=<last-id>`) is stable and scales.

**Know the alternatives exist:** [GraphQL](https://graphql.org/) (client specifies exactly what it wants — good for many clients with different needs), [tRPC](https://trpc.io/) (typed RPC for TypeScript-everywhere apps), [gRPC](https://grpc.io/) (binary, fast, service-to-service). REST is the right default and the one you must know.

📖 [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html) — the ladder from "HTTP as transport" to true REST
📖 [Zalando's RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/) — a real company's real, detailed rulebook. Skim it to see what rigor looks like.
📖 [Microsoft's API design best practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design)

---

## 6.9 JSON, and the serialization trap

JSON has six types: string, number, boolean, null, array, object. **That's it.** No dates, no `undefined`, no `Map`, no `BigInt`, no comments.

```ts
JSON.stringify({ when: new Date(), missing: undefined, big: 10n });
// dates become ISO strings, undefined keys VANISH, BigInt throws
```

So `new Date()` goes out as `"2026-08-07T12:00:00.000Z"` and comes back as a **string**, not a Date. Your TypeScript type says `Date`; the runtime value is a string. (Module 01 warned you: types are erased.) This bites everyone exactly once. Handle it explicitly — parse at the boundary, with Zod (Module 08).

Also note: JSON numbers are IEEE 754 doubles, so integers above 2^53 lose precision. Database IDs that are 64-bit integers must be sent as **strings**. Another reason this course uses UUIDs.

---

## Lab 06 — Get your hands in the traffic

**A. Read a real conversation.**
```bash
curl -v https://api.github.com/users/torvalds
```
Write down: what did DNS resolve to? What TLS version? List five response headers and say what each does. What's the `Content-Type`?

**B. All the verbs.** Use [httpbin.org](https://httpbin.org/) (an endpoint that echoes your request back):
```bash
curl -X GET    "https://httpbin.org/get?status=done&limit=10"
curl -X POST   https://httpbin.org/post   -H "Content-Type: application/json" -d '{"title":"Learn HTTP"}'
curl -X PUT    https://httpbin.org/put    -H "Content-Type: application/json" -d '{"title":"Replaced"}'
curl -X PATCH  https://httpbin.org/patch  -H "Content-Type: application/json" -d '{"status":"done"}'
curl -X DELETE https://httpbin.org/delete
curl -I        https://httpbin.org/get
curl -v        https://httpbin.org/status/404
curl -v        https://httpbin.org/status/500
```
For each: what came back, and what did the server learn about your request?

**C. Cookies, observed.**
```bash
curl -c jar.txt https://httpbin.org/cookies/set/session/abc123
cat jar.txt
curl -b jar.txt https://httpbin.org/cookies
```
Now do the same thing in your browser's devtools: Application → Cookies. Find a site with an `HttpOnly` session cookie and try to read it with `document.cookie` in the console. Watch it not be there. That's the protection working.

**D. Cause a CORS error yourself.** Open any site (say `https://example.com`), open the console, and run:
```js
fetch("https://api.github.com/zen").then(r => r.text()).then(console.log)   // works — GitHub sends CORS headers
fetch("https://www.google.com").then(r => r.text()).then(console.log)       // blocked
```
Read the exact error text. Then check the Network tab: **did the request actually go out?** (Yes.) Explain to yourself why that matters.

**E. Design the TaskFlow API — on paper, before writing any code.**

Write `docs/api.md` specifying every endpoint you'll need: method, path, query params, request body, success status + body, and every error status. Include auth endpoints. Think about:
- Should completing a task be `PATCH /tasks/:id` with `{status:"done"}`, or `POST /tasks/:id/complete`? Argue both, then pick.
- How do you paginate `GET /tasks`?
- What happens on `GET /tasks/:id` for a task belonging to *another user* — 404 or 403? (Think about what each answer leaks.)

```text
Prompt for Claude Code:
Here's my REST API design for a multi-user todo app: [paste docs/api.md].

Critique it as an API reviewer would. Specifically:
1. Are my resources and verbs idiomatic REST?
2. Which status codes did I get wrong?
3. What did I forget entirely? (pagination, versioning, rate limiting,
   error format, idempotency, partial failure...)
4. Where does my design leak information a user shouldn't learn?

Don't rewrite it — give me a numbered list of issues and let me fix them.
```

---

## Understanding Gate

1. What makes a method *idempotent*, and why does it matter on a mobile network?
2. `PUT` vs `PATCH` — describe a data-loss bug caused by confusing them.
3. Your fetch fails with a CORS error. Did the request reach the server? How do you check? Where's the fix?
4. What does `HttpOnly` prevent, and what does it *not* prevent?
5. Why does `SameSite` exist? What attack does it stop?
6. 401 vs 403 vs 404 — when would you deliberately return 404 for something that exists?
7. You send `{ dueDate: new Date() }` from the browser. What type is `dueDate` when it arrives at the server? Why?
8. Why is HTTP being stateless the reason cookies exist?

---

**Next:** [Module 07 — Databases and SQL](07-databases-sql.md)
