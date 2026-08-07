# Module 13 — The Software Development Lifecycle

**Time:** ~6 hours
**Prereq:** [Module 04](04-git.md), [Module 11](11-testing.md), [Module 12](12-containers.md)

---

## Why this module exists

Everything so far has been about making software work. This module is about how software gets made *by teams, repeatedly, without breaking* — the part that turns a working project into a job.

If you join a company, this is the module you'll live inside from day one. It's also the part nobody teaches, because it isn't code.

---

## 13.1 The cycle

```
   ┌──────────────────────────────────────────────────────────┐
   │                                                          │
   ▼                                                          │
 Plan  →  Code  →  Review  →  Test  →  Deploy  →  Observe  ───┘
```

The single most important property of this loop: **make it short.** A team that deploys 20 times a day has small changes, small blast radius, and fast recovery. A team that deploys once a quarter has enormous risky releases and can't tell which of 400 changes broke things.

This is measured. The [DORA research program](https://dora.dev/) found four metrics that predict both delivery performance *and* organizational outcomes:

- **Deployment frequency** — how often you ship
- **Lead time for changes** — commit → production
- **Change failure rate** — % of deploys causing a problem
- **Time to restore service** — how fast you recover

Counterintuitively, the teams that deploy *most often* are also the *most stable*. Small changes are easier to review, easier to test, and easier to roll back.

📖 [DORA: capabilities](https://dora.dev/capabilities/) · 📕 *Accelerate* (Forsgren, Humble, Kim) — the research behind it

---

## 13.2 How teams actually work

### Roles you'll interact with

| Role | Owns | Asks you |
|---|---|---|
| **Product Manager** | What to build and why; priorities | "How long? What are the tradeoffs?" |
| **Designer** | User flows, visual design, interaction | "Can we do this? What's expensive?" |
| **Engineer** | How it's built | — |
| **Tech Lead / Staff Eng** | Technical direction, architecture, unblocking | "Why this approach?" |
| **Engineering Manager** | The people, growth, delivery | "What's in your way?" |
| **QA** (sometimes) | Test strategy, exploratory testing | "How do I break this?" |
| **DevOps / SRE / Platform** | Infra, CI/CD, reliability | "What does this need to run?" |

### Agile, honestly

**Scrum:** fixed-length sprints (usually 2 weeks), a groomed backlog, sprint planning, daily standup, review/demo, retrospective. Prescriptive; good scaffolding for new teams; often becomes ritual.

**Kanban:** continuous flow, a board with work-in-progress limits, pull work when you have capacity. Better when work is unpredictable (support, platform teams).

Most real teams run something in between and call it Scrum.

**The ceremonies, and what they're actually for:**

- **Standup** (15 min, daily) — *not* a status report to a manager. It's for surfacing **blockers** and coordination. "Yesterday I did X, today Y, I'm blocked on Z." If it takes 40 minutes, it's broken.
- **Planning** — decide what the team commits to. Estimates are in relative points, not hours, because humans are bad at absolute time estimates and slightly less bad at relative ones.
- **Retrospective** — the team inspects its own process and changes one thing. **The most valuable ceremony and the first one teams skip when busy**, which is exactly backwards.
- **Refinement / grooming** — break big vague things into small clear things before they're picked up.
- **Demo / review** — show working software to stakeholders.

📖 [Atlassian Agile Coach](https://www.atlassian.com/agile) — genuinely good free material on all of this
📖 [Shape Up](https://basecamp.com/shapeup) (Basecamp, free) — a well-argued alternative to sprints. Read it to see that Scrum isn't the only option.

---

## 13.3 Jira and issue tracking

A **ticket** is the unit of work: an atomic, describable, verifiable change.

Hierarchy: **Epic** (a big goal, weeks) → **Story** (user-facing value, days) → **Task/Sub-task** (an implementation step, hours). Plus **Bug** and **Spike** (a timeboxed investigation, where the deliverable is knowledge, not code).

A story is written from the user's perspective:

> **As a** logged-in user, **I want to** filter my tasks by status **so that** I can focus on what's still outstanding.
>
> **Acceptance criteria**
> - [ ] Filter control offers: All, To do, In progress, Done
> - [ ] The selected filter is reflected in the URL and survives a refresh
> - [ ] Empty results show "No tasks match this filter"
> - [ ] Filtering is keyboard-accessible
> - [ ] Filter state persists when navigating back from a task

**Acceptance criteria are the contract.** They're what "done" means, what QA verifies, and what your tests assert. A ticket without them will be built wrong and everyone will be annoyed. If you're handed one, *write the criteria yourself and confirm them* — that habit alone will make you popular.

A typical workflow: `Backlog → To Do → In Progress → In Review → QA → Done`. **Keep your tickets moving and up to date.** In a remote team, the board is how everyone knows what's happening without asking you.

**Definition of Done** — a team-wide standard, e.g.: code reviewed, tests written and passing, CI green, docs updated, deployed to staging, acceptance criteria verified. Without one, "done" means five different things to five people.

📖 [Jira: Getting started](https://www.atlassian.com/software/jira/guides) · [Writing user stories](https://www.atlassian.com/agile/project-management/user-stories) · [Linear's method](https://linear.app/method) — a sharper, more opinionated take on the same problems

---

## 13.4 Code review

The highest-leverage activity in software teams. It catches bugs, spreads knowledge, and enforces standards — in that order of what people expect, and roughly the reverse order of actual value.

### As the author

- **Keep PRs small.** Under ~400 lines. Review quality [falls off a cliff](https://smallbusinessprogramming.com/optimal-pull-request-size/) beyond that; a 2,000-line PR gets "LGTM" and nothing else.
- **Write a real description:** what changed, why, how to test it, what you're unsure about, screenshots for UI.
- **Review your own diff first.** You'll catch the debug logs and the accidentally committed file.
- **Make CI green before requesting review.** Don't spend a colleague's attention on something a machine would have caught.
- **Don't take it personally.** Comments are about the code. This is genuinely a skill and it takes a while.

### As the reviewer

- **Be fast.** A PR sitting for two days blocks a person and goes stale. Review within a few hours.
- **Ask questions instead of issuing orders.** "What happens if `tasks` is empty here?" beats "this is wrong."
- **Distinguish severity.** Prefix with `blocking:`, `suggestion:`, or `nit:` so the author knows what must change versus what's taste.
- **Praise good things.** Genuinely — it's how standards spread.
- **Look for:** correctness, edge cases, security (authorization! injection! secrets!), N+1 queries, missing tests, naming, and whether it fits the existing architecture.
- **Don't nitpick formatting.** That's Prettier's job. If you're arguing about formatting, fix the tooling.

📖 [Google's Code Review Developer Guide](https://google.github.io/eng-practices/review/) — the best public writing on this; short and applicable anywhere
📖 [Conventional Comments](https://conventionalcomments.org/)

> **On AI-generated code in review:** you are accountable for every line you submit, regardless of who typed it. "Claude wrote it" is not a defence, and reviewers can tell — AI-generated code that the author doesn't understand has a distinct smell (over-general abstractions, unused parameters, defensive checks for impossible conditions, comments restating the code). Read and prune before you push.

---

## 13.5 Continuous Integration

**CI** runs your checks automatically on every push. The point is *fast feedback* — knowing within minutes, not after a reviewer wastes an hour.

You already built the containerized version of this in [Module 12](12-containers.md#127-ci-in-containers) — build the image, run the suite inside it, push only from `main`. Here's the simpler source-based form, because it's what you'll see in most repos and it isolates the concepts:

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push: { branches: [main] }
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: taskflow_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready --health-interval 10s
          --health-timeout 5s --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }

      - run: npm ci                       # ← from the lockfile, not package.json (Module 05)
      - run: npm run typecheck
      - run: npm run lint
      - run: npm run test
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/taskflow_test
      - run: npm run build
```

Then, in GitHub: Settings → Branches → add a rule requiring these checks to pass and one approving review before merging to `main`. **Now `main` can't break**, which is the entire point.

**Continuous Delivery** — every change that passes CI is *deployable*. **Continuous Deployment** — it deploys automatically. Both need enough test confidence to trust the pipeline.

> Which style should you keep? The source-based pipeline is faster and simpler; the containerized one tests the actual artifact you deploy. Most teams run both: fast source checks on every push for feedback, and an image build + smoke test on `main`. Decide deliberately rather than by default.

📖 [GitHub Actions docs](https://docs.github.com/en/actions) · [Understanding workflows](https://docs.github.com/en/actions/using-workflows/about-workflows)

---

## 13.6 Branching strategy

**Trunk-Based Development** — short-lived branches (hours to two days) merged into `main` constantly. This is what high-performing teams do, and it's what pairs with CI.

**GitFlow** — long-lived `develop`, `release/*`, `hotfix/*` branches. Designed for versioned desktop software with scheduled releases. Widely used, widely inappropriate for web apps. [Its author now says so](https://nvie.com/posts/a-successful-git-branching-model/#note-of-reflection-march-5-2020).

The problem with long branches is arithmetic: merge pain grows with the square of divergence. Merge daily.

**Feature flags** decouple *deploying* from *releasing*: ship the code turned off, enable it for 1% of users, then 10%, then everyone. This is how you merge incomplete work safely and how you roll back a feature without rolling back a deploy.

📖 [Trunk Based Development](https://trunkbaseddevelopment.com/) · [Martin Fowler on feature toggles](https://martinfowler.com/articles/feature-toggles.html)

---

## 13.7 Environments and deployment

| Environment | Purpose |
|---|---|
| **Local** | Your machine |
| **Preview / PR** | An ephemeral deploy per pull request — reviewers can click the actual thing |
| **Staging** | Production-like, with fake data. Final check. |
| **Production** | Real users, real data |

The same **build artifact** should move through these, configured only by environment variables ([12-Factor](https://12factor.net/), Module 05). Rebuilding per environment means you never tested what you shipped.

That sentence used to be abstract. It isn't any more: **the artifact is the image you pushed in [Module 12](12-containers.md)**, identified by a digest. `ghcr.io/you/taskflow-api@sha256:9f2a…` in staging and in production is the same bytes — provably, not by convention.

### Deploying TaskFlow

- **Frontend** — a folder of static files. [Vercel](https://vercel.com/docs), [Netlify](https://docs.netlify.com/), or [Cloudflare Pages](https://developers.cloudflare.com/pages/). Connect the repo, done.
- **API** — a long-running process. Point [Fly.io](https://fly.io/docs/), [Render](https://render.com/docs), [Railway](https://docs.railway.com/) or [Cloud Run](https://cloud.google.com/run/docs) at your image tag and pass the env vars. They'll also build from a `Dockerfile` in your repo, but deploying the **already-tested image** is stronger: nothing is rebuilt between the test and the deploy.
- **Database** — managed, always. [Neon](https://neon.com/docs), [Supabase](https://supabase.com/docs), or your host's Postgres. **Never run your own production database while learning** — backups, failover, and upgrades are a specialist job.

> **This is also what makes rollback fast.** Rolling back is "deploy the previous digest" — seconds, no rebuild, no git revert, no waiting for CI. That's the difference between a 30-second incident and a 30-minute one, and it's the [DORA](https://dora.dev/) *time to restore service* metric in concrete form.

**Deployment strategies:**
- **Rolling** — replace instances gradually
- **Blue-green** — stand up the new version alongside, switch traffic, keep the old one warm for instant rollback
- **Canary** — 5% of traffic to the new version, watch the metrics, proceed or abort

**Migrations in a deploy** are the sharp edge. During a rolling deploy, old and new code run *simultaneously* against one database. So migrations must be **backwards-compatible**: add the column, deploy code that writes both old and new, backfill, deploy code that reads new, *then* drop the old column in a later deploy. Expand/contract (Module 07). Getting impatient here is how you cause an outage.

**Always know how to roll back.** Before deploying, answer: how do I undo this in under five minutes? If the answer involves a migration you can't reverse, you need a different plan.

---

## 13.8 Observability

Once it's live, you can't attach a debugger. You need the system to tell you what it's doing.

**The three pillars:**
- **Logs** — discrete events. Structured JSON, with a request id correlating all lines from one request.
- **Metrics** — numbers over time: request rate, error rate, p50/p95/p99 latency, CPU, DB connections. (Use percentiles, not averages — an average hides the 1% of users having a terrible time.)
- **Traces** — one request's journey across services, with timing per hop. [OpenTelemetry](https://opentelemetry.io/) is the standard.

**Error tracking** is the highest-value thing to add first: [Sentry](https://docs.sentry.io/) captures exceptions with stack traces, request context, and which users were affected. Add it to both frontend and backend in an afternoon.

**Alert on symptoms, not causes.** "Error rate above 2% for 5 minutes" and "p95 latency over 2s" are actionable. "CPU is at 80%" might be completely fine. Every alert that isn't actionable trains people to ignore alerts.

**Postmortems.** When something breaks, write down: what happened, the timeline, the impact, the root cause, and what will prevent it next time. **Blameless** — the goal is a better system, not a guilty person. A culture where people hide mistakes is a culture that repeats them.

📖 [Google SRE Book](https://sre.google/sre-book/table-of-contents/) — free, and the definitive text. Read "Monitoring Distributed Systems" and "Postmortem Culture."

---

## 13.9 Slack and working with humans

Communication is a technical skill and it's evaluated as one.

- **Ask good questions.** Not "it doesn't work." Instead: what you're trying to do, what you tried, what you expected, what happened, the exact error, and what you've already ruled out. This is the same discipline as writing a bug report — and it very often solves the problem while you're writing it ([rubber duck debugging](https://en.wikipedia.org/wiki/Rubber_duck_debugging)).
- **Don't ask to ask.** "Hey, are you free?" costs a round trip. Ask the actual question.
- **Use threads.** Keep channels readable.
- **Public channels over DMs.** Someone else has the same question, and answers become searchable.
- **Timebox.** Stuck for 30–60 minutes? Ask. Struggling is how you learn; suffering silently for a day is not.
- **Write things down.** A decision only discussed in a call didn't happen. Put it in the ticket, the PR, or a doc.
- **Async by default** for distributed teams. Assume nobody is reading in real time; write messages that stand alone.

📖 [How to ask good questions](https://jvns.ca/blog/good-questions/) — Julia Evans
📖 [The XY problem](https://xyproblem.info/) — you're stuck on your attempted solution, not your actual problem. Notice when you're doing this.

---

## Lab 13 — Run TaskFlow like a real team would

1. **Set up a board.** [Jira](https://www.atlassian.com/software/jira/free) free tier, [Linear](https://linear.app/) free tier, or GitHub Projects. Create an epic ("Task filtering and search") and break it into 4–6 stories, each with proper acceptance criteria.

2. **Write your CI workflow.** Typecheck, lint, test (against a Postgres service container), build. Get it green.

3. **Deliberately break it.** Push a type error. Watch CI fail. Read the log and find the failing step. Fix it. Push a failing test. Same. **You need to have seen a red build before you see one under pressure.**

4. **Enable branch protection** on `main`: require CI to pass, require one review. Then try to push directly to `main` and watch it get rejected.

5. **Work one story properly, end to end:**
   - Move the ticket to In Progress
   - `git switch -c feat/task-filters`
   - Small commits, conventional messages
   - Push, open a PR referencing the ticket, write a real description
   - **Review your own PR on GitHub** and leave at least three comments on your own code as though you were a reviewer. This is unusual, slightly awkward, and extremely effective.
   - Get CI green, merge, close the ticket

6. **Deploy it.** Frontend to Vercel, API to Fly.io or Render, database to Neon. Set the environment variables. **Get a real URL that a friend can open.** Expect this to take longer than you think — CORS, environment variables, and migrations will each bite once.

7. **Add Sentry** to both apps. Then deliberately throw an error in production and find it in the dashboard, with the stack trace and the user context.

8. **Write a postmortem** for the worst bug you hit during this course: timeline, impact, root cause, prevention. One page.

```text
Prompt for Claude Code:
I'm about to open a PR for [describe the change]. Act as a demanding but
constructive senior reviewer.

Here's the diff: [paste git diff main...HEAD]

1. Review it comment by comment with file:line references, prefixed
   `blocking:`, `suggestion:`, or `nit:`
2. Tell me whether my PR description [paste it] gives you what you need
3. Is this PR too big? If so, how would you split it?
4. What tests are missing?
5. If this broke in production at 3am, would the logs tell me why?
```

---

## Understanding Gate

1. Why do teams that deploy more often have *fewer* failures, not more?
2. What is a Definition of Done and what breaks without one?
3. Why keep PRs under 400 lines?
4. What must be true of a database migration during a rolling deploy?
5. Why does CI need to run on every PR rather than before release?
6. What's the difference between continuous delivery and continuous deployment?
7. Why alert on error rate rather than CPU?
8. What makes a postmortem blameless, and why does it matter practically?
9. You're stuck for 90 minutes. What do you do, and what exactly do you write?

---

**Next:** [Module 14 — Capstone & What's Next](14-capstone.md)
