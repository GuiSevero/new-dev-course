# Module 07 — Databases and SQL

**Time:** ~8 hours
**Prereq:** [Module 03](03-complexity-and-data.md), [Module 05](05-project-organization.md)

---

## Why this module exists

Your application code is disposable — you'll rewrite it, change frameworks, throw the frontend away. **Your data outlives all of it.** A bad data model is the one mistake you can't refactor your way out of cheaply, because there's production data in the wrong shape.

This is also the layer where AI assistance is least reliable, because good schema design depends on business rules the model can't see. This module is where you earn the right to make those decisions.

**We build the database first, on purpose.** Everything above it — API, UI — is a projection of this.

---

## 7.1 Why a database and not files?

You could store tasks in a JSON file (you did, in Lab 02). Here's what a database gives you that a file doesn't:

- **Concurrency** — two users writing at once, without corrupting each other
- **Transactions** — several changes that all happen or none do
- **Queries** — "all incomplete tasks due this week, sorted by priority" without loading everything into memory
- **Indexes** — that query stays fast at 10 million rows
- **Constraints** — the database itself refuses to store invalid data
- **Durability** — a power cut mid-write doesn't lose or corrupt your data
- **Concurrent access from multiple processes** — which you'll have the moment you run two copies of your server

The last one alone rules out files for anything real.

### Relational vs the rest

**Relational (Postgres, MySQL, SQLite)** — data in tables with defined columns and enforced relationships. Queried with SQL. Strong consistency guarantees.

**Document (MongoDB, DynamoDB)** — JSON-ish documents, flexible shape.

**Key-value (Redis)** — a giant hash map, usually in memory. For caches, sessions, rate limiting.

The industry has largely converged back on: **use Postgres by default.** It's relational, it handles JSON well when you need flexibility, it's free, it's extremely reliable, and it scales far past where you'll need it. Choose something else only when you can articulate why Postgres won't do.

📖 [PostgreSQL: About](https://www.postgresql.org/about/) · [Just use Postgres](https://www.amazingcto.com/postgres-for-everything/) — an opinionated but correct argument

---

## 7.2 Relational modelling

Tables (relations) have rows (records) and columns (attributes). Rows are linked by **keys**.

- **Primary key** — uniquely identifies a row. Every table needs one.
- **Foreign key** — a column that points at another table's primary key. The database *enforces* that the target exists. This is **referential integrity**, and it's the thing that stops your data from rotting.

### Relationship shapes

**One-to-many** — one user has many tasks. The "many" side holds the foreign key.

```
users                    tasks
─────                    ─────
id (PK)  ◄────────────── user_id (FK)
email                    id (PK)
                         title
```

**Many-to-many** — a task can have many tags; a tag can be on many tasks. Needs a **join table**.

```
tasks          task_tags              tags
─────          ─────────              ────
id (PK) ◄───── task_id (FK)
               tag_id  (FK) ─────────► id (PK)
               PRIMARY KEY (task_id, tag_id)
```

**One-to-one** — usually just extra columns, unless you're splitting rarely-used or sensitive fields.

### Normalization

Store each fact **once**, in one place. If a user changes their email, exactly one row should change.

```sql
-- ❌ denormalized: the email is duplicated on every task
tasks(id, title, user_email, user_name)

-- ✅ normalized: tasks point at a user
users(id, email, name)
tasks(id, title, user_id → users.id)
```

The practical rules ("third normal form", informally): every table describes one kind of thing; every column depends on the primary key and nothing else; no repeating groups (never a `tag1, tag2, tag3` column set — that's a join table).

**When to denormalize:** deliberately, later, for measured performance reasons, accepting the cost of keeping copies in sync. Not on day one.

📖 [Database normalization, explained](https://www.freecodecamp.org/news/database-normalization-1nf-2nf-3nf-table-examples/) · 🎓 [CMU Intro to Database Systems](https://15445.courses.cs.cmu.edu/) — a full, free, genuinely excellent university course if you want real depth

---

## 7.3 Designing the TaskFlow schema

Do this on paper first. Ask: what are the *things*, and how do they relate?

- A **user** has an id, email, name, and timestamps.
- A **list** belongs to one user, has a name.
- A **task** belongs to one list, has a title, description, status, priority, due date, timestamps.
- Later: sessions and accounts (Better Auth creates these in Module 09).

```sql
CREATE TABLE users (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email       text NOT NULL UNIQUE,
  name        text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE lists (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name        text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (user_id, name)
);

CREATE TYPE task_status AS ENUM ('todo', 'in_progress', 'done');

CREATE TABLE tasks (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  list_id      uuid NOT NULL REFERENCES lists(id) ON DELETE CASCADE,
  title        text NOT NULL CHECK (length(trim(title)) > 0),
  description  text,
  status       task_status NOT NULL DEFAULT 'todo',
  priority     smallint NOT NULL DEFAULT 0 CHECK (priority BETWEEN 0 AND 3),
  due_date     timestamptz,
  completed_at timestamptz,
  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now(),

  -- an invariant the database itself will enforce, forever, for every writer
  CONSTRAINT completed_at_matches_status
    CHECK ((status = 'done') = (completed_at IS NOT NULL))
);

CREATE INDEX tasks_list_id_idx ON tasks (list_id);
CREATE INDEX tasks_due_date_idx ON tasks (due_date) WHERE status <> 'done';
```

**Every decision here is deliberate. Study them:**

- **`uuid` vs auto-incrementing `integer` for IDs.** UUIDs can be generated by the client (helps idempotency, Module 06), don't leak how many users you have, and don't collide when merging data. They're bigger and less cache-friendly. Both are defensible; this course uses UUIDs. (See also [UUIDv7](https://datatracker.ietf.org/doc/html/rfc9562#name-uuid-version-7), which sorts chronologically and gets you most of the integer benefits.)
- **`timestamptz`, never `timestamp`.** `timestamptz` stores an absolute moment; `timestamp` stores a wall-clock reading with no timezone, which is almost always a bug waiting for a daylight-saving transition. **Store UTC, convert for display.**
- **`text`, not `varchar(255)`.** In Postgres they perform identically, and `255` is a number inherited from 1990s MySQL that encodes no real business rule. If there's a genuine limit, use a `CHECK`.
- **`NOT NULL` everywhere you can.** Every nullable column is a branch every future reader must handle. Make nullability mean something — here, `completed_at IS NULL` genuinely means "not completed."
- **`ON DELETE CASCADE`** — deleting a user deletes their lists, which deletes their tasks. The alternative (`RESTRICT`) refuses the delete while children exist. Choose consciously; the default silently leaves orphans if you don't declare a foreign key at all.
- **`CHECK` constraints put invariants in the database.** Application code can have bugs; a second service can be written by someone else; a migration script can be careless. The database is the last line of defence, and it never forgets. That last constraint makes "done but no completion time" *impossible to store*.
- **`UNIQUE (user_id, name)`** — a composite uniqueness rule: two users can each have a list called "Work", but one user can't have two.

📖 [PostgreSQL data types](https://www.postgresql.org/docs/current/datatype.html) · [Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)

---

## 7.4 SQL

Connect to your database (from Module 00 / 05):

```bash
docker compose up -d
docker compose exec db psql -U postgres -d taskflow
```

Useful `psql` commands: `\dt` list tables, `\d tasks` describe a table, `\x` toggle expanded output, `\q` quit.

### Reading data

```sql
SELECT id, title, status FROM tasks;

SELECT * FROM tasks
WHERE status = 'todo'
  AND due_date < now() + interval '7 days'
ORDER BY due_date ASC NULLS LAST
LIMIT 20;

-- Aggregation: one row per group
SELECT status, count(*) AS n, max(created_at) AS newest
FROM tasks
GROUP BY status
HAVING count(*) > 5           -- HAVING filters GROUPS; WHERE filters ROWS
ORDER BY n DESC;
```

**The clause evaluation order** — not the order you write them, and knowing it explains most SQL confusion:

```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

This is why you can't use a `SELECT` alias in `WHERE` (the alias doesn't exist yet) but *can* in `ORDER BY`.

### Joins

```sql
-- INNER JOIN: only rows with a match on both sides
SELECT t.title, l.name AS list_name, u.email
FROM tasks t
INNER JOIN lists l ON l.id = t.list_id
INNER JOIN users u ON u.id = l.user_id
WHERE u.email = 'ada@example.com';

-- LEFT JOIN: all rows from the left, NULLs where there's no match
SELECT l.name, count(t.id) AS task_count
FROM lists l
LEFT JOIN tasks t ON t.list_id = l.id      -- lists with zero tasks still appear
GROUP BY l.id, l.name;
```

> ⚠️ In that last query, `count(t.id)` returns 0 for empty lists but `count(*)` would return 1 — because there *is* a row, with NULL columns. Aggregate functions skip NULLs. This is the single most common SQL mistake.

**Join types:** `INNER` (matches only), `LEFT` (all of the left), `RIGHT` (all of the right — just flip the tables and use LEFT), `FULL OUTER` (everything), `CROSS` (every combination — rarely what you want).

### NULL is not a value

`NULL` means *unknown*. So:

```sql
SELECT NULL = NULL;        -- NULL, not true!
SELECT NULL <> 'done';     -- NULL, not true!
SELECT * FROM tasks WHERE due_date <> '2026-01-01';   -- ⚠️ excludes rows where due_date IS NULL
```

Use `IS NULL` / `IS NOT NULL`, and `COALESCE(x, fallback)`. Three-valued logic (true/false/unknown) surprises everyone; expect it.

### Writing data

```sql
INSERT INTO lists (user_id, name) VALUES ('...uuid...', 'Work')
RETURNING *;                             -- Postgres gives you the inserted row back. Very handy.

UPDATE tasks
SET status = 'done', completed_at = now(), updated_at = now()
WHERE id = '...uuid...'
RETURNING *;

DELETE FROM tasks WHERE id = '...uuid...';

-- Upsert: insert, or update if it collides with a unique constraint
INSERT INTO lists (user_id, name) VALUES ('...', 'Work')
ON CONFLICT (user_id, name) DO UPDATE SET name = EXCLUDED.name
RETURNING *;
```

> 🚨 **`UPDATE` and `DELETE` without a `WHERE` clause modify every row in the table.** This is a real way real people have really ruined real production databases. Habit to build now: type the `WHERE` clause *first*, then go back and write the `UPDATE`. And run a `SELECT` with the same `WHERE` before every destructive statement.

### Beyond the basics — worth knowing they exist

```sql
-- Subquery
SELECT * FROM tasks WHERE list_id IN (SELECT id FROM lists WHERE user_id = '...');

-- CTE: name a subquery, read top to bottom instead of inside out
WITH overdue AS (
  SELECT * FROM tasks WHERE due_date < now() AND status <> 'done'
)
SELECT list_id, count(*) FROM overdue GROUP BY list_id;

-- Window function: aggregate WITHOUT collapsing rows
SELECT title, list_id,
       row_number() OVER (PARTITION BY list_id ORDER BY created_at) AS position_in_list,
       count(*)     OVER (PARTITION BY list_id)                     AS tasks_in_list
FROM tasks;
```

Window functions look intimidating and are worth the hour it takes to learn them — rankings, running totals, and "top N per group" all become one query.

---

## 7.5 Indexes

An index is a separate data structure (usually a **B-tree** — Module 03's `O(log n)`) that lets Postgres find rows without scanning the whole table.

```sql
EXPLAIN ANALYZE SELECT * FROM tasks WHERE list_id = '...uuid...';
```

Read the output. `Seq Scan` = reading every row. `Index Scan` = using an index. With 20 rows Postgres will choose a seq scan anyway (it's faster for tiny tables) — so **seed a lot of rows before drawing conclusions**:

```sql
INSERT INTO tasks (list_id, title)
SELECT '...uuid...', 'task ' || i FROM generate_series(1, 200000) i;
```

Now run `EXPLAIN ANALYZE` again, add the index, and run it a third time. Watch the cost and the time.

**Rules of thumb:**
- Index foreign keys (you'll join and filter on them constantly). Postgres does *not* do this automatically.
- Index columns you frequently filter or sort by.
- Composite index column order matters: `(user_id, status)` serves `WHERE user_id = ?` and `WHERE user_id = ? AND status = ?`, but not `WHERE status = ?` alone. Leftmost prefix.
- **Indexes are not free.** Each one costs disk and slows every `INSERT`/`UPDATE`. Don't index everything.
- A function on the column defeats the index: `WHERE lower(email) = 'x'` won't use an index on `email` — you need an index on `lower(email)`.

📖 [Use The Index, Luke!](https://use-the-index-luke.com/) — a free book solely about SQL indexing. The best resource on this topic anywhere.
📖 [Postgres: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) · 🔧 [explain.dalibo.com](https://explain.dalibo.com/) — paste a plan, get a readable visualization

---

## 7.6 Transactions and ACID

A transaction groups statements so they **all** happen or **none** do.

```sql
BEGIN;
  UPDATE tasks SET status = 'done', completed_at = now() WHERE id = '...';
  UPDATE lists SET updated_at = now() WHERE id = '...';
COMMIT;      -- or ROLLBACK to undo everything since BEGIN
```

**ACID:**
- **Atomicity** — all or nothing. A crash between the two updates above leaves neither applied.
- **Consistency** — constraints hold before and after; a transaction can't leave invalid data.
- **Isolation** — concurrent transactions don't see each other's half-finished work.
- **Durability** — once `COMMIT` returns, it survives a power cut.

**Isolation levels** trade correctness against concurrency. Postgres defaults to `READ COMMITTED`, which prevents dirty reads but allows non-repeatable reads. `SERIALIZABLE` behaves as if transactions ran one at a time — safest, but transactions can be aborted and must be retried.

The classic bug this prevents:

```
Two requests both run: read count (5) → add 1 → write 6.
Result: 6. Two increments, one lost. A "lost update" / race condition.
```

Fix it with an atomic statement (`UPDATE t SET count = count + 1`), a row lock (`SELECT ... FOR UPDATE`), or a higher isolation level. **Recognizing that shape** — read, modify, write, concurrently — is the skill.

📖 [Postgres transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html) · 📕 *Designing Data-Intensive Applications* (Kleppmann) ch. 7 — the best explanation of isolation ever written

---

## 7.7 Migrations

Your schema will change. You cannot just edit the database by hand — production has data, and your teammates need the same changes.

A **migration** is a versioned, ordered, committed script that changes the schema. Migrations are code. They live in git. They run automatically on deploy.

```
migrations/
├── 0000_create_users.sql
├── 0001_create_lists_and_tasks.sql
└── 0002_add_priority_to_tasks.sql
```

**Rules:**
- Never edit an applied migration. Write a new one.
- Always test the migration on a copy of production-like data first.
- Prefer **expand/contract** for anything risky: add the new column → backfill it → switch the code to use it → *then*, in a later deploy, drop the old one. Never rename a column in one shot while old code is still running.
- Adding a `NOT NULL` column to a large table can lock it. Add nullable → backfill in batches → add the constraint.

You'll generate migrations from your Drizzle schema in Module 08 — but you now understand what they are and why they're serious.

---

## 7.8 SQL injection — the reason this matters for security

```ts
// ☠️ NEVER. If someone's title is:  '); DROP TABLE tasks; --
db.query(`SELECT * FROM tasks WHERE title = '${userInput}'`);
```

The input is concatenated into the SQL text, so the *data* becomes *code*. This is [the #1 web vulnerability of the last 25 years](https://owasp.org/Top10/A03_2021-Injection/).

```ts
// ✅ Parameterized query — the driver sends SQL and values separately.
// The value can never be interpreted as SQL, no matter what it contains.
db.query("SELECT * FROM tasks WHERE title = $1", [userInput]);
```

Every ORM (including Drizzle) parameterizes by default. **The danger is when you drop to raw SQL and interpolate a string.** If you ever write `` sql`... ${x} ...` `` , stop and check whether your library is parameterizing it or interpolating it.

😄 [xkcd 327: Exploits of a Mom](https://xkcd.com/327/) · 📖 [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

---

## Lab 07 — Live in the database for a day

**A. Create the schema.** Type the DDL from 7.3 into `psql` **by hand** — don't paste. You'll notice things by typing them.

**B. Seed realistic data.** 3 users, ~5 lists each, and 200,000 tasks (use `generate_series`). You need volume for the index work to mean anything.

**C. Write these queries yourself.** No Claude until you've tried each one:

1. All tasks for `ada@example.com`, newest first, 20 at a time.
2. Every list with its task count — *including lists with zero tasks*.
3. Overdue incomplete tasks, grouped by user, with a count.
4. The three most recently completed tasks per list. (Window function.)
5. Users who have created no tasks at all. (Two ways: `LEFT JOIN ... WHERE ... IS NULL`, and `NOT EXISTS`.)
6. Completion rate per user as a percentage, rounded to 1 decimal, with users who have no tasks showing 0 rather than NULL.
7. Move every task from one list to another in a single statement.
8. Delete a user and verify with a `SELECT` that their lists and tasks went too. Explain which constraint did that.

**D. Feel an index.**
```sql
EXPLAIN ANALYZE SELECT * FROM tasks WHERE due_date < now() AND status = 'todo';
-- record the time
CREATE INDEX tasks_due_status_idx ON tasks (status, due_date);
EXPLAIN ANALYZE SELECT * FROM tasks WHERE due_date < now() AND status = 'todo';
-- record it again
```
Write down both numbers and the ratio. Then try `CREATE INDEX ... ON tasks (due_date, status)` instead and see whether column order changed anything for *this* query.

**E. Break a constraint on purpose.** Try each of these and read the exact error:
```sql
INSERT INTO tasks (list_id, title) VALUES ('00000000-0000-0000-0000-000000000000', 'x');  -- bad FK
INSERT INTO tasks (list_id, title) VALUES ('<real-list>', '   ');                          -- CHECK
UPDATE tasks SET status = 'done' WHERE id = '<some-id>';                                   -- the invariant!
INSERT INTO users (email, name) VALUES ('ada@example.com', 'Dup');                          -- UNIQUE
```
That third one is the point of the whole section: **the database refused to let your application create inconsistent data.**

**F. Transactions.**
```sql
BEGIN;
DELETE FROM tasks WHERE status = 'done';
SELECT count(*) FROM tasks;    -- your session sees the deletion...
ROLLBACK;
SELECT count(*) FROM tasks;    -- ...and now it never happened
```
Open a *second* `psql` session and, while the first transaction is open, run the count there. What do you see, and what does that tell you about isolation?

---

## Structured practice (do at least one)

- 🎓 [SQLBolt](https://sqlbolt.com/) — free, interactive, ~2 hours, all 18 lessons. Start here.
- 🎓 [Select Star SQL](https://selectstarsql.com/) — free interactive book on a real dataset. Excellent for `JOIN` and window function intuition.
- 🎓 [PostgreSQL Exercises](https://pgexercises.com/) — free, Postgres-specific, graded. Do the JOIN and aggregation sections.
- 📖 [PostgreSQL Tutorial](https://neon.com/postgresql/tutorial) — thorough written reference
- 🔧 A GUI helps you explore: [DBeaver](https://dbeaver.io/) (free, everything), [TablePlus](https://tableplus.com/), or [pgAdmin](https://www.pgadmin.org/)

---

## Understanding Gate

1. Why is the `completed_at_matches_status` CHECK constraint better than enforcing that rule in your API?
2. `INNER JOIN` vs `LEFT JOIN` — give a real query where using the wrong one silently gives wrong results.
3. Why does `count(*)` differ from `count(t.id)` after a LEFT JOIN?
4. What does `ON DELETE CASCADE` do, and when is it dangerous?
5. What is a lost update, and give two ways to prevent it?
6. When would adding an index make your application *slower*?
7. Why `timestamptz` and not `timestamp`?
8. You need to rename a column on a live table with running traffic. Describe the safe sequence.
9. Explain SQL injection to a non-technical manager in three sentences.

```text
Prompt for Claude Code:
Here's my TaskFlow schema: [paste DDL]. Review it as a database engineer.

1. What constraints am I missing that would let bad data in?
2. Which indexes will I need for these queries: [list yours from Lab 07]?
3. What will break when this table has 50 million rows?
4. Give me three schema-design decisions I made that a senior engineer
   might disagree with, and the argument on each side.

Then quiz me with 5 SQL questions against this schema — show me the
question, let me write the query, then critique mine. Don't give answers first.
```

---

**Next:** [Module 08 — Building the Backend API](08-backend-api.md)
