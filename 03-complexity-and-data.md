# Module 03 — Complexity, Data Structures & Clean Code

**Time:** 4–5 hours
**Prereq:** [Module 02](02-programming-fundamentals.md)

---

## Why this module exists

Your todo app will work fine with 10 tasks. The question professionals ask reflexively is: *what happens at 10,000? At 10 million?* Sometimes the answer is "nothing, don't worry." Sometimes the answer is "the page takes 40 seconds to load and the database falls over." Being able to tell those apart, cheaply, in your head, is what this module teaches.

You will not be writing sorting algorithms. You will be building the instinct to notice a loop inside a loop.

---

## 3.1 Big-O notation

Big-O describes **how the work grows as the input grows** — not how fast something is in milliseconds. It ignores constants and hardware. It answers: "if I give it 10× more data, does it take 10× longer, 100× longer, or barely any longer?"

| Notation | Name | 1,000 items | 1,000,000 items | Typical example |
|---|---|---|---|---|
| `O(1)` | constant | 1 | 1 | `map.get(key)`, array index |
| `O(log n)` | logarithmic | ~10 | ~20 | binary search, B-tree index lookup |
| `O(n)` | linear | 1,000 | 1,000,000 | one loop, `array.find()` |
| `O(n log n)` | linearithmic | ~10,000 | ~20,000,000 | a good sort |
| `O(n²)` | quadratic | 1,000,000 | 1,000,000,000,000 | nested loop over the same data |
| `O(2ⁿ)` | exponential | heat death | heat death | naive recursive fibonacci |

Read that `O(n²)` row again. A million items with a nested loop is a *trillion* operations. That's not "slow," that's "never finishes."

### Reading it off code

```ts
// O(1) — one operation regardless of size
const first = tasks[0];

// O(n) — touches every element once
const done = tasks.filter(t => t.status === "done");

// O(n²) — for each task, scan all tasks. The killer.
const withOwners = tasks.map(task => ({
  ...task,
  owner: users.find(u => u.id === task.userId)   // ← this scan runs n times
}));

// O(n) — same result, using a Map. Build once, then constant-time lookups.
const usersById = new Map(users.map(u => [u.id, u]));
const withOwnersFast = tasks.map(task => ({
  ...task,
  owner: usersById.get(task.userId)
}));
```

That last transformation — **replace a nested search with a hash map** — is genuinely one of the highest-value patterns in everyday programming. You'll use it constantly.

Rules of thumb:
- A loop → `O(n)`. A loop inside a loop over the same data → `O(n²)`.
- Calling `.find()`, `.includes()`, or `.indexOf()` **inside** a loop → almost always `O(n²)`.
- Halving the problem each step → `O(log n)`.
- Sorting → `O(n log n)`, and it's usually fine.

### Space complexity

Same idea for memory. Building a `Map` of a million users is `O(n)` extra memory — usually a great trade for turning `O(n²)` time into `O(n)`. **Time and space trade against each other**, and picking which to spend is a real engineering decision.

📖 [Big-O Cheat Sheet](https://www.bigocheatsheet.com/) — bookmark it
📖 [base-cs series](https://medium.com/basecs) by Vaidehi Joshi — friendly, illustrated CS fundamentals
🎓 [Frontend Masters: Complete Intro to Computer Science](https://frontendmasters.com/courses/computer-science-v2/) (paid) — algorithms & data structures in JavaScript, well taught

---

## 3.2 The complexity that will actually bite you: the database

Here's the thing nobody tells beginners. Your JavaScript is running at maybe 100 million simple operations per second. A single network round-trip to your database is ~1 millisecond on a good day. **One database query costs as much as ~100,000 lines of JavaScript.**

So this innocent code:

```ts
const lists = await db.getLists(userId);            // 1 query
for (const list of lists) {
  list.tasks = await db.getTasksForList(list.id);   // 1 query PER LIST
}
```

With 50 lists that's 51 queries. It's called the **N+1 query problem**, it is the most common performance bug in web applications by a wide margin, and it's invisible in development where you have 3 lists.

Two standard fixes:

```ts
// 1. One query with a JOIN — let the database do the work
const rows = await db.getListsWithTasks(userId);

// 2. Two queries, then stitch in memory with a Map (O(n))
const lists = await db.getLists(userId);
const tasks = await db.getTasksForLists(lists.map(l => l.id));  // WHERE list_id IN (...)
const byList = Map.groupBy(tasks, t => t.listId);
```

**The real lesson:** complexity analysis is about counting *expensive* operations, not all operations. On a server, the expensive ones are network calls and disk reads. Count those.

---

## 3.3 Data structures — the five you need

| Structure | JS | Lookup | Insert | Use it when |
|---|---|---|---|---|
| **Array** | `[]` | `O(n)` by value, `O(1)` by index | `O(1)` push | Ordered list, you iterate it |
| **Hash map** | `Map` / `{}` | `O(1)` by key | `O(1)` | Lookup by id — *the workhorse* |
| **Set** | `Set` | `O(1)` | `O(1)` | Membership tests, deduplication |
| **Tree** | (Postgres B-tree indexes) | `O(log n)` | `O(log n)` | Sorted data, range queries |
| **Queue / Stack** | array + `push`/`shift`/`pop` | — | `O(1)` | Task queues, undo history, the call stack |

```ts
// Set: dedupe and membership, both O(1)
const uniqueTags = [...new Set(tasks.flatMap(t => t.tags))];
const urgent = new Set(["p0", "p1"]);
tasks.filter(t => urgent.has(t.priority));   // O(1) per check, not O(n)

// Map: keyed lookup, and unlike {} it preserves insertion order and allows any key type
const byId = new Map(tasks.map(t => [t.id, t]));
```

**`Map` vs plain object.** Use `Map` when keys are dynamic data (ids, emails) — it's faster for frequent add/delete, keys can be any type, and it has a real `.size`. Use `{}` for fixed, known-at-write-time keys (config, a record of options).

**Trees matter here** because that's what a database index *is* — a B-tree that turns a `O(n)` table scan into an `O(log n)` lookup. Module 07 makes this concrete. Trees also matter because the DOM is one, and your React component hierarchy is one.

---

## 3.4 Clean code, concretely

Abstract advice is useless. Here is the specific advice.

**Name things for what they mean, not what they are.**
```ts
const d = new Date();                  // ❌
const data = await res.json();         // ❌ (data is not a noun that means anything)
const list = tasks.filter(/*...*/);    // ❌
const dueDate = new Date();            // ✅
const overdueTasks = tasks.filter(/*...*/);  // ✅
```
Booleans read as questions: `isLoading`, `hasPermission`, `canEdit`. Functions read as verbs: `createTask`, `formatDueDate`. Arrays are plural.

**Functions should do one thing, at one level of abstraction.**
```ts
// ❌ mixes HTTP, validation, business rules, and SQL in one place
async function handler(req) { /* 80 lines */ }

// ✅ each layer reads like a sentence
async function handler(req) {
  const input = parseCreateTaskBody(req);     // HTTP → domain
  const task = await taskService.create(input); // business logic
  return jsonResponse(201, task);              // domain → HTTP
}
```

**Guard clauses over nesting.**
```ts
// ❌ arrow of doom
if (user) { if (user.isActive) { if (task) { doTheThing(); } } }

// ✅ handle exceptional cases first, then the happy path reads straight down
if (!user) throw new UnauthorizedError();
if (!user.isActive) throw new ForbiddenError("account disabled");
if (!task) throw new NotFoundError("task", id);
doTheThing();
```

**Comments explain *why*, not *what*.**
```ts
// ❌ increment the counter
counter++;

// ✅ Stripe rate-limits at 100 req/s; we batch to stay under it even during backfills.
const BATCH_SIZE = 50;
```
If you need a comment to explain *what* code does, first try renaming things so you don't.

**Magic numbers get names.**
```ts
if (age > 17)                          // ❌ 17 what? why?
const MINIMUM_AGE = 18;
if (age >= MINIMUM_AGE)                // ✅
```

**Delete code aggressively.** Commented-out code is not documentation, it's litter — git remembers it for you. Unused functions are lies about what the system does.

📖 [Refactoring catalog](https://refactoring.com/catalog/) — Martin Fowler's named refactorings. Learn the vocabulary; it makes code review conversations 10× faster.
📖 [Grug Brained Developer](https://grugbrain.dev/) — funny, short, and quietly the best essay on complexity written this decade. Read the whole thing.
📖 [A Philosophy of Software Design](https://web.stanford.edu/~ouster/cgi-bin/book.php) (John Ousterhout) — the best book on this topic. Short. Buy it.

---

## 3.5 Premature optimization

> "Premature optimization is the root of all evil." — Knuth, and the most-abused quote in software.

The full quote adds: *"…yet we should not pass up our opportunities in that critical 3%."* The actual discipline:

1. **Make it work.** Correct and readable first, always.
2. **Measure.** Your intuition about what's slow is usually wrong. Use `console.time()`, the browser Performance tab, `EXPLAIN ANALYZE` in Postgres.
3. **Optimize the measured bottleneck.** Only that one.

But this is *not* a license to write `O(n²)` because "premature optimization." Choosing the right data structure up front costs nothing and isn't optimization — it's design. The rule is: don't micro-optimize unmeasured code; do avoid algorithmically hopeless designs.

---

## Lab 03 — Feel the difference

**A. Make the difference visible.**

Write `bench.ts`. Generate 20,000 fake tasks and 5,000 fake users. Join tasks to users two ways — with `.find()` inside `.map()`, and with a `Map` — and time both with `console.time()`/`console.timeEnd()`.

Then run it again at 50,000 tasks. Note how the two numbers diverge. Write down the ratio. **This is the module in one experiment.**

**B. Predict, then measure.**

For each snippet, write down the Big-O *before* running anything:

```ts
// 1
tasks.forEach(t => console.log(t.title));
// 2
tasks.forEach(a => tasks.forEach(b => { if (a.id !== b.id && a.title === b.title) dupes.push(a) }));
// 3
const seen = new Set<string>();
for (const t of tasks) { if (seen.has(t.title)) dupes.push(t); seen.add(t.title); }
// 4
[...tasks].sort((a, b) => a.title.localeCompare(b.title));
// 5
function count(n: number): number { return n <= 1 ? 1 : count(n-1) + count(n-2) }
```

Then rewrite #2 as #3's approach and confirm you understand *why* they're equivalent in output but not in cost.

**C. Refactor for readability.**

```text
Prompt for Claude Code:
Write me a single 60-line TypeScript function that processes a list of
tasks — filtering, grouping, computing stats, and formatting output. Make it
deliberately BAD: bad names, deep nesting, magic numbers, mixed levels of
abstraction, one accidental O(n²). Don't tell me where the problems are.
```

Now refactor it yourself. Then ask Claude to review your refactor and tell you what you missed.

---

## Understanding Gate

1. Your API endpoint is slow. Would you rather remove an `O(n²)` loop over 100 items, or one database query? Why?
2. What is the N+1 query problem? Give an example from a todo app.
3. When is `O(n²)` totally fine?
4. Why is `array.includes()` inside a loop a smell, and what replaces it?
5. What does a database index have to do with `O(log n)`?
6. Someone says "don't optimize prematurely" about your nested loop over a 1M-row table. Are they right?

```text
Prompt for Claude Code:
Give me 6 short functions. For each, I'll state the time complexity and
whether it has a hidden performance problem. Include at least one that LOOKS
O(n²) but isn't, and one that looks O(n) but is secretly worse (hint: a
built-in method with hidden cost, or a database call in a loop). One at a
time; don't reveal answers early.
```

---

**Next:** [Module 04 — Git and Version Control](04-git.md)
