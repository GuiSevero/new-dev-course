# Module 14 — Capstone & What's Next

**Time:** 8+ hours
**Prereq:** everything

---

## Part 1 — Finish and ship

At this point you should have a working, deployed TaskFlow. Before you call it done, walk this checklist. Be honest; nobody's watching.

### Functionality
- [ ] Sign up with email + password
- [ ] Sign in with GitHub (OAuth)
- [ ] Sign out, and confirm the session is actually gone (not just the UI)
- [ ] Create, rename, and delete lists
- [ ] Create, edit, complete, and delete tasks
- [ ] Filter by status, with the filter in the URL
- [ ] Every screen has loading, error, and empty states

### Correctness & security
- [ ] Every endpoint validates its input, with sensible 400s
- [ ] Every endpoint requires a session (401 without one)
- [ ] Every endpoint checks ownership (404 for another user's resource)
- [ ] No request body can set a field you didn't intend (mass assignment)
- [ ] No secret is in the repository; `.env.example` documents what's needed
- [ ] Error responses leak nothing — no stack traces, SQL, or paths
- [ ] Session cookie is `HttpOnly`, `Secure` in production, and `SameSite` is set
- [ ] Login is rate limited
- [ ] Database constraints enforce your invariants independently of app code

### Quality
- [ ] `npm run typecheck`, `lint`, `test`, `build` all pass from a clean clone
- [ ] Integration test per endpoint covering happy path + 400 + 401 + 404
- [ ] At least one Playwright journey
- [ ] CI runs on every PR; `main` is protected
- [ ] The whole app is usable with only a keyboard
- [ ] Lighthouse accessibility score above 95

### Operations
- [ ] `docker compose up` from a clean clone brings up the whole stack, migrated, with no manual steps
- [ ] The API image is multi-stage, runs as a non-root user, and contains no secrets or dev dependencies
- [ ] CI builds the image, tests it, and pushes it tagged with the git SHA — never `latest` alone
- [ ] The API shuts down gracefully on `SIGTERM` (closes the pool, finishes in-flight requests)
- [ ] Deployed and reachable at a public URL
- [ ] Migrations run automatically on deploy
- [ ] Sentry (or equivalent) captures errors from both apps
- [ ] The README lets a stranger run it locally in under 10 minutes
- [ ] You can roll back a bad deploy — and you've actually done it once, on purpose, and timed it

**The real test:** hand the README to someone who has never seen the project and watch them try to run it. Say nothing. Note every place they get stuck. Fix all of it.

---

## Part 2 — Extend it, alone

This is where the course actually gets graded. Pick **two** of these and build them **without a step-by-step guide**. Use Claude freely — but as a colleague you're directing, not an oracle you're obeying. You should be able to explain every line.

### 🟢 Approachable
- **Due dates and overdue highlighting.** Timezones will bite. Store UTC (Module 07), format in the user's locale.
- **Search.** Start with `ILIKE '%term%'`, then discover why that can't use an index, then look into [Postgres full-text search](https://www.postgresql.org/docs/current/textsearch.html).
- **Drag to reorder.** The interesting part isn't the drag library, it's the data model — how do you store an order that survives concurrent inserts without renumbering every row? (Look up fractional indexing.)
- **Dark mode.** Persisted, respecting `prefers-color-scheme`, with no flash on load.

### 🔵 Short experiments (do these even if you pick two features from below)

- **Run your API on a different runtime.** Install [Bun](https://bun.sh/) and start the same app with `Bun.serve({ fetch: app.fetch, port: 3000 })` instead of `@hono/node-server`. Every route, every middleware, every test unchanged. Then articulate *why* that worked — the answer is the whole point of Module 08.1.
- **Add a typed client with [Hono RPC](https://hono.dev/docs/guides/rpc).** Export your app's type, use `hc<AppType>` in the frontend, and delete your hand-written `request<T>()` wrapper. Now a typo in a URL or a wrong request body is a *compile* error. Then change a response shape in the API and watch the frontend refuse to build. This is the end state of the "define once, derive everywhere" thread running through Modules 05, 08 and 10.

### 🟡 Meatier
- **Tags** — a many-to-many relationship (Module 07). Filter by multiple tags. Watch out for the N+1 (Module 03).
- **Soft delete + undo.** Add `deleted_at`, filter it everywhere, offer a 10-second undo. Notice how one nullable column now affects every query you've written — that's a real lesson about schema decisions.
- **Optimistic UI everywhere**, with correct rollback on failure. Test it under network throttling.
- **Real-time updates** with Server-Sent Events or WebSockets. Two browser tabs stay in sync. This breaks the request/response model from Module 06 — understand why you need something else.
- **Email notifications** for tasks due tomorrow. Needs a scheduled job, an email provider, idempotency (don't send twice), and a way to test it without spamming yourself.

### 🔴 Genuinely hard
- **Shared lists.** Invite another user; roles of viewer/editor/owner. This turns your ownership model into real authorization (Module 09) and touches every single query.
- **Offline support.** Optimistic local writes, a sync queue, conflict resolution. You'll learn why distributed state is hard.
- **Activity log / audit trail.** Every change recorded with who and when. Consider what this costs in writes.
- **Multi-tenancy** with Postgres [row-level security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) — authorization enforced by the database itself, not your code.

**For each feature, follow the whole loop:** write the ticket with acceptance criteria → design the schema change → write the migration → tests → API → UI → PR → review → CI → deploy → verify in production.

---

## Part 3 — The exam

No code. Sit down with a blank document and write, in your own words, without looking anything up. Give yourself two hours.

1. **Trace a request end to end.** A user clicks "Complete" on a task. Describe everything that happens: the React event handler, the mutation, the HTTP request (method, path, headers, body), DNS/TCP/TLS, your server's routing, validation, the session lookup, the authorization check, the SQL that runs, the constraint that's enforced, the response, the cache invalidation, the re-render. Name every layer. **Be specific about what could go wrong at each step.**

2. **Draw your architecture from memory.** Every box, every arrow, every protocol. Then annotate it with where each of these lives: validation, authorization, business logic, caching, error handling.

3. **Explain to a non-technical person**, in under 200 words each: what a database index is; what a session cookie does; why the same code needs testing at three different levels.

4. **Security review your own app.** Write down five ways someone could attack it. Then go check whether each one works.

5. **What would break at 100,000 users?** Be specific — which query, which endpoint, which resource runs out first. What would you measure to find out?

6. **What are the three worst decisions in your codebase?** Every codebase has them. Knowing yours is the mark of an engineer rather than a code-typist.

Then, and only then:

```text
Prompt for Claude Code:
I just finished a full-stack course and built [describe TaskFlow]. Here are
my written answers to a self-assessment: [paste all six].

Grade me honestly as a hiring manager for a junior engineer role. For each
answer: what's correct, what's vague, what's wrong, and what a strong
candidate would have said instead. Don't be nice — be useful. End with the
three concepts I most need to shore up before interviewing.
```

---

## Part 4 — What to learn next

You now have a foundation. Here's the map of what's adjacent, roughly in order of usefulness.

### Deepen the fundamentals
- 🎓 [CS50x](https://cs50.harvard.edu/x/) — if you skipped it, do it now. C, memory, algorithms. It will make everything you've learned feel less like magic.
- 📖 [Teach Yourself Computer Science](https://teachyourselfcs.com/) — a curated, opinionated self-study curriculum with one best book/course per topic. The most honest guide to filling CS gaps.
- 📕 [*Designing Data-Intensive Applications*](https://dataintensive.net/) (Kleppmann) — the single best technical book of the last decade. Read it slowly, over months.
- 🎓 [MIT Missing Semester](https://missing.csail.mit.edu/) — the rest of the lectures.
- 🎓 [Nand2Tetris](https://www.nand2tetris.org/) — build a computer from logic gates up to an OS. Transformative if you want to *really* know.

### Broaden the stack
- **Caching & Redis** — [Redis University](https://redis.io/university/) is free
- **Message queues** — background jobs, retries, dead letter queues
- **Docker & containers, properly** — [Docker docs](https://docs.docker.com/), then Kubernetes only if you actually need it
- **Cloud** — pick one (AWS, GCP, Cloudflare) and learn its primitives: compute, object storage, managed DB, queue, CDN
- **Infrastructure as Code** — [Terraform](https://developer.hashicorp.com/terraform/tutorials) or [Pulumi](https://www.pulumi.com/docs/)
- **Another language** — [Go](https://go.dev/tour/) (simple, concurrent, great for services) or [Rust](https://doc.rust-lang.org/book/) (hard, and teaches you memory properly). A second language is what turns "I know TypeScript" into "I know programming."
- **System design** — [System Design Primer](https://github.com/donnemartin/system-design-primer), [ByteByteGo](https://bytebytego.com/)

### Sharpen the craft
- 📕 [*A Philosophy of Software Design*](https://web.stanford.edu/~ouster/cgi-bin/book.php) (Ousterhout) — short, and the best book on managing complexity
- 📕 [*The Pragmatic Programmer*](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/) — on being a professional
- 📕 [*Refactoring*](https://martinfowler.com/books/refactoring.html) (Fowler) — the named moves
- 📖 [Julia Evans' blog & zines](https://jvns.ca/) — the best writing on *how to learn* systems topics, plus excellent debugging material
- 📖 [Dan Luu](https://danluu.com/) — sharp, evidence-based writing on how software actually works in industry

### Build things
The real answer. Pick something you personally want to exist and build it badly, then improve it. Ideas that stretch different muscles: a bookmarking tool with full-text search (search + data modelling), a habit tracker with charts (time-series + visualization), a Slack bot (webhooks + async), a personal finance importer (parsing + money + idempotency), a URL shortener (caching + scale + analytics).

**Ship it publicly.** A deployed thing with a README beats a folder of tutorial code, both for learning and for hiring.

---

## Part 5 — Using AI well from here

You've had Claude alongside you for 100 hours. A few things worth stating explicitly now that you have the context to judge them.

**What it's excellent at:**
- Explaining unfamiliar code, at whatever depth you ask for
- Boilerplate and mechanical transformations
- "What is this error telling me?"
- Generating test cases and edge cases you didn't think of
- Reviewing your code adversarially — the "attack this" prompts in Modules 08, 09, 11 are genuinely valuable
- Being a rubber duck that talks back
- Exploring an unfamiliar library's API without reading all the docs first

**Where it will quietly hurt you:**
- **Confident wrongness.** It will produce plausible code for an API that doesn't exist. Verify against primary docs.
- **Skipped struggle.** The productive discomfort of being stuck is where learning happens. Ask *after* 30 minutes, not before 30 seconds.
- **Accumulated unowned complexity.** Code you didn't write and don't understand grows until you can't debug it. This is the failure mode, and it's slow enough that you don't notice until it's bad.
- **Architecture.** It optimizes locally, for the prompt in front of it. Cross-cutting design decisions are yours.
- **Security and data modelling.** It doesn't know your threat model or your business rules. These are the two areas to be most skeptical.

**The habits that keep you sharp:**
1. Write the plan yourself, before prompting.
2. Read every line before accepting it. If you can't explain it, don't keep it.
3. Ask "why this and not X?" — the comparison is where the understanding is.
4. Have it quiz you, not just answer you.
5. Once a week, build something small with no AI at all. It's a check on whether the skill is still there.

> The engineers who will thrive are not the ones who type fastest, and not the ones who refuse to use AI. They're the ones with **enough depth to know when the output is wrong** — and that depth only comes from doing the hard parts yourself, at least once.

You just did the hard parts. Go build something.
