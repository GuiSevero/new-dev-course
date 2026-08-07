# Module 08 — Building the Backend API

**Time:** ~10 hours
**Prereq:** Modules [02](02-programming-fundamentals.md), [05](05-project-organization.md), [06](06-client-server-http.md), [07](07-databases-sql.md)

---

## Why this module exists

This is where it becomes real: a program that listens on a port, speaks HTTP, validates untrusted input, talks to Postgres, and hands back JSON. Every concept from the previous modules shows up at once.

> **The code in this module is a reference sketch, not a copy-paste solution.** Library APIs change; check the linked docs for current syntax. More importantly: typing it yourself, hitting the errors, and fixing them *is the exercise*. If you paste your way through this module you will have learned nothing and you'll know it in Module 14.

---

## 8.1 What a web server actually is

Strip away the framework. This is Node's built-in HTTP server, with nothing installed:

```ts
// raw.ts — run with: npx tsx raw.ts
import { createServer } from "node:http";

createServer((req, res) => {
  if (req.method === "GET" && req.url === "/health") {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ ok: true }));
    return;
  }
  res.writeHead(404, { "Content-Type": "text/plain" });
  res.end("Not Found");
}).listen(3000);
```

Run it, then `curl -v localhost:3000/health` and `curl -v localhost:3000/nope`. **That is a web server.** A request comes in, you branch on method and path, you write a response. Everything else is convenience.

Now the same thing written against the **web standard** `Request`/`Response` objects — the exact same classes the browser's `fetch` uses (Module 06):

```ts
// standard.ts
import { serve } from "@hono/node-server";

function handler(req: Request): Response {
  const url = new URL(req.url);
  if (req.method === "GET" && url.pathname === "/health") {
    return Response.json({ ok: true });
  }
  return new Response("Not Found", { status: 404 });
}

serve({ fetch: handler, port: 3000 });
```

**Hold onto this shape: `(Request) => Response`.** A Hono application *is* a function with exactly that signature — `app.fetch` is the handler, and `serve()` is just the thing that plugs it into Node's socket layer.

That's not a metaphor, it's literally true, and it has three consequences you'll use in this course:

1. The framework is a **thin, inspectable layer**, not a black box. Everything it adds — routing, body parsing, validation, middleware, error handling — sits between those two objects.
2. **The same code runs anywhere** that speaks the standard: Node, Bun, Deno, Cloudflare Workers. Swapping the runtime means swapping the `serve()` line.
3. **You can test it without a server.** `app.fetch(new Request(...))` returns a `Response`. No ports, no network, no mocking (Module 11).

Everything a framework adds is convenience over `(Request) => Response`. Knowing the primitive underneath means you'll never think of a framework as magic.

---

## 8.2 Layered architecture

Do not put database queries inside route handlers. Here's the structure, and why each layer exists:

```
HTTP request
    ↓
[ Route ]        parse + validate input, call a service, format the response.
    ↓            Knows about HTTP. Knows nothing about SQL.
[ Service ]      business rules, authorization, orchestration, transactions.
    ↓            Knows nothing about HTTP or SQL. This is your actual application.
[ Repository ]   database queries. Knows about SQL. Knows nothing about HTTP.
    ↓
PostgreSQL
```

**Dependencies point one way, always.** A repository must never import a route; a service must never see a `Request` object.

Why bother, for a todo app? Three concrete payoffs:

1. **Testability.** Services are plain functions — testable without starting a server (Module 11).
2. **You can change one layer without touching the others.** Swap REST for GraphQL: rewrite routes only.
3. **It's where the bugs aren't.** Business rules scattered across route handlers get duplicated, then drift, then contradict each other.

The cost is more files. For a 3-endpoint app that's overhead; for anything real it pays back within a month. We're doing it because *learning where logic belongs* is the point.

📖 [Martin Fowler: PresentationDomainDataLayering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)

---

## 8.3 Drizzle: schema as TypeScript

An **ORM** maps database rows to language objects. [Drizzle](https://orm.drizzle.team/) is a thin, SQL-shaped one: queries look like SQL, and the types are derived from your schema.

```bash
cd apps/api
npm install drizzle-orm pg
npm install -D drizzle-kit @types/pg tsx
```

```ts
// apps/api/src/db/schema.ts
import { pgTable, uuid, text, timestamp, smallint, pgEnum, unique, index } from "drizzle-orm/pg-core";

export const taskStatus = pgEnum("task_status", ["todo", "in_progress", "done"]);

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  name: text("name").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const lists = pgTable("lists", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => [unique().on(t.userId, t.name)]);

export const tasks = pgTable("tasks", {
  id: uuid("id").primaryKey().defaultRandom(),
  listId: uuid("list_id").notNull().references(() => lists.id, { onDelete: "cascade" }),
  title: text("title").notNull(),
  description: text("description"),
  status: taskStatus("status").notNull().default("todo"),
  priority: smallint("priority").notNull().default(0),
  dueDate: timestamp("due_date", { withTimezone: true }),
  completedAt: timestamp("completed_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => [index("tasks_list_id_idx").on(t.listId)]);

// Types derived from the schema — one source of truth, all the way to the browser
export type Task = typeof tasks.$inferSelect;
export type NewTask = typeof tasks.$inferInsert;
```

Those last two lines are the payoff: **the shape of a task is defined once, in the database schema, and flows into your service layer, your API types, and (via `packages/shared`) your React components.** Change a column, and everything that's now wrong stops compiling.

### Migrations

```ts
// apps/api/drizzle.config.ts
import { defineConfig } from "drizzle-kit";
export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

```bash
npx drizzle-kit generate    # diff schema.ts against migrations → write a new .sql file
npx drizzle-kit migrate     # apply pending migrations
npx drizzle-kit studio      # a GUI to browse your data
```

**Open the generated `.sql` file and read it before applying it.** Every time. Migration tools are helpful, not omniscient — and a generated `DROP COLUMN` you didn't notice is how data disappears. This is the single most important habit in this section.

### Queries

```ts
// apps/api/src/db/client.ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";
import { env } from "../config";

// A connection POOL, not a connection. Opening a TCP+auth handshake per query
// would be absurdly slow, so we keep a handful open and reuse them.
const pool = new Pool({ connectionString: env.DATABASE_URL, max: 10 });
export const db = drizzle(pool, { schema });
```

```ts
import { and, eq, desc, lt, sql } from "drizzle-orm";

// SELECT ... WHERE list_id = $1 AND status = $2 ORDER BY created_at DESC LIMIT 20
const rows = await db.select().from(tasks)
  .where(and(eq(tasks.listId, listId), eq(tasks.status, "todo")))
  .orderBy(desc(tasks.createdAt))
  .limit(20);

const [created] = await db.insert(tasks).values({ listId, title }).returning();

const [updated] = await db.update(tasks)
  .set({ status: "done", completedAt: new Date(), updatedAt: new Date() })
  .where(eq(tasks.id, id))
  .returning();

// A transaction: both statements commit, or neither does
await db.transaction(async (tx) => {
  await tx.insert(tasks).values({ listId, title });
  await tx.update(lists).set({ updatedAt: new Date() }).where(eq(lists.id, listId));
});
```

Compare each of these to the SQL you wrote by hand in Module 07 — that's why you did that first. An ORM you can't mentally translate to SQL is an ORM that will surprise you.

📖 [Drizzle docs](https://orm.drizzle.team/docs/overview) · [Query API](https://orm.drizzle.team/docs/rqb) · [Migrations](https://orm.drizzle.team/docs/migrations)

---

## 8.4 Hono: the HTTP layer

[Hono](https://hono.dev/) is a small, fast router built directly on the `(Request) => Response` shape from 8.1.

```bash
npm install hono @hono/node-server zod @hono/zod-validator
```

```ts
// apps/api/src/index.ts
import { Hono } from "hono";
import { serve } from "@hono/node-server";
import { cors } from "hono/cors";
import { logger } from "hono/logger";
import { tasksRoutes } from "./features/tasks/tasks.routes";
import { env } from "./config";

const app = new Hono();

app.use("*", logger());
app.use("/api/*", cors({ origin: env.WEB_ORIGIN, credentials: true }));  // ← Module 06's CORS

app.get("/health", (c) => c.json({ ok: true }));
app.route("/api/tasks", tasksRoutes);

serve({ fetch: app.fetch, port: env.PORT }, (info) => {
  console.log(`API listening on http://localhost:${info.port}`);
});
```

That `serve({ fetch: app.fetch })` is the same line from 8.1. **The app and the server are separate things** — which is exactly why it's testable and portable.

### Middleware

`app.use()` registers a function that runs *around* your handler:

```ts
app.use("*", async (c, next) => {
  const start = Date.now();
  await next();                     // ← everything downstream runs here
  c.header("X-Response-Time", `${Date.now() - start}ms`);
});
```

It's an onion: each middleware can act before `next()`, after it, or short-circuit by returning a response without calling it at all (that's how an auth guard works — Module 09). `logger()` and `cors()` above are just pre-built ones.

### Routes and validation

```ts
// apps/api/src/features/tasks/tasks.routes.ts
import { Hono } from "hono";
import { z } from "zod";
import { zValidator } from "@hono/zod-validator";
import { taskService } from "./tasks.service";

const ListQuery = z.object({
  status: z.enum(["todo", "in_progress", "done"]).optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),  // ← query strings are ALWAYS strings; coerce
});

const CreateTask = z.object({
  listId: z.string().uuid(),
  title: z.string().min(1, "Title is required").max(200),
  description: z.string().max(2000).optional(),
  dueDate: z.coerce.date().optional(),      // ← "2026-09-01T…" (a string, per Module 06) becomes a Date
});

const UpdateTask = CreateTask.pick({ title: true, description: true })
  .extend({ status: z.enum(["todo", "in_progress", "done"]) })
  .partial();                                // ← PATCH: every field optional (Module 06)

const IdParam = z.object({ id: z.string().uuid() });

export const tasksRoutes = new Hono()
  .get("/", zValidator("query", ListQuery), async (c) => {
    const { status, limit } = c.req.valid("query");     // typed, validated, coerced
    return c.json(await taskService.list({ status, limit }));
  })

  .post("/", zValidator("json", CreateTask), async (c) => {
    const task = await taskService.create(c.req.valid("json"));
    c.header("Location", `/api/tasks/${task.id}`);
    return c.json(task, 201);                            // ← Module 06: 201 for created
  })

  .patch("/:id", zValidator("param", IdParam), zValidator("json", UpdateTask), async (c) => {
    const { id } = c.req.valid("param");
    return c.json(await taskService.update(id, c.req.valid("json")));
  })

  .delete("/:id", zValidator("param", IdParam), async (c) => {
    await taskService.remove(c.req.valid("param").id);
    return c.body(null, 204);                            // ← no body
  });
```

> Zod v4 also offers shorter top-level forms (`z.uuid()`, `z.email()`). `z.string().uuid()` works in both v3 and v4 — check [zod.dev](https://zod.dev/) for what your installed version prefers.

**The Zod schemas are the most important thing on this page.** They are *runtime* validation at the boundary — the thing Module 01 told you TypeScript cannot do for you, because types are erased before the server ever starts. A request with a missing title, a 5,000-character description, or `listId: 12345` is rejected with a 400 before your service runs.

Two things worth noticing:

- **`c.req.valid("json")` is fully typed**, inferred from the schema. One declaration gives you the runtime check *and* the compile-time type. Same principle as Drizzle's `$inferSelect` above and `z.infer` in the React form (Module 10) — **define once, derive everywhere.**
- **These are the same schemas the frontend uses.** Move `CreateTask` into `packages/shared` and your React form (Module 10) validates against the identical rules. Change the max length in one place; both ends update. That's the payoff of the monorepo from Module 05.

By default `zValidator` returns a 400 with Zod's raw issue list. To shape it into your standard error envelope, pass a hook:

```ts
zValidator("json", CreateTask, (result, c) => {
  if (!result.success) {
    return c.json({
      error: {
        code: "VALIDATION_FAILED",
        message: "Invalid request body",
        details: result.error.issues.map(i => ({ field: i.path.join("."), issue: i.message })),
      },
    }, 400);
  }
})
```

> **Bonus, once you're comfortable:** because the routes above are *chained*, Hono can export the app's full type, and [`hc<AppType>`](https://hono.dev/docs/guides/rpc) gives your frontend a typed client where the paths, params and response shapes are all checked at compile time. Not required for this course — but look at it in Module 14, because it's the logical conclusion of everything the shared-types thread has been building toward.

📖 [Hono docs](https://hono.dev/docs/) · [Routing](https://hono.dev/docs/api/routing) · [Middleware](https://hono.dev/docs/guides/middleware) · [Validation](https://hono.dev/docs/guides/validation) · [Node adapter](https://hono.dev/docs/getting-started/nodejs)

> **On frameworks generally:** the concepts here — routing, middleware chains, boundary validation, typed handlers — are universal. [Fastify](https://fastify.dev/) is the most widely deployed Node framework and what you're most likely to meet at an established company; [Express](https://expressjs.com/) is the older one you'll see in most tutorials and legacy code. Both express the same ideas with different syntax and their own (non-standard) request/response objects. Learn one properly; the second takes an afternoon.

---

## 8.5 Service and repository layers

```ts
// apps/api/src/features/tasks/tasks.repository.ts
import { and, eq, desc } from "drizzle-orm";
import { db } from "../../db/client";
import { tasks, lists, type NewTask } from "../../db/schema";

export const taskRepository = {
  async findById(id: string) {
    const [row] = await db.select().from(tasks).where(eq(tasks.id, id)).limit(1);
    return row ?? null;
  },

  // Ownership is enforced in the QUERY, not by filtering afterwards.
  // If you fetch first and check later, you have already leaked the row to your process
  // and it's one careless `return` away from leaking to the user.
  async findByIdForUser(id: string, userId: string) {
    const [row] = await db.select({ task: tasks })
      .from(tasks)
      .innerJoin(lists, eq(lists.id, tasks.listId))
      .where(and(eq(tasks.id, id), eq(lists.userId, userId)))
      .limit(1);
    return row?.task ?? null;
  },

  async create(input: NewTask) {
    const [row] = await db.insert(tasks).values(input).returning();
    return row;
  },
};
```

```ts
// apps/api/src/features/tasks/tasks.service.ts
import { taskRepository } from "./tasks.repository";
import { NotFoundError, ForbiddenError } from "../../lib/errors";

export const taskService = {
  async create(input: { listId: string; title: string; dueDate?: string }, userId: string) {
    // Business rules live HERE — not in the route, not in the repository.
    const list = await listRepository.findByIdForUser(input.listId, userId);
    if (!list) throw new NotFoundError("list", input.listId);

    return taskRepository.create({
      listId: input.listId,
      title: input.title.trim(),
      dueDate: input.dueDate ? new Date(input.dueDate) : null,
    });
  },

  async complete(id: string, userId: string) {
    const task = await taskRepository.findByIdForUser(id, userId);
    if (!task) throw new NotFoundError("task", id);          // 404, not 403 — see below
    if (task.status === "done") return task;                 // idempotent (Module 06)

    return taskRepository.update(id, { status: "done", completedAt: new Date() });
  },
};
```

> **404 vs 403 for someone else's task.** Returning 403 tells the caller *"that id exists, you just can't have it"* — an information leak that lets an attacker enumerate your data. Returning 404 tells them nothing. Use 404 for resources the user isn't allowed to know about, and 403 only when they already legitimately know it exists.

---

## 8.6 Errors, logging, and the boundary

```ts
// apps/api/src/lib/errors.ts

// Constraining the status to the ones we actually use means a typo like `4004`
// is a compile error, not a runtime surprise.
export type ErrorStatus = 400 | 401 | 403 | 404 | 409 | 422 | 429 | 500;

export class AppError extends Error {
  constructor(
    message: string,
    public readonly status: ErrorStatus,
    public readonly code: string,
  ) {
    super(message);
    this.name = new.target.name;
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, public readonly id: string) {
    super(`${resource} not found`, 404, "NOT_FOUND");
  }
}
export class ForbiddenError extends AppError {
  constructor(msg = "Forbidden") { super(msg, 403, "FORBIDDEN"); }
}
export class UnauthorizedError extends AppError {
  constructor(msg = "Authentication required") { super(msg, 401, "UNAUTHENTICATED"); }
}
```

```ts
// apps/api/src/index.ts — one place that turns any thrown error into a well-formed response

app.onError((err, c) => {
  if (err instanceof AppError) {
    return c.json({ error: { code: err.code, message: err.message } }, err.status);
  }
  // Unknown error: log the details for YOU, return nothing useful to the CALLER.
  console.error({ err, path: c.req.path, method: c.req.method });
  return c.json({ error: { code: "INTERNAL", message: "Something went wrong" } }, 500);
});

app.notFound((c) =>
  c.json({ error: { code: "NOT_FOUND", message: "No such endpoint" } }, 404),
);
```

Because `onError` catches anything thrown anywhere downstream, your services can simply `throw new NotFoundError("task", id)` and never think about HTTP again — which is exactly the layering rule from 8.2.

> 🔒 **Never return a stack trace, SQL error, or file path to a client.** Those messages tell an attacker your framework, your schema, and your directory layout. Log them server-side; return an opaque message and a request id the user can quote to support.

**Logging.** `console.log` is fine to learn with; production wants **structured** logs (JSON with fields, not prose) so they can be searched and alerted on. Look at [pino](https://getpino.io/). Log: request method, path, status, duration, user id, and a **request id** that ties every log line for one request together. Never log passwords, tokens, or full request bodies.

---

## 8.7 Security checklist for any API

| Concern | What to do |
|---|---|
| **Injection** | Parameterized queries only. Drizzle does this — don't break it with raw string interpolation. |
| **Validation** | Validate every input at the boundary. Reject unknown fields. Set max lengths. |
| **Mass assignment** | Never spread a request body into a DB update. A user could send `{"userId": "<someone else>"}`. Pick fields explicitly. |
| **Broken access control** | Check ownership on *every* read and write. It's [OWASP #1](https://owasp.org/Top10/A01_2021-Broken_Access_Control/). |
| **Rate limiting** | Cap requests per IP/user, especially on login. |
| **Secrets** | Environment only. Never in code, logs, or error responses. |
| **CORS** | An explicit allow-list of origins. Never `*` with credentials. |
| **Payload size** | Cap request body size or someone will POST a 2 GB file. |
| **Dependencies** | `npm audit`; keep them current. |

📖 [OWASP Top 10](https://owasp.org/www-project-top-ten/) · [OWASP API Security Top 10](https://owasp.org/www-project-api-security/) · [Cheat Sheet Series](https://cheatsheetseries.owasp.org/)

---

## Lab 08 — Build the TaskFlow API

Work endpoint by endpoint. **After each one, test it with `curl` before writing the next.** Do not build all seven and then debug.

1. `GET /health` — returns `{ ok: true }`. Get the server running first.
2. Wire up Drizzle: schema, config, `drizzle-kit generate`, **read the generated SQL**, `migrate`. Confirm the tables exist with `psql` or `drizzle-kit studio`.
3. `POST /api/lists` and `GET /api/lists`.
4. `POST /api/tasks` — with validation. Then deliberately send: no title, an empty title, a 10,000-character title, a `listId` that isn't a UUID, a `listId` that's a valid UUID but doesn't exist. **Each should fail with a sensible status and message.** Fix the ones that don't.
5. `GET /api/tasks?listId=&status=&limit=&cursor=` — with real pagination.
6. `PATCH /api/tasks/:id` — partial updates. Confirm that omitting a field leaves it unchanged (Module 06: this is what makes it PATCH, not PUT).
7. `DELETE /api/tasks/:id` — 204 on success, 404 if absent. Call it twice; the second call should 404, and you should be able to argue whether that's the right choice.
8. Centralized error handling. Then throw an error on purpose in a service and confirm the client gets a clean message while your terminal gets the stack trace.
9. Hardcode a `userId` for now (auth arrives in Module 09) and enforce ownership on every endpoint.

**Then break it deliberately:**
- Stop the database container while the server runs. What happens on the next request? Is that a 500 with a readable log, or a crash?
- Send `Content-Type: text/plain` with a JSON body.
- Send `{"title": "x", "isAdmin": true}` — does your validation reject unknown fields, or silently accept them? (Find out. It matters.)
- Send 1,000 requests in a loop (`for i in $(seq 1 1000); do curl -s ... ; done`). What breaks first?

```text
Prompt for Claude Code:
Here is my TaskFlow API: [paste routes, service, repository].

Review it as a security-minded backend engineer. Specifically check for:
1. Broken access control — can user A reach user B's data through ANY path?
2. Mass assignment — can a request body set fields I didn't intend?
3. Inputs that reach the database without validation
4. Error responses that leak internals
5. Any N+1 query pattern
6. Business logic that's leaked into the route layer

Give me a numbered list with file:line references. Don't fix anything —
I'll fix them and then you re-review.
```

---

## Understanding Gate

1. Why validate input at the API boundary when TypeScript already types the body?
2. What is a connection pool, and what happens without one?
3. Why must ownership be checked in the SQL `WHERE` clause rather than after fetching?
4. Route, service, repository — which layer knows about HTTP status codes? Which knows about SQL? Why does that separation matter?
5. What's mass assignment, and how does explicitly picking fields prevent it?
6. Your endpoint returns 500 with `error: relation "tasks" does not exist`. Name two things wrong with that response.
7. Why does `DELETE` return 204 rather than 200 with a body?
8. Why is `taskService.complete` written to be idempotent?

---

**Next:** [Module 09 — Authentication & Authorization](09-auth.md)
