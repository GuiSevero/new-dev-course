# Module 02 — Programming Fundamentals with TypeScript

**Time:** 8–10 hours. The biggest module. Don't rush it.
**Prereq:** [Module 01](01-how-computers-run-code.md)

---

## Why this module exists

You need enough fluency to **read** code confidently and **write** it slowly. That's the bar. You are not trying to become a fast JavaScript author — Claude is faster than you'll ever be. You are trying to become someone who can look at 40 lines of TypeScript and say "that's wrong, and here's why."

Everything in this course is TypeScript, front to back. One language for the browser, the server, the tests, and the build scripts.

---

## 2.1 Set up a scratchpad

```bash
mkdir -p ~/projects/ts-practice && cd ~/projects/ts-practice
npm init -y
npm install -D typescript tsx @types/node
npx tsc --init
```

What just happened:
- `npm init -y` created `package.json` — the manifest describing this project.
- `-D` installs as a **dev dependency**: needed to build/test, not to run in production.
- `tsx` runs a `.ts` file directly (compile + execute in one step).
- `npx tsc --init` created `tsconfig.json` — the TypeScript compiler's configuration.

Open `tsconfig.json` and make sure you have `"strict": true`. Non-negotiable. Strict mode is what makes TypeScript worth using; without it you get the syntax and none of the safety.

Run a file with:
```bash
npx tsx play.ts
```

---

## 2.2 Values, types, and variables

```ts
// let  = can be reassigned.  const = cannot.  Default to const.
const name: string = "Ada";
let count: number = 0;
const isDone: boolean = false;
const nothing: null = null;          // "deliberately empty"
const missing: undefined = undefined; // "never assigned"

// Usually you omit the annotation — TS infers it. Prefer inference:
const city = "Lisbon";  // TS knows: string
```

**`null` vs `undefined`:** `undefined` means "no value was ever put here"; `null` means "someone explicitly put nothing here." In practice: pick one for your own code (this course uses `null` in the database, `undefined` in TypeScript), and always handle both at boundaries.

**The billion-dollar mistake.** Tony Hoare, who invented null references in 1965, [called them exactly that](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/). `strict: true` turns TypeScript into a machine for preventing them: it forces you to prove a value isn't null before you use it.

```ts
function greet(user: { name: string } | null) {
  // console.log(user.name);  // ❌ 'user' is possibly 'null'
  if (!user) return "hello, stranger";
  return `hello, ${user.name}`;   // ✅ TS narrowed the type
}
```

That's **narrowing** — TypeScript follows your control flow and knows that after the early return, `user` can't be null. Learning to work *with* narrowing rather than fighting it (by casting or using `!`) is most of what "getting good at TypeScript" means.

---

## 2.3 Shaping data: objects, arrays, interfaces, unions

```ts
type TaskStatus = "todo" | "in_progress" | "done";   // union of literals

interface Task {
  id: string;
  title: string;
  status: TaskStatus;
  dueDate?: Date;          // ? = optional, i.e. Date | undefined
  readonly createdAt: Date; // can't be reassigned after construction
}

const tasks: Task[] = [
  { id: "1", title: "Learn TypeScript", status: "in_progress", createdAt: new Date() },
];
```

Why `"todo" | "done"` instead of `string`? Because now `status: "dnoe"` is a compile error, and your editor autocompletes the valid values. **Make illegal states unrepresentable** — this is one of the highest-leverage ideas in typed programming.

**`type` vs `interface`:** roughly interchangeable for object shapes. `interface` can be re-opened and extended; `type` can express unions, intersections and computed types. Team convention matters more than the choice. This course uses `type` unless extending.

### Useful type operations

```ts
type TaskId = Task["id"];                    // index access → string
type NewTask = Omit<Task, "id" | "createdAt">;  // everything except those
type TaskPatch = Partial<NewTask>;           // all fields optional
type Summary = Pick<Task, "id" | "title">;   // just those two
type Keys = keyof Task;                      // "id" | "title" | "status" | ...
```

These matter enormously in real code: your "create a task" payload is `Omit<Task, "id">`, your "update a task" payload is `Partial<...>`, and you get all three from **one** source of truth. When the shape changes, everything downstream fails to compile — which is the whole point.

📖 [TypeScript Handbook — Everyday Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html) · [Utility Types reference](https://www.typescriptlang.org/docs/handbook/utility-types.html)

---

## 2.4 Functions

```ts
// Named function
function add(a: number, b: number): number {
  return a + b;
}

// Arrow function — same thing, different syntax, different `this` binding
const multiply = (a: number, b: number): number => a * b;

// Default and rest parameters
const greet = (name: string, greeting = "Hello") => `${greeting}, ${name}`;
const sum = (...nums: number[]) => nums.reduce((acc, n) => acc + n, 0);
```

**Functions are values.** You can pass them, return them, store them in arrays. This is the single most important idea in JavaScript.

```ts
const twice = (fn: (n: number) => number, x: number) => fn(fn(x));
twice(n => n + 3, 10);   // 16
```

### Closures

A function "closes over" the variables in scope where it was *defined*, and keeps them alive.

```ts
function makeCounter() {
  let count = 0;                 // lives on the heap, kept alive by the closure
  return () => ++count;
}
const next = makeCounter();
next(); // 1
next(); // 2   ← the same `count`, still alive after makeCounter() returned
```

Closures are how React hooks work, how middleware works, how nearly every callback-based API works. They're also a classic source of stale-value bugs (Module 10 revisits this as React's "stale closure" problem).

### Array methods you must know cold

```ts
const nums = [1, 2, 3, 4, 5];

nums.map(n => n * 2);              // [2,4,6,8,10]   transform each
nums.filter(n => n % 2 === 0);     // [2,4]          keep some
nums.find(n => n > 3);             // 4              first match, or undefined
nums.reduce((a, n) => a + n, 0);   // 15             fold into one value
nums.some(n => n > 4);             // true
nums.every(n => n > 0);            // true
nums.sort((a, b) => b - a);        // ⚠️ MUTATES the original array
```

`map`/`filter`/`reduce` return **new** arrays; `sort`, `push`, `splice`, `reverse` **mutate**. That distinction is exactly the value-vs-reference lesson from Module 01, and in React it's the difference between your UI updating and silently not updating.

---

## 2.5 Asynchronous code

The single most important practical skill in this module.

```ts
// A Promise represents a value that will exist later — or an error.
async function fetchTask(id: string): Promise<Task> {
  const res = await fetch(`https://api.example.com/tasks/${id}`);
  if (!res.ok) throw new Error(`Request failed: ${res.status}`);
  return res.json() as Promise<Task>;
}
```

Rules that will save you hours:

```ts
// ❌ Forgetting await — you get a Promise object, not the data
const task = fetchTask("1");
console.log(task.title);   // undefined

// ❌ Sequential when it could be parallel — 300ms instead of 100ms
const a = await fetchTask("1");
const b = await fetchTask("2");
const c = await fetchTask("3");

// ✅ Parallel — all three in flight at once
const [a2, b2, c2] = await Promise.all([fetchTask("1"), fetchTask("2"), fetchTask("3")]);

// ✅ Parallel, tolerating individual failures
const results = await Promise.allSettled([fetchTask("1"), fetchTask("2")]);
```

**Error handling.** An `await`ed promise that rejects throws — so `try/catch` works:

```ts
try {
  const task = await fetchTask("1");
} catch (err) {
  // `err` is typed `unknown` in strict mode — because anyone can throw anything
  if (err instanceof Error) console.error(err.message);
  else console.error("unknown error", err);
}
```

An **unhandled promise rejection** — a promise that fails with nobody listening — will crash a Node process. Every `async` function you call must either be `await`ed or have a `.catch()`.

📖 [MDN: Asynchronous JavaScript](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Async_JS) · [javascript.info: Promises, async/await](https://javascript.info/async)

---

## 2.6 Modules

How code in one file becomes available in another. This course uses **ES Modules** (ESM) everywhere.

```ts
// src/tasks.ts
export type Task = { id: string; title: string };
export function createTask(title: string): Task {
  return { id: crypto.randomUUID(), title };
}
export default createTask;   // at most one default per module
```

```ts
// src/main.ts
import createTask, { type Task } from "./tasks.js";   // note the .js extension in ESM
import { readFile } from "node:fs/promises";          // node: prefix = built-in module
import { z } from "zod";                              // bare specifier = from node_modules
```

Three kinds of import path — relative (`./`), built-in (`node:`), and package (bare). Recognizing which is which at a glance tells you instantly where code comes from.

> You will meet **CommonJS** (`require()`/`module.exports`) in older code and older tutorials. It's the previous system. Know it exists; write ESM.

---

## 2.7 Errors, and how to fail well

```ts
// Define error types that carry meaning
class NotFoundError extends Error {
  constructor(public readonly resource: string, public readonly id: string) {
    super(`${resource} ${id} not found`);
    this.name = "NotFoundError";
  }
}

// Then callers can branch on the KIND of failure, not on string matching
try { /* ... */ }
catch (e) {
  if (e instanceof NotFoundError) return respond404();
  throw e;    // ← re-throw what you can't handle. Never swallow errors.
}
```

Two anti-patterns to never write:

```ts
try { risky(); } catch (e) {}                    // ❌ silence — the bug still happened
try { risky(); } catch (e) { console.log(e); }   // ❌ logged and then continued as if fine
```

**Fail fast, fail loud, fail with context.** An error message should say what was being attempted, with which inputs, and why it failed. "Error" is not a message.

---

## 2.8 Programming principles worth internalizing now

These are not rules, they're *pressures*. Every one of them can be over-applied.

| Principle | What it means | The failure mode of overdoing it |
|---|---|---|
| **DRY** — Don't Repeat Yourself | Every piece of *knowledge* has one home | Coupling unrelated things because they looked similar once |
| **YAGNI** — You Aren't Gonna Need It | Don't build for imagined futures | Under-designing something you obviously will need |
| **KISS** | Simplest thing that works | Simple-looking code that hides complexity elsewhere |
| **Single Responsibility** | A module has one reason to change | 300 four-line files |
| **Separation of concerns** | HTTP handling ≠ business logic ≠ database access | Ceremony for a 50-line app |
| **Composition over inheritance** | Build from small pieces; prefer functions | — (this one's hard to overdo) |
| **Pure functions** | Same input → same output, no side effects | Not everything can be pure; I/O exists |

The one that pays off immediately: **naming**. `const d = new Date()` vs `const dueDate = new Date()`. Code is read far more often than written — including by an AI trying to help you, which reads your names as intent.

📖 [Martin Fowler on Refactoring](https://refactoring.com/) · [SOLID, explained without enterprise Java](https://www.digitalocean.com/community/conceptual-articles/s-o-l-i-d-the-first-five-principles-of-object-oriented-design)

---

## 2.9 Debugging — the actual skill

Faster than asking an AI, and it's what you'll do when the AI is wrong.

1. **Read the error message.** All of it. Out loud if needed. Most beginners' errors are literally described in the first line.
2. **Read the stack trace top-down.** Find the first line that's *your* file, not a library's. That's usually where to look.
3. **Reproduce it reliably.** A bug you can trigger on demand is nearly solved.
4. **Bisect.** Comment out half. Still broken? The problem's in the other half. Repeat. This finds a bug in a 1000-line file in ~10 steps.
5. **Check your assumptions with a print.** `console.log({ userId, task })` — log an object, not a bare value, so you see the names.
6. **Use a real debugger.** In VS Code, click the gutter to set a breakpoint, then Run → Start Debugging. Step through line by line and inspect every variable. It feels slow; it is much faster.

📖 [VS Code debugging docs](https://code.visualstudio.com/docs/editor/debugging) · [Node.js debugging guide](https://nodejs.org/en/learn/getting-started/debugging)

---

## Lab 02 — Build a todo engine with no UI, no server, no database

This is the core of TaskFlow, in memory, in one file. When you get to Module 08 you'll recognize this logic — it'll just be talking to a database instead of an array.

Create `todo.ts`:

```ts
export type TaskStatus = "todo" | "done";

export type Task = {
  id: string;
  title: string;
  status: TaskStatus;
  createdAt: Date;
  completedAt: Date | null;
};

export type CreateTaskInput = { title: string };
```

Now implement, **by hand, without Claude writing them for you**:

1. `createTask(input: CreateTaskInput): Task` — generate an id with `crypto.randomUUID()`. Reject an empty or whitespace-only title by throwing a meaningful error.
2. `completeTask(task: Task): Task` — return a **new** task with `status: "done"` and `completedAt` set. Do not mutate the input. (Why not? Answer that in a comment.)
3. `filterByStatus(tasks: Task[], status: TaskStatus): Task[]`
4. `sortByCreated(tasks: Task[], dir: "asc" | "desc"): Task[]` — must not mutate the input array. Find out what `toSorted()` is.
5. `summarize(tasks: Task[]): { total: number; done: number; percentComplete: number }` — use `reduce`. Handle the empty array without producing `NaN`.
6. `groupByStatus(tasks: Task[]): Record<TaskStatus, Task[]>`
7. An async one: `saveTasks(tasks: Task[], path: string): Promise<void>` and `loadTasks(path: string): Promise<Task[]>`, using `node:fs/promises`. **Watch what happens to your `Date` objects through `JSON.stringify` and back** — this is a genuinely important lesson about serialization, and you will hit it again in Module 06.

Then write a tiny CLI at the bottom that adds a few tasks, completes one, and prints the summary. Run with `npx tsx todo.ts`.

**Now the important part:**

```text
Prompt for Claude Code:
Here is my todo.ts [paste it]. Do NOT rewrite it. Instead:
1. Review it as a senior engineer would in a code review — comment by comment,
   pointing at specific lines.
2. Tell me which functions accidentally mutate their inputs, if any.
3. Point out where my types are weaker than they could be.
4. Give me 3 edge-case inputs that would break my code, but don't tell me how
   to fix them — I'll try first.
```

---

## Structured courses (pick at least one, do it alongside this module)

- 🎓 [javascript.info](https://javascript.info/) — free, thorough, the best written JS reference-course there is. Read Part 1, chapters 2–6.
- 🎓 [Total TypeScript: Beginner's TypeScript Tutorial](https://www.totaltypescript.com/tutorials/beginners-typescript) — free, interactive, 18 exercises. Do all of them.
- 🎓 [Eloquent JavaScript](https://eloquentjavascript.net/) — free book. Chapters 1–5 and 11 (async).
- 🎓 [CS50x](https://cs50.harvard.edu/x/) (Harvard, free) — if you want real depth on the *computer science* side. It's in C, which sounds irrelevant but is exactly why it teaches memory and pointers properly. Weeks 0–5 are transformative. Highly recommended if you have the time.
- 🎓 [Execute Program](https://www.executeprogram.com/) — paid, spaced-repetition drills for JS/TS. Unusually effective if you struggle to retain syntax.

---

## Understanding Gate

1. What's the difference between `const` and `readonly` and `Object.freeze`?
2. Why does `map` "work" in React state updates but `push` doesn't?
3. Explain a closure to a non-programmer.
4. What is the type of `err` in a `catch` block under `strict`, and why?
5. When would `Promise.all` be wrong, and what would you use instead?
6. You have `type Task = {...}` with 8 fields. Your create endpoint shouldn't accept `id` or `createdAt`. How do you express that without writing a second type by hand?
7. Give an example of DRY making code *worse*.

```text
Prompt for Claude Code:
Quiz me on TypeScript fundamentals. Give me 8 short code snippets that each
have exactly one bug — type errors, mutation bugs, async mistakes, or
closure problems. Show one at a time. I'll say what's wrong and how to fix
it. Score me at the end and tell me which category I'm weakest in.
Do not show me the answers before I guess.
```

---

**Next:** [Module 03 — Complexity, Data Structures & Clean Code](03-complexity-and-data.md)
