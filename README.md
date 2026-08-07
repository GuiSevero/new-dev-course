# From Zero to Full-Stack: A Computer Science Onboarding Course

**For:** someone who has never written software professionally, but is comfortable with AI tools like Claude Code.
**Goal:** by the end you will have built and deployed a real todo application — React frontend, HTTP API backend, PostgreSQL database, authentication, tests, CI — **and you will be able to explain every single piece of it.**

---

## Read this part before anything else

Anyone can type *"build me a todo app"* into Claude and get working code in 90 seconds. That is not what this course is for, and if you use it that way you will waste your time.

The scarce skill is no longer *producing* code. It is:

- knowing **what** to build and how the pieces fit
- **reading** code and judging whether it is correct
- **debugging** when the machine does something you did not expect
- **naming** the thing you don't understand precisely enough to get a useful answer

This course uses AI heavily — deliberately. But it uses it as a *tutor and pair*, not as a vending machine. Every module has a **Understanding Gate** at the end. You do not move on until you pass it.

### The three rules

> **Rule 1 — Never keep code you cannot explain out loud.**
> If Claude writes something you don't understand, you have two options: ask it to explain until you do, or delete it and write it yourself. There is no third option. Unexplained code in your project is debt that will come due at 2am.

> **Rule 2 — Predict, then run.**
> Before you run any command or open any page, say out loud what you expect to happen. When reality disagrees with your prediction, *that gap is the entire lesson*. Chase it.

> **Rule 3 — Break it on purpose.**
> After anything works, break it deliberately: delete a line, change a type, unplug the database. Watch how it fails. Systems are best understood at their failure boundaries — and you'll recognize that error message forever.

### How to use Claude Code in this course

You'll be given prompts throughout, in blocks like this:

```text
Prompt for Claude Code:
Explain X to me like I'm a smart person from a different field...
```

Copy them, but *modify them* — the value is in the follow-up questions you ask, not the first answer.

There is a [CLAUDE.template.md](CLAUDE.template.md) in this folder. When you create your project repository, copy it in. It tells Claude Code to act as a teacher instead of a code dispenser (asking you questions, explaining before writing, refusing to hand you finished answers when you're supposed to be learning).

---

## Course map

| # | Module | What you'll be able to do | Est. |
|---|--------|---------------------------|------|
| 00 | [Setting Up Your Machine](00-setup.md) | Install a complete dev environment from scratch; use a terminal | 3–4 h |
| 01 | [How a Computer Actually Runs Your Code](01-how-computers-run-code.md) | Explain the path from source text to running process; stack vs heap | 4 h |
| 02 | [Programming Fundamentals with TypeScript](02-programming-fundamentals.md) | Read and write TS: types, functions, async, modules | 8–10 h |
| 03 | [Complexity, Data Structures & Clean Code](03-complexity-and-data.md) | Reason about Big-O; pick the right data structure; name things well | 4–5 h |
| 04 | [Git and Version Control](04-git.md) | Branch, commit, merge, rebase, resolve conflicts, review a PR | 5–6 h |
| 05 | [Project Organization](05-project-organization.md) | Structure a repo, manage dependencies, handle config and secrets | 3 h |
| 06 | [Client–Server Communication & HTTP](06-client-server-http.md) | Explain DNS→TCP→TLS→HTTP; verbs, status codes, cookies, REST design | 6 h |
| 07 | [Databases and SQL](07-databases-sql.md) | Model data relationally; write real SQL; understand indexes & transactions | 8 h |
| 08 | [Building the Backend API](08-backend-api.md) | Build a typed, validated, layered REST API over Postgres | 10 h |
| 09 | [Authentication & Authorization](09-auth.md) | Explain sessions vs JWT, OAuth 2.0, OIDC; ship real login | 6 h |
| 10 | [Building the Frontend](10-frontend.md) | React + Vite + Tailwind/Radix + TanStack Query + RHF/Zod | 12 h |
| 11 | [Testing](11-testing.md) | Unit, integration and end-to-end tests that actually catch bugs | 6 h |
| 12 | [Containers](12-containers.md) | Explain what a container really is; ship a reproducible image | 7 h |
| 13 | [The Software Development Lifecycle](13-sdlc-teams-deploy.md) | Work like a team does: tickets, PRs, CI/CD, deploy, observe | 6 h |
| 14 | [Capstone & What's Next](14-capstone.md) | Ship it, then extend it without help | 8 h+ |

**Appendices**
- [A — Prompting Claude Code to teach you](appendix-a-prompting.md)
- [B — Glossary](appendix-b-glossary.md)
- [C — Where to go after this course](appendix-c-further-learning.md)

**Total: roughly 100–110 hours.** At 10 h/week that's ~11 weeks. Do not speed-run it. The modules build on each other in a strict order — 06 will not make sense if you skipped 01.

---

## The thing you're building

**TaskFlow** — a multi-user todo app.

```
┌─────────────────────────────────────────────────────────────┐
│  BROWSER                                                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  React app (TypeScript)                                │  │
│  │  Vite · Tailwind · Radix UI · TanStack Query · RHF+Zod │  │
│  └───────────────────────────────────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────┘
                            │  HTTP/JSON over TLS
                            │  GET /api/tasks   POST /api/tasks
                            │  Cookie: session=…
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  SERVER                                                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Hono on Node (TypeScript)                             │  │
│  │  routes → validation → services → repositories         │  │
│  │  Better Auth (sessions, OAuth)                         │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │  SQL via Drizzle ORM          │
│                              ▼                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  PostgreSQL — users, sessions, lists, tasks            │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

You will build it **bottom-up**: database first, then API, then UI. That's deliberate. Most beginners start with the UI, and end up with a pretty app built on a data model that can't support it.

### Features you'll ship

1. Sign up / log in (email+password, and "Sign in with GitHub")
2. Create, read, update, delete tasks
3. Mark tasks done; filter by status
4. Organize tasks into lists
5. Your tasks are yours — nobody else can see or edit them
6. Tests at three levels, running automatically on every push
7. Deployed to a public URL

---

## Before you start

You need:

- A computer (macOS, Windows, or Linux) where you can install software
- ~50 GB free disk space
- A [GitHub account](https://github.com/signup) (free)
- Claude Code ([install guide](https://docs.claude.com/en/docs/claude-code/overview))
- Patience for the specific feeling of *"I have no idea what this error means."* That feeling is the job. Professionals feel it daily; they've just built the habit of calmly reading the message instead of panicking.

Now go to [Module 00](00-setup.md).
