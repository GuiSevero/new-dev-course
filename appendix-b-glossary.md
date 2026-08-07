# Appendix B — Glossary

Terms in the order you're likely to meet them, grouped by area. Use this when a word in a module or a Stack Overflow answer means nothing to you.

---

## Machine & runtime

**Process** — a running program, with its own memory and OS resources.
**Thread** — a stream of execution within a process. JavaScript's main work happens on one.
**Stack** — memory for function calls; fast, ordered, automatically reclaimed. Overflows if you recurse forever.
**Heap** — memory for objects; large, unordered, garbage-collected in JS.
**Garbage collection (GC)** — automatic freeing of unreachable heap memory.
**Memory leak** — memory you no longer need but the GC can't free because something still references it.
**Reference vs value** — objects are passed by reference (both names point at the same thing); primitives are copied.
**Runtime** — the program that executes your code (Node, Bun, a browser).
**V8** — Google's JavaScript engine. Powers Chrome and Node. (Bun uses a different engine, JavaScriptCore, from Safari.)
**JIT** — just-in-time compilation; interpret first, compile hot code to machine code as it runs.
**Transpile** — translate source to other source (TypeScript → JavaScript), not to machine code.
**Event loop** — the mechanism that runs queued callbacks when the call stack is empty. See Module 01.
**Blocking** — occupying the thread so nothing else can run. On a server, catastrophic.
**Microtask / macrotask** — promise callbacks run before timers. This is why `Promise.resolve().then()` beats `setTimeout(…, 0)`.
**PATH** — the list of directories your shell searches for commands.
**Environment variable** — a key-value string passed to a process at startup; how you configure software without editing it.
**stdout / stderr** — a process's two output streams (normal, and errors).
**Exit code** — the number a program returns. `0` = success.
**Port** — a number (0–65535) identifying a network endpoint on a machine.

---

## Language & types

**Type inference** — the compiler working out a type you didn't write.
**Narrowing** — TypeScript refining a union type based on your control flow.
**Union type** — `"todo" | "done"`; one of several possibilities.
**Generic** — a type parameterized by another type: `Array<Task>`, `Promise<T>`.
**Strict mode** — TypeScript's set of stricter checks. Always on.
**Type erasure** — types are deleted at compile time and do not exist at runtime.
**Closure** — a function plus the variables it captured from where it was defined.
**Pure function** — same input → same output, no side effects.
**Side effect** — anything a function does beyond returning a value: I/O, mutation, logging.
**Mutation** — changing an existing value in place, rather than producing a new one.
**Immutability** — never mutating; always producing new values.
**Promise** — an object representing a value that will exist later, or an error.
**async/await** — syntax for working with promises as if they were synchronous.
**Race condition** — behaviour that depends on the unpredictable ordering of concurrent operations.
**Idempotent** — doing it twice has the same effect as doing it once.
**ESM / CommonJS** — the modern (`import`) and legacy (`require`) module systems.
**Serialization** — converting a value to a transmittable format (usually JSON) and back.

---

## Complexity & data structures

**Big-O** — how work grows with input size. `O(n)`, `O(n²)`, `O(log n)`.
**Hash map** — key → value in `O(1)`. `Map` in JS.
**Set** — a collection with `O(1)` membership tests and no duplicates.
**B-tree** — the balanced tree structure behind database indexes; `O(log n)` lookups.
**N+1 query** — one query to get a list, then one more per item. The classic performance bug.
**Amortized** — average cost over many operations (array `push` is amortized `O(1)`).

---

## Git

**Repository (repo)** — a project plus its full history.
**Commit** — a snapshot plus metadata plus a parent pointer.
**Branch** — a movable pointer to a commit. Not a copy.
**HEAD** — a pointer to your current branch/commit.
**Staging area (index)** — where you assemble the next commit.
**Merge** — combine two branches, creating a commit with two parents.
**Rebase** — replay your commits onto a different base, rewriting them.
**Conflict** — two branches changed the same lines; Git asks you to decide.
**Remote** — another copy of the repo, usually on a server. `origin` by convention.
**Fetch vs pull** — download changes vs download-and-integrate.
**Pull request (PR)** — a request to merge a branch, plus a review conversation.
**Squash** — collapse several commits into one.
**Reflog** — Git's log of everywhere HEAD has been. Your undo-the-undo.
**Cherry-pick** — apply a single commit from elsewhere onto your branch.
**Bisect** — binary-search the history for the commit that introduced a bug.

---

## Web & HTTP

**Client / server** — roles, not machines. The initiator and the listener.
**Stateless** — the server remembers nothing between requests.
**DNS** — name → IP address.
**TCP** — reliable ordered byte stream over unreliable packets.
**TLS/SSL** — encryption + server identity. The S in HTTPS.
**Round trip (RTT)** — one there-and-back network journey. The unit of latency.
**Latency vs bandwidth** — delay vs throughput. Latency is usually your problem.
**Request/response** — one message out, one message back.
**Method / verb** — GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS.
**Safe method** — doesn't change anything (GET, HEAD).
**Status code** — 2xx success, 3xx redirect, 4xx your fault, 5xx my fault.
**Header** — key-value metadata on a request or response.
**Body / payload** — the actual content.
**Origin** — scheme + host + port. All three must match to be "same origin."
**Same-origin policy** — the browser rule preventing cross-origin reads by default.
**CORS** — the server's opt-in mechanism for allowing cross-origin reads.
**Preflight** — the `OPTIONS` request the browser sends before a non-simple cross-origin request.
**Cookie** — a small value the browser stores and re-sends on every request to that domain.
**HttpOnly** — a cookie flag making it unreadable by JavaScript.
**SameSite** — a cookie flag controlling whether it's sent on cross-site requests. CSRF defence.
**Cache-Control / ETag** — headers governing caching and revalidation.
**REST** — resources as URLs, manipulated with HTTP verbs.
**Endpoint** — one method + path combination.
**Idempotency key** — a client-supplied id so a retried request isn't processed twice.
**Pagination** — returning results in chunks. Offset-based or cursor-based.
**WebSocket / SSE** — protocols for server-initiated messages, which plain HTTP can't do.
**Rate limiting** — capping requests per client per time window.
**CDN** — geographically distributed caches serving static assets close to users.

---

## Databases

**Relational database** — data in tables with enforced relationships. Postgres, MySQL.
**Schema** — the structure: tables, columns, types, constraints.
**Row / record, column / field** — a single entity, and one of its attributes.
**Primary key** — uniquely identifies a row.
**Foreign key** — points at another table's primary key; the DB enforces it exists.
**Referential integrity** — the guarantee that foreign keys point at real rows.
**Constraint** — a rule the database enforces: `NOT NULL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`.
**Normalization** — storing each fact exactly once.
**Join** — combining rows from multiple tables. `INNER`, `LEFT`, `FULL`.
**Index** — a lookup structure making queries fast, at the cost of write speed and disk.
**Sequential scan** — reading every row. What an index avoids.
**EXPLAIN** — shows the plan the database will use for a query.
**Transaction** — a group of statements that all commit or all roll back.
**ACID** — Atomicity, Consistency, Isolation, Durability.
**Isolation level** — how much concurrent transactions can see of each other.
**Lost update** — two concurrent read-modify-writes; one is silently overwritten.
**Deadlock** — two transactions each waiting on a lock the other holds.
**Migration** — a versioned script that changes the schema.
**Expand/contract** — the safe multi-deploy pattern for schema changes.
**ORM** — maps database rows to language objects. Drizzle, Prisma.
**Connection pool** — a set of reused database connections.
**SQL injection** — untrusted input interpreted as SQL. Prevented by parameterized queries.
**CTE** — a named subquery (`WITH … AS`).
**Window function** — an aggregate that doesn't collapse rows.
**Upsert** — insert, or update if it already exists.

---

## Auth

**Authentication (authN)** — who are you? → 401.
**Authorization (authZ)** — are you allowed? → 403.
**Hash** — a one-way function. Passwords are stored hashed, never encrypted.
**Salt** — random per-password data mixed into the hash, defeating rainbow tables.
**Argon2 / bcrypt / scrypt** — deliberately slow password hashing functions.
**Session** — server-side record of a logged-in user, referenced by an opaque cookie.
**JWT** — a signed, self-contained, *readable* token. Not encrypted. Not revocable.
**Claim** — a field in a JWT payload (`sub`, `exp`, `iat`).
**Bearer token** — "whoever holds this is authorized." Treat like a password.
**Refresh token** — a long-lived, revocable token used to get new short-lived access tokens.
**OAuth 2.0** — delegated authorization: let an app access an API on your behalf.
**OpenID Connect (OIDC)** — an identity layer on OAuth 2.0. "Sign in with X."
**ID token** — the OIDC JWT describing who the user is.
**Authorization code flow** — the secure OAuth flow: redirect, code, server-side exchange.
**PKCE** — proof that the party redeeming the code started the flow.
**State parameter** — CSRF protection for the OAuth redirect.
**Scope** — the specific permissions being requested.
**CSRF** — tricking a logged-in user's browser into making a request. Mitigated by `SameSite` and tokens.
**XSS** — injecting JavaScript into a page. Mitigated by escaping output and `HttpOnly` cookies.
**IDOR** — accessing another user's object by guessing its id. An access-control failure.
**RBAC / ABAC / ReBAC** — role-, attribute-, and relationship-based access control.
**Least privilege** — grant the minimum access needed.

---

## Frontend

**DOM** — the browser's live object tree representing the page.
**Element vs component** — a DOM node vs a reusable piece of your UI.
**JSX** — HTML-like syntax that compiles to function calls.
**Props** — data passed into a component. Read-only.
**State** — data a component owns that can change over time.
**Hook** — a function letting a component use React features (`useState`, `useEffect`).
**Render** — React calling your component function to produce a description of the UI.
**Reconciliation** — diffing the new description against the old to find minimal DOM changes.
**Key** — a stable identity for a list item across renders.
**Controlled / uncontrolled input** — value driven by React state, or by the DOM.
**Derived state** — a value computed from other state. Compute it; don't store it.
**Server state** — data owned by the backend, cached on the client. Use a query library.
**Stale closure** — a callback capturing an outdated value from an old render.
**Memoization** — caching a computed result (`useMemo`, `useCallback`).
**Optimistic update** — update the UI before the server confirms, roll back on failure.
**Hydration** — attaching React to server-rendered HTML.
**Bundler** — combines and transforms your modules for the browser. Vite, esbuild, Rollup.
**Tree shaking** — dropping unused code from the bundle.
**HMR** — hot module replacement; swap a changed module without reloading.
**Code splitting** — loading parts of your JS on demand.
**a11y** — accessibility. Usable by everyone, including keyboard and screen-reader users.
**ARIA** — attributes conveying meaning to assistive technology when HTML can't.
**Focus trap** — keeping keyboard focus inside a modal while it's open.

---

## Containers

**Container** — a process with a namespaced view of the system and cgroup limits on it. Not a VM.
**Namespace** — a kernel feature isolating what a process can *see*: PIDs, mounts, network, users, hostname.
**cgroup** — a kernel feature limiting what a process can *use*: CPU, memory, I/O.
**Image** — an ordered stack of read-only filesystem layers plus metadata. The thing you deploy.
**Layer** — one filesystem diff, produced by one Dockerfile instruction. Content-addressed and shared between images.
**Build cache** — reuse of unchanged layers. Invalidated by the first changed instruction and everything after it.
**Build context** — the files sent to the builder before the build starts. Trimmed by `.dockerignore`.
**Multi-stage build** — several `FROM` stages where only the last one ships; keeps compilers and dev deps out of production.
**Volume** — Docker-managed storage that outlives a container. Where your database data actually lives.
**Bind mount** — a host directory mapped into a container. Good for dev hot-reload, not for production.
**Port publishing** (`-p 3000:3000`) — exposing a container port on the host. `EXPOSE` alone does nothing.
**Registry** — where images are stored and pulled from. Docker Hub, GHCR, ECR.
**Tag vs digest** — a mutable name (`:1.2.0`, `:latest`) vs an immutable content hash (`@sha256:…`). Deploy digests.
**Exit code 137** — the container was killed by SIGKILL; almost always the OOM killer hitting a memory limit.
**Graceful shutdown** — handling `SIGTERM` to finish in-flight work and close connections before exiting.
**Orchestrator** — runs containers across many machines: scheduling, restarts, rolling deploys. Kubernetes, Nomad, ECS.
**Distroless** — a base image containing only your app and its runtime — no shell, no package manager.

## Testing & delivery

**Unit / integration / E2E** — one function / several parts together / the whole system.
**Test double** — any stand-in: mock, stub, spy, fake.
**Mock** — a fake dependency you assert against. Use sparingly.
**Fixture / factory** — reusable test data setup.
**Assertion** — the check that decides pass or fail.
**Coverage** — the percentage of lines executed by tests. A diagnostic, not a target.
**Flaky test** — passes and fails without code changes. Worse than no test.
**Regression test** — a test written for a bug you already had, so it can't come back.
**TDD** — write the failing test first.
**CI** — automatically running checks on every push.
**CD** — continuous delivery (always deployable) or deployment (ships automatically).
**Pipeline** — the sequence of automated steps from commit to production.
**Artifact** — the built output that gets deployed.
**Environment** — local, preview, staging, production.
**Feature flag** — a switch to enable code without redeploying.
**Blue-green / canary / rolling** — deployment strategies.
**Rollback** — reverting to the previous version.
**Observability** — being able to tell what a running system is doing.
**Structured logging** — logs as machine-parseable fields, not prose.
**Metric** — a number over time. Use percentiles (p95), not averages.
**Trace** — one request's path across services, with timings.
**SLO / SLA** — a reliability target you set / a reliability promise you make contractually.
**Postmortem** — a blameless write-up after an incident.
**Technical debt** — accepted shortcuts that will cost more later. Sometimes a good trade — if it's deliberate and tracked.

---

## Process

**Epic / story / task / spike** — big goal / user-facing value / implementation step / timeboxed investigation.
**Acceptance criteria** — the checklist defining "done" for a ticket.
**Definition of Done** — the team-wide standard every ticket must meet.
**Backlog** — the ordered list of what's not started.
**Sprint** — a fixed-length iteration, usually two weeks.
**Standup / retro / refinement** — daily blocker sync / process improvement / breaking work down.
**Velocity** — points completed per sprint. A planning aid, never a performance metric.
**WIP limit** — a cap on how much work is in progress at once.
**Trunk-based development** — short-lived branches merged frequently into `main`.
**Code review** — a colleague reads your change before it merges.
**LGTM / nit / blocking** — review shorthand: looks good to me / trivial preference / must fix.
**Bikeshedding** — arguing at length about something trivial because it's easy to have opinions about.
**Yak shaving** — the recursive chain of prerequisites you hit before the actual task.
**Rubber ducking** — explaining the problem aloud until you spot the answer yourself.
