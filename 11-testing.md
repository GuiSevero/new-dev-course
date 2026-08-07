# Module 11 — Testing

**Time:** ~6 hours
**Prereq:** [Module 08](08-backend-api.md), [Module 10](10-frontend.md)

---

## Why this module exists

Tests are not about proving code correct. They're about **making change safe**. A codebase without tests calcifies: every refactor is a gamble, so nobody refactors, so the code rots.

There's also a specific reason tests matter more in the AI era, and it's worth being blunt about it: **when you generate code faster than you can read it, tests are the only thing standing between you and confidently shipping something broken.** A test is an executable statement of what you *meant*. Claude can write the implementation; you must own the intent.

---

## 11.1 The levels

| Level | Tests | Speed | Confidence per test | How many |
|---|---|---|---|---|
| **Unit** | One function, in isolation | ~1ms | Low | Many |
| **Integration** | Several units together — e.g. an API route through to a real database | ~50ms | High | **Most of your effort** |
| **End-to-end (E2E)** | The whole system through a real browser | ~5s | Highest | A handful |

The classic [test pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) says mostly unit tests. Kent C. Dodds' [testing trophy](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications) argues the sweet spot has moved to integration, and for web apps he's right:

> **"The more your tests resemble the way your software is used, the more confidence they can give you."**

A unit test that mocks the database proves your function calls the mock correctly. It does not prove your SQL is right, your constraint holds, or your endpoint returns the right status. Integration tests prove those, and they catch the bugs that actually occur.

**What to unit test:** pure logic with interesting edge cases — date math, permission rules, formatting, complexity-carrying algorithms.
**What to integration test:** every API endpoint, against a real Postgres.
**What to E2E test:** the two or three journeys that, if broken, mean your product is down. Signup→login→create task→complete it. That's it.

---

## 11.2 What makes a good test

**Arrange, Act, Assert:**

```ts
it("marks a task as done and records the completion time", async () => {
  // Arrange
  const list = await createTestList(userId);
  const task = await createTestTask(list.id, { status: "todo" });

  // Act
  const result = await taskService.complete(task.id, userId);

  // Assert
  expect(result.status).toBe("done");
  expect(result.completedAt).toBeInstanceOf(Date);
});
```

**Test behaviour, not implementation.** A test that asserts "the `update` method was called with these arguments" breaks when you refactor and passes when the behaviour is wrong. A test that asserts "after completing, the task reads as done" survives refactoring and fails when the behaviour is wrong. **If your test breaks every time you refactor without changing behaviour, the test is bad — not the refactor.**

**Test names describe behaviour, not functions.**
```ts
it("complete()")                                            // ❌ tells you nothing
it("returns 404 when the task belongs to another user")     // ✅
```

**One reason to fail per test.** Twelve assertions in one test means you fix the first failure and re-run to find the second.

**Tests must be independent and order-independent.** A test that only passes after another test ran is a trap. Each test creates the data it needs and cleans up.

**No logic in tests.** No loops, no conditionals, no clever helpers computing expected values. A test with a bug in it is worse than no test. Write the expected value out literally, even if it's repetitive.

---

## 11.3 Vitest

```bash
npm install -D vitest
```

```jsonc
// package.json
"scripts": { "test": "vitest run", "test:watch": "vitest", "test:ui": "vitest --ui" }
```

### Unit tests

```ts
// apps/api/src/features/tasks/tasks.logic.test.ts
import { describe, it, expect } from "vitest";
import { isOverdue } from "./tasks.logic";

describe("isOverdue", () => {
  it("is true for an incomplete task with a past due date", () => {
    expect(isOverdue({ status: "todo", dueDate: new Date("2020-01-01") })).toBe(true);
  });

  it("is false for a completed task, even if the due date passed", () => {
    expect(isOverdue({ status: "done", dueDate: new Date("2020-01-01") })).toBe(false);
  });

  it("is false when there is no due date", () => {
    expect(isOverdue({ status: "todo", dueDate: null })).toBe(false);
  });
});
```

**Edge cases are where the bugs live.** For every function ask: empty input? null? one element? a huge value? a negative? the boundary exactly (`===` vs `<`)? a timezone boundary? a duplicate?

### Integration tests against a real database

```ts
// apps/api/src/features/tasks/tasks.routes.test.ts
import { describe, it, expect, beforeEach } from "vitest";
import { app } from "../../index";
import { resetDb, createUserWithSession } from "../../test/helpers";

describe("POST /api/tasks", () => {
  beforeEach(async () => { await resetDb(); });   // truncate tables between tests

  it("creates a task and returns 201 with a Location header", async () => {
    const { cookie, listId } = await createUserWithSession();

    // app.request() runs the whole app in-process and returns a real Response.
    // No server, no port, no network — this is the (Request) => Response shape
    // from Module 08.1, called directly.
    const res = await app.request("/api/tasks", {
      method: "POST",
      headers: { "Content-Type": "application/json", cookie },
      body: JSON.stringify({ listId, title: "Write tests" }),
    });

    expect(res.status).toBe(201);
    expect(res.headers.get("location")).toMatch(/^\/api\/tasks\//);
    const body = await res.json();
    expect(body.title).toBe("Write tests");
    expect(body.status).toBe("todo");
  });

  it("rejects an empty title with 400", async () => {
    const { cookie, listId } = await createUserWithSession();
    const res = await app.request("/api/tasks", {
      method: "POST",
      headers: { "Content-Type": "application/json", cookie },
      body: JSON.stringify({ listId, title: "" }),
    });
    expect(res.status).toBe(400);
  });

  it("returns 401 without a session", async () => { /* ... */ });

  it("returns 404 when creating a task in another user's list", async () => { /* ... */ });
});
```

Notice: **no mocking.** A real app, real middleware, real routing, real validation, real Postgres. The only thing missing is the socket — and no bug you care about lives in the socket.

That last test is a *security* test. Write one for every endpoint; they're the tests that would have caught the bugs you found attacking yourself in Lab 09.

**Use a separate test database** (`taskflow_test`), and reset it between tests — either `TRUNCATE ... CASCADE` or wrap each test in a transaction that rolls back. Never point tests at a database you care about.

### React component tests

```bash
npm install -D @testing-library/react @testing-library/user-event @testing-library/jest-dom jsdom
```

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";

it("shows a validation error when the title is empty", async () => {
  const user = userEvent.setup();
  render(<TaskForm listId="1" onDone={() => {}} />);

  await user.click(screen.getByRole("button", { name: /add task/i }));

  expect(await screen.findByRole("alert")).toHaveTextContent(/title is required/i);
});
```

**Query by role and accessible name**, the way a user (or a screen reader) finds things — not by CSS class or test id. This has a wonderful side effect: **if your test can't find the element by its accessible name, neither can a screen reader.** Testing Library's query priority is an accessibility checklist in disguise.

📖 [Vitest guide](https://vitest.dev/guide/) · [Testing Library](https://testing-library.com/docs/react-testing-library/intro/) · [Which query should I use?](https://testing-library.com/docs/queries/about/#priority)

---

## 11.4 Mocking — sparingly

Mocking replaces a real dependency with a fake. Legitimate uses: third-party HTTP APIs, email/SMS sending, payment providers, the system clock, randomness.

```ts
import { vi } from "vitest";

vi.useFakeTimers();
vi.setSystemTime(new Date("2026-08-07T12:00:00Z"));   // now your date logic is deterministic
```

**Don't mock your own database.** A mocked query proves nothing about whether the query is correct. Use a real Postgres in a container — it's fast enough, and it's the only way to test constraints, cascades, and transactions.

> Every mock is an assumption that the real thing behaves as you imagine. Mocks let tests pass while production breaks. That's why the advice is "as few as you can get away with."

For HTTP calls to third parties, [MSW](https://mswjs.io/) intercepts at the network level, so your code under test uses real `fetch` and doesn't know it's being tested.

---

## 11.5 Playwright — end-to-end

```bash
npm init playwright@latest
```

```ts
// e2e/tasks.spec.ts
import { test, expect } from "@playwright/test";

test("a new user can sign up, create a task, and complete it", async ({ page }) => {
  const email = `user-${Date.now()}@example.com`;

  await page.goto("/signup");
  await page.getByLabel("Email").fill(email);
  await page.getByLabel("Password").fill("correct-horse-battery-staple");
  await page.getByRole("button", { name: "Sign up" }).click();

  await expect(page).toHaveURL(/\/lists/);

  await page.getByRole("button", { name: "New task" }).click();
  await page.getByLabel("Title").fill("Buy milk");
  await page.getByRole("button", { name: "Add task" }).click();

  const task = page.getByRole("listitem").filter({ hasText: "Buy milk" });
  await expect(task).toBeVisible();

  await task.getByRole("checkbox").check();
  await expect(task).toHaveAttribute("data-status", "done");
});
```

Playwright runs a real Chromium/Firefox/WebKit, clicks real buttons, and hits your real API and database. It's the closest thing to a user you can automate.

**Playwright's superpowers:**
- **Auto-waiting** — locators retry until the element is actionable. This eliminates the `sleep(500)` flakiness that plagued older tools.
- `npx playwright codegen` — records your clicks and writes the test for you. Great for a first draft; always clean it up afterwards.
- **Trace viewer** (`npx playwright show-trace`) — a time-travel debugger with DOM snapshots, network log, and console for every step of a failed run. When a test fails in CI, this is how you find out why without reproducing locally.

**Keep E2E tests few.** They're slow and they fail for environmental reasons. Two or three critical journeys, not exhaustive coverage.

**Flaky tests are worse than no tests** — a suite that fails randomly trains everyone to ignore failures. When a test flakes: fix it or delete it. Never "just re-run it" as a policy.

📖 [Playwright docs](https://playwright.dev/docs/intro) · [Best practices](https://playwright.dev/docs/best-practices) · [Trace viewer](https://playwright.dev/docs/trace-viewer)

---

## 11.6 Coverage, and TDD

```bash
npx vitest run --coverage
```

Coverage measures which lines *ran* during tests — not whether they were meaningfully checked. 100% coverage with no assertions is 100% meaningless.

Use it to find **untested areas**, not as a target. Chasing a coverage number produces tests written to satisfy the number. 70–80% on code that matters, with the critical paths at 100%, beats 95% of padding.

**Test-Driven Development** — red, green, refactor:

1. **Red:** write a failing test for behaviour that doesn't exist.
2. **Green:** write the minimum code to pass it.
3. **Refactor:** clean it up; tests keep you honest.

TDD's real benefit isn't testing — it's **design pressure**. Code that's hard to test is usually badly coupled, and TDD makes you feel that before you've built on top of it. Try it for at least one feature in the lab; it's a different way of thinking, and you should experience it before deciding how much of it you want.

📖 [Kent Beck: Test-Driven Development by Example](https://www.oreilly.com/library/view/test-driven-development/0321146530/) — the original, still the best
📖 [Martin Fowler on testing](https://martinfowler.com/testing/)

---

## Lab 11 — Test TaskFlow

1. **Set up Vitest** in both `apps/api` and `apps/web`. Get one trivial test passing before writing real ones.
2. **A test database.** `taskflow_test` in a second container or a second database in the same one. Write a `resetDb()` helper and prove it runs between tests.
3. **Unit tests** for your pure logic: `isOverdue`, the summary/percentage calculation (test the empty-list case — you did handle the `NaN`, right?), and any date formatting. Cover the edges.
4. **Integration tests for every endpoint.** For each one, at minimum:
   - the happy path, with the right status code and response shape
   - each validation failure → 400
   - unauthenticated → 401
   - another user's resource → 404
   - a nonexistent id → 404
5. **A regression test.** Take one bug you actually hit while building — write a test that fails against the old broken behaviour, then confirm it passes now. **This is the highest-value test you will write**, and it's the habit to keep: every bug fix ships with a test.
6. **Component tests** for `TaskForm`: validation errors appear, submit is disabled while submitting, success calls the callback. Query by role only.
7. **One Playwright E2E**: signup → create list → create task → complete it → reload → still complete. Then run `npx playwright show-trace` on a deliberately broken version and explore the trace viewer.
8. **Try TDD once.** Pick an unbuilt feature — say "archive a task" — and write the failing tests first, at both the service and route level, before any implementation.

**Then break it deliberately:** comment out the ownership check in one endpoint. Does a test fail? If not, your test suite has a hole exactly where it matters most. Write that test.

```text
Prompt for Claude Code:
Here's my TaskFlow API endpoint and its tests: [paste both].

1. What behaviour is NOT covered? List specific missing cases.
2. Which of my tests are testing implementation rather than behaviour, and
   would break during a harmless refactor?
3. Which are so trivial they cost more to maintain than they're worth?
4. Give me 5 edge-case inputs I haven't considered.
5. If you deliberately introduced a subtle bug in this endpoint, which of my
   tests would catch it? Which bug would slip through?

Then: introduce one subtle bug into my implementation, show me the modified
code without saying what you changed, and let me find it by reading the code
and running the tests.
```

---

## Understanding Gate

1. Why do integration tests give more confidence per test than unit tests?
2. Why is mocking your own database usually a bad idea?
3. What's wrong with a test that asserts a repository method was called?
4. What does 100% coverage prove? What doesn't it prove?
5. Why is a flaky test worse than no test?
6. What's the design benefit of TDD, separate from the tests it produces?
7. Why does Testing Library push you to query by role?
8. You have 30 minutes and no tests. Which one do you write first, and why?

---

**Next:** [Module 12 — Containers](12-containers.md)
