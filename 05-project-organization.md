# Module 05 — Project Organization

**Time:** ~3 hours
**Prereq:** [Module 04](04-git.md)

---

## Why this module exists

A folder structure is an argument about how the system is divided. Get it right and new code has an obvious home; get it wrong and every change touches nine files. This module also covers the boring-but-essential machinery — dependencies, config, secrets, tooling — that every project needs and no tutorial explains.

---

## 5.1 `package.json` — the project manifest

Every JavaScript project has one. It's the answer to "what is this, what does it need, and how do I run it?"

```jsonc
{
  "name": "@taskflow/api",
  "version": "0.1.0",
  "type": "module",              // use ES modules (import/export), not CommonJS
  "private": true,               // guard against accidentally publishing to npm
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "test": "vitest run",
    "typecheck": "tsc --noEmit",
    "lint": "eslint ."
  },
  "dependencies": {              // needed to RUN in production
    "hono": "^4.10.0",
    "drizzle-orm": "^0.44.0"
  },
  "devDependencies": {           // needed only to build/test/develop
    "typescript": "^5.9.0",
    "vitest": "^3.2.0"
  }
}
```

**`scripts` are the project's public interface.** A newcomer (or a CI pipeline, or Claude) should be able to run `dev`, `build`, `test`, `lint` without knowing anything else. Keep those four names consistent across every project you ever work on.

### Version ranges and the lockfile

```
"hono": "^4.10.2"    → >=4.10.2 <5.0.0   (caret: any compatible minor/patch)
"hono": "~4.10.2"    → >=4.10.2 <4.11.0   (tilde: patches only)
"hono": "4.10.2"     → exactly 4.10.2
```

That's [semantic versioning](https://semver.org/): `MAJOR.MINOR.PATCH`. Major = breaking changes, minor = new features (backwards compatible), patch = bug fixes.

So if `package.json` allows a range, how does your teammate get *exactly* your versions? The **lockfile** — `package-lock.json` (or `pnpm-lock.yaml`, or `bun.lock`). It records the exact resolved version of every package *and every package's packages*, recursively.

> **Commit the lockfile. Always.** It is what makes builds reproducible. Without it, "works on my machine" comes back, and a transitive dependency can change under you between Tuesday and Wednesday.

In CI you use `npm ci` (not `npm install`) — it installs strictly from the lockfile and fails if `package.json` disagrees.

### `node_modules`

The folder where dependencies are physically installed. It's enormous — a modest project has 300 MB and 40,000 files — because JavaScript's culture is many tiny packages, and each has its own dependencies.

Never commit it, never edit it, and if things get weird: delete it and reinstall. That genuinely is the standard first debugging step.

```bash
rm -rf node_modules && npm install
```

**Supply chain awareness.** Installing a package means running someone else's code on your machine and shipping it to your users. Before adding a dependency: is it maintained? How many dependencies does *it* pull in? Could I write this in 20 lines? Check [npmgraph](https://npmgraph.js.org/) or [Socket](https://socket.dev/) for anything you're unsure about. This is not paranoia — [real attacks happen this way](https://en.wikipedia.org/wiki/Npm_left-pad_incident) regularly.

📖 [npm docs: package.json](https://docs.npmjs.com/cli/v11/configuring-npm/package-json) · [Semantic Versioning](https://semver.org/)

---

## 5.2 Monorepo layout

TaskFlow has two applications (a web app and an API) that share types. Three options:

1. **Two separate repos** — clean isolation, but sharing types means publishing a package. Painful for a small team.
2. **One repo, one package** — everything jumbled together. Fine for tiny projects, bad once frontend and backend need different builds.
3. **Monorepo** — one repository, multiple packages, managed by workspaces. ✅ This is what we'll do, and it's what most modern teams do.

```
taskflow/
├── apps/
│   ├── api/                  # Hono backend (Node)
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── web/                  # React frontend
│       ├── src/
│       ├── package.json
│       └── tsconfig.json
├── packages/
│   └── shared/               # types + Zod schemas used by BOTH
│       ├── src/
│       └── package.json
├── docker-compose.yml        # local Postgres
├── package.json              # workspace root
├── tsconfig.base.json        # shared compiler settings
├── .env.example
├── .gitignore
├── CLAUDE.md
└── README.md
```

Root `package.json`:

```jsonc
{
  "name": "taskflow",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "npm run dev --workspaces --if-present",
    "typecheck": "npm run typecheck --workspaces --if-present",
    "test": "npm run test --workspaces --if-present"
  }
}
```

Now `apps/web` can `import { type Task } from "@taskflow/shared"` and the package manager links it locally — no publishing.

**Why this matters more than it sounds:** the `shared` package is where your API contract lives. When you change the shape of a task in one place, *the frontend stops compiling*. That's a full-stack type-safety guarantee, and it's the main reason to use TypeScript on both ends.

📖 [npm workspaces](https://docs.npmjs.com/cli/v11/using-npm/workspaces) · [pnpm workspaces](https://pnpm.io/workspaces) (faster; worth switching to later) · [Turborepo](https://turborepo.com/docs) (caching + task orchestration, for when builds get slow)

---

## 5.3 Organizing code *inside* an app

Two competing schemes:

**By technical layer** — the default in tutorials:
```
src/
├── controllers/
├── services/
├── repositories/
└── models/
```
Every feature is smeared across four folders. Fine for small apps; annoying as they grow.

**By feature** — better as things scale:
```
src/
├── features/
│   ├── tasks/
│   │   ├── tasks.routes.ts
│   │   ├── tasks.service.ts
│   │   ├── tasks.repository.ts
│   │   ├── tasks.schema.ts
│   │   └── tasks.test.ts
│   └── auth/
├── db/
│   ├── schema.ts
│   └── client.ts
├── lib/            # generic, reusable, feature-agnostic helpers
└── index.ts        # composition root — wires everything together
```

Everything about tasks lives in one folder. Deleting the feature is `rm -rf features/tasks`. **We'll use this one.**

Two rules that keep it healthy:
- **Dependencies point inward.** Routes may import services; services may import repositories; *never the reverse*. If your database layer imports something HTTP-shaped, the layering has broken.
- **`lib/` is for genuinely generic things.** A date formatter, yes. `formatTaskDueDate`, no — that belongs in the tasks feature.

📖 [Bulletproof React](https://github.com/alan2207/bulletproof-react) — an opinionated, well-argued project-structure reference. Read `docs/project-structure.md`.

---

## 5.4 Configuration and secrets

**The [Twelve-Factor App](https://12factor.net/config) rule: store config in the environment, not in code.** The same build artifact should run in dev, staging and production, configured only by environment variables.

```bash
# .env  — GITIGNORED, never committed
DATABASE_URL=postgres://postgres:devpassword@localhost:5432/taskflow
BETTER_AUTH_SECRET=a-long-random-string-generated-locally
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...
```

```bash
# .env.example — COMMITTED. Documents what's needed, with no real values.
DATABASE_URL=postgres://postgres:password@localhost:5432/taskflow
BETTER_AUTH_SECRET=generate-with-openssl-rand-base64-32
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
```

**Validate your config at startup.** A missing env var should crash the process immediately with a clear message — not produce `undefined` that surfaces as a mystery bug 40 minutes into production traffic.

```ts
// src/config.ts
import { z } from "zod";

const EnvSchema = z.object({
  DATABASE_URL: z.string().url(),
  BETTER_AUTH_SECRET: z.string().min(32),
  PORT: z.coerce.number().default(3000),
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
});

// Throws at import time — the server never starts misconfigured. This is a feature.
export const env = EnvSchema.parse(process.env);
```

This pattern — **fail fast at the boundary** — recurs throughout the course. Same idea as validating HTTP request bodies (Module 08).

⚠️ **Frontend secrets don't exist.** Anything Vite exposes as `VITE_*` is bundled into JavaScript that ships to the browser. Users can read it. There is no such thing as a secret in a frontend app; anything sensitive lives on the server. ([Vite env docs](https://vite.dev/guide/env-and-mode))

---

## 5.5 Tooling every project should have

| Tool | Purpose | Setup |
|---|---|---|
| **TypeScript** | Type checking | [`tsconfig` reference](https://www.typescriptlang.org/tsconfig) — start from [tsconfig/bases](https://github.com/tsconfig/bases) |
| **ESLint** | Catches likely bugs & enforces patterns | [Getting started](https://eslint.org/docs/latest/use/getting-started) |
| **Prettier** | Formats code; ends all style arguments | [Install](https://prettier.io/docs/install) |
| **EditorConfig** | Consistent indentation across editors | [editorconfig.org](https://editorconfig.org/) |
| **Husky + lint-staged** | Run lint/format on staged files before commit | [Husky](https://typicode.github.io/husky/) |

> **Formatting is not worth a single minute of human discussion.** Install Prettier, enable format-on-save, never think about it again. That's the entire point of the tool.

A `tsconfig.base.json` worth starting from:

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2023"],
    "strict": true,                          // ← the important one
    "noUncheckedIndexedAccess": true,        // arr[0] is T | undefined. Annoying. Correct.
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

---

## 5.6 The README

Your project's front door. Write it for a person who has never seen the repo and has 5 minutes.

```markdown
# TaskFlow
A multi-user todo app. React + Hono + PostgreSQL.

## Quick start
    cp .env.example .env
    docker compose up -d          # starts Postgres
    npm install
    npm run db:migrate
    npm run dev                   # web on :5173, api on :3000

## Architecture
[the three-tier diagram]

## Layout
- apps/web — React frontend (Vite)
- apps/api — Hono HTTP API (Node)
- packages/shared — types & schemas used by both

## Common tasks
- `npm test` — run all tests
- `npm run db:studio` — browse the database
```

Test it by following it yourself on a clean clone. Every README rots; the ones that don't are the ones someone actually ran.

---

## Lab 05 — Scaffold TaskFlow for real

This is the skeleton you'll fill in for the rest of the course. Build it by hand — you'll be glad you know where everything came from.

```bash
cd ~/projects/taskflow
mkdir -p apps/api/src apps/web packages/shared/src
```

1. Write the root `package.json` with workspaces (above).
2. Write `tsconfig.base.json` (above), and a `tsconfig.json` in each package that `extends` it.
3. Create `packages/shared` with a `package.json` (`"name": "@taskflow/shared"`) and a `src/index.ts` exporting the `Task` type from Lab 02.
4. Create `apps/api/package.json` depending on `"@taskflow/shared": "*"`. Run `npm install` at the root and then find the symlink npm created inside `node_modules/@taskflow/`. Confirm you understand what happened.
5. Write `docker-compose.yml` for Postgres:

   ```yaml
   services:
     db:
       image: postgres:17
       environment:
         POSTGRES_PASSWORD: devpassword
         POSTGRES_DB: taskflow
       ports: ["5432:5432"]
       volumes: ["taskflow-data:/var/lib/postgresql/data"]
   volumes:
     taskflow-data:
   ```

   `docker compose up -d`, then `docker compose ps`. Compare this to the long `docker run` from Module 00 — same thing, written down and version-controlled. That's the point of compose.
6. Write `.env.example` and a gitignored `.env`. Add `src/config.ts` with Zod validation. **Prove it works** by deleting a variable from `.env` and confirming the process refuses to start with a readable message.
7. Install and configure Prettier + ESLint. Enable format-on-save in VS Code.
8. Write the README. Commit everything on a branch, open a PR, merge it.

```text
Prompt for Claude Code:
I've scaffolded a monorepo at [paste your `tree -L 3 -I node_modules` output].
Review the STRUCTURE only — don't write code.

1. Is anything in the wrong place, or missing?
2. Where should each of these go, and why: a shared date-formatting helper,
   the database schema, Zod schemas for API request bodies, a script that
   seeds test data?
3. What will hurt when this project has 20 features instead of 2?
```

---

## Understanding Gate

1. Why commit the lockfile but not `node_modules`?
2. `dependencies` vs `devDependencies` — what breaks if you get it wrong?
3. What does `^4.10.2` allow, and what risk does that carry?
4. Why does the `shared` package exist? What concrete bug does it prevent?
5. Why must config come from the environment rather than a committed file?
6. Why is there no such thing as a secret in a frontend app?
7. Organizing by feature vs by layer — argue for each, then say which you'd pick for a 50-person team.

---

**Next:** [Module 06 — Client–Server Communication & HTTP](06-client-server-http.md)
