# Module 12 — Containers

**Time:** ~7 hours
**Prereq:** [Module 01](01-how-computers-run-code.md), [Module 08](08-backend-api.md), [Module 11](11-testing.md)

---

## Why this module exists

Your app currently runs one way: `npm run dev`, on your laptop, with Postgres in a container you started in Module 00 and haven't thought about since.

To ship it, you need to answer a question you haven't faced yet: **what, exactly, is the thing you deploy?** Not "the repo" — a server can't run a repo. Something has to turn your source into a self-contained, runnable unit that behaves identically on your machine, in CI, and in production.

That unit is a **container image**, and understanding it is the difference between deploying and *praying*.

This module is also a direct sequel to Module 01. A container is not a small computer. It's a **process** — with a lie told to it about what the filesystem and network look like. Once you see that, the whole thing demystifies.

---

## 12.1 The problem

You've already met it in miniature. Your API needs Node 22, Postgres 17, and a specific set of npm packages. The production server has Node 18 and no Postgres. A colleague has Node 24 and Postgres 16. Someone's on Windows.

Historical answers, in order of desperation:

1. **A README with setup steps.** Rots immediately. Silently wrong on one OS.
2. **A configuration management tool** (Ansible, Chef) that scripts the server setup. Better — but it mutates a long-lived machine, and after two years of scripts nobody knows what's actually on it. This is called *configuration drift*, and it's why "don't touch that server, it works" is a real thing people say.
3. **A virtual machine image.** Genuinely reproducible! But a VM emulates hardware and boots a whole operating system: gigabytes of disk, ~30 seconds to start, hundreds of MB of RAM before your code runs.
4. **A container.** The reproducibility of a VM at roughly the cost of a process.

### Containers vs virtual machines

```
        VIRTUAL MACHINES                        CONTAINERS
  ┌─────────┬─────────┬─────────┐      ┌─────────┬─────────┬─────────┐
  │  App A  │  App B  │  App C  │      │  App A  │  App B  │  App C  │
  ├─────────┼─────────┼─────────┤      ├─────────┼─────────┼─────────┤
  │  libs   │  libs   │  libs   │      │  libs   │  libs   │  libs   │
  ├─────────┼─────────┼─────────┤      └────┬────┴────┬────┴────┬────┘
  │ Guest   │ Guest   │ Guest   │           │         │         │
  │ OS      │ OS      │ OS      │      ┌────┴─────────┴─────────┴────┐
  │ +kernel │ +kernel │ +kernel │      │    Container runtime         │
  ├─────────┴─────────┴─────────┤      ├──────────────────────────────┤
  │       Hypervisor            │      │      Host OS kernel          │
  ├─────────────────────────────┤      ├──────────────────────────────┤
  │       Host hardware         │      │      Host hardware           │
  └─────────────────────────────┘      └──────────────────────────────┘
     Each app carries a kernel.          All apps SHARE the host kernel.
     GBs, ~30s boot.                     MBs, ~50ms start.
```

**The whole trick:** containers don't virtualize the machine. They use two Linux kernel features to give an ordinary process a *distorted view* of the system:

- **Namespaces** — isolate what a process can *see*. There are several: PID (which processes exist), mount (what the filesystem looks like), network (what interfaces and ports exist), user (which UIDs exist), UTS (hostname), IPC. A process in a new PID namespace genuinely believes it is PID 1 and that no other processes exist.
- **cgroups** (control groups) — limit what a process can *use*: CPU shares, memory, disk I/O, number of processes.

That's it. There is no container "object" in the kernel. A container is a normal process with namespaces around it and cgroups on it.

> **Honest caveat for macOS and Windows.** The kernel features above are Linux-only. On a Mac or Windows machine, Docker Desktop quietly runs a lightweight **Linux VM**, and your containers are processes inside *that*. So "shares the host kernel" means the VM's kernel. This matters practically: it's why file I/O across the boundary is slow, why `--platform` occasionally matters on Apple Silicon, and why the process-visibility demo below behaves differently for you than for a Linux user.

📖 [Cloudflare: What is a container?](https://www.cloudflare.com/learning/cloud/what-is-containerization/) · [Docker: what is a container](https://www.docker.com/resources/what-container/)
📺 [Liz Rice — Containers From Scratch](https://www.youtube.com/watch?v=8fi7uSYlOdc) ⭐ — builds a container in ~100 lines of Go, live. The single best way to internalize "it's just a process."

---

## 12.2 Prove it's just a process

Do these now. This section is the conceptual core of the module.

**A. The PID namespace lie.**

```bash
docker run --rm -it alpine sh
# inside the container:
ps aux
```

You'll see two processes: `sh` as **PID 1**, and your `ps`. The container is convinced it just booted. Meanwhile your laptop is running 400 processes it can't see.

`exit`, and on **Linux or WSL** run a container in the background and find it from outside:

```bash
docker run -d --name spy alpine sleep 300
ps aux | grep "sleep 300"      # ← there it is, in YOUR process table
```

Same process, two PIDs: one inside its namespace, one on the host. **Nothing was virtualized.** (On macOS this won't appear in `ps` — it's inside Docker Desktop's VM. `docker top spy` shows it regardless.)

**B. cgroups are real limits.**

```bash
docker run --rm --memory=32m alpine sh -c \
  'dd if=/dev/zero of=/dev/null bs=1M count=100; echo survived'

# Now actually allocate past the limit:
docker run --rm --memory=32m python:3-alpine python -c "x = 'a' * (200*1024*1024)"
```

The second one gets killed. `docker inspect --format '{{.State.OOMKilled}}' <id>` on a stopped container confirms it. That's the kernel's OOM killer enforcing a cgroup limit — **the same mechanism that will kill your API in production when it leaks memory** (Module 01). Recognizing "exit code 137" as "OOM-killed" will save you an afternoon someday.

**C. The mount namespace.**

```bash
docker run --rm alpine ls /
```

A complete-looking Linux filesystem — and none of it is your disk. It's the Alpine image's layers, mounted as this process's root. Your actual files are simply not in its view.

**D. Signals still work.** From Module 01: `Ctrl+C` sends `SIGINT`. Containers are processes, so this is unchanged:

```bash
docker run --rm -it alpine sh -c 'trap "echo caught SIGTERM; exit 0" TERM; sleep 300'
# from another terminal:
docker stop <container-name>
```

`docker stop` sends `SIGTERM`, waits 10 seconds, then `SIGKILL`. **Your API must handle `SIGTERM`** by closing the database pool and finishing in-flight requests — otherwise every deploy drops connections mid-request. This is called *graceful shutdown*, and you'll add it in the lab.

---

## 12.3 Images and layers

An **image** is an ordered stack of read-only filesystem layers plus metadata (default command, env vars, exposed ports). A **container** is an image plus a thin writable layer on top.

```
      ┌──────────────────────────┐
      │ writable layer           │  ← the running container; discarded on rm
      ├──────────────────────────┤
      │ COPY dist ./dist         │  ┐
      ├──────────────────────────┤  │
      │ RUN npm ci               │  │  read-only image layers,
      ├──────────────────────────┤  │  shared between containers
      │ COPY package.json .      │  │
      ├──────────────────────────┤  │
      │ FROM node:22-slim        │  ┘
      └──────────────────────────┘
```

Each instruction in a `Dockerfile` produces a layer. Layers are content-addressed (hashed), so identical layers are stored once and shared — ten containers from one image cost one copy of the image plus ten small writable layers.

```bash
docker pull node:22-slim
docker history node:22-slim      # see the layers and their sizes
docker image ls
```

### The build cache — and why instruction order matters

Docker reuses a cached layer only if that instruction *and every instruction before it* are unchanged. Change one line, and **every layer below it is rebuilt.**

```dockerfile
# ❌ Slow: any source edit invalidates COPY, so npm ci re-runs. Every build. ~90s.
COPY . .
RUN npm ci

# ✅ Fast: dependencies only re-install when package files change. ~3s otherwise.
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

**Order instructions from least- to most-frequently-changed.** Your dependencies change weekly; your source changes every 30 seconds.

This is cache invalidation (Module 06) and it's worth measuring rather than believing:

```bash
time docker build -t taskflow-api .     # cold
time docker build -t taskflow-api .     # warm — should be near-instant
# now edit one line of src/index.ts and rebuild. Note the difference.
```

### `.dockerignore`

Same idea as `.gitignore`, and it matters more than it sounds: everything not ignored gets sent to the builder as "build context" *before the build starts*, and any of it can bust your cache.

```
node_modules
dist
.git
.env
*.log
coverage
playwright-report
```

Shipping `.git` or `.env` into an image is a real and common way to leak secrets.

📖 [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) · [Build cache](https://docs.docker.com/build/cache/)

---

## 12.4 A Dockerfile for the API

```dockerfile
# syntax=docker/dockerfile:1

# ---------- deps: install node_modules, cached on lockfile changes only ----------
FROM node:22-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
COPY apps/api/package.json apps/api/
COPY packages/shared/package.json packages/shared/
RUN npm ci

# ---------- build: compile TypeScript ----------
FROM node:22-slim AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build --workspace @taskflow/api

# ---------- runtime: only what's needed to RUN ----------
FROM node:22-slim AS runtime
ENV NODE_ENV=production
WORKDIR /app

COPY package.json package-lock.json ./
COPY apps/api/package.json apps/api/
COPY packages/shared/package.json packages/shared/
RUN npm ci --omit=dev && npm cache clean --force

COPY --from=build /app/apps/api/dist ./apps/api/dist
COPY --from=build /app/packages/shared/dist ./packages/shared/dist

USER node
EXPOSE 3000
CMD ["node", "apps/api/dist/index.js"]
```

Every line here is a decision worth understanding:

- **Multi-stage build.** `FROM ... AS build` and `FROM ... AS runtime` are separate images; only the last one ships. Your TypeScript compiler, dev dependencies, test files and source never reach production. Smaller image, smaller attack surface, faster deploys. Compare `docker image ls` for a single-stage build versus this one.
- **`node:22-slim`, pinned.** Not `node:latest` — that changes under you and destroys reproducibility. Not full `node:22` (~1.1 GB of build tools you don't need). `-alpine` is smaller still but uses musl instead of glibc, which occasionally breaks native modules; `slim` is the safer default.
- **`USER node`.** By default containers run as **root**. A container escape as root is far worse than as an unprivileged user. The official Node images ship a `node` user — use it. *(Everything after this line runs as `node`, so it goes after the installs.)*
- **`ENV NODE_ENV=production`** — makes npm skip dev dependencies and puts libraries in production mode.
- **`EXPOSE 3000`** — documentation, not enforcement. It doesn't publish anything; `-p` does.
- **`CMD ["node", "..."]` in exec form** (a JSON array), not `CMD node ...`. Shell form wraps your process in `/bin/sh`, which **does not forward signals** — so `docker stop` hits the shell and your app never sees `SIGTERM`, never shuts down gracefully, and gets `SIGKILL`ed 10 seconds later. This is the single most common Dockerfile bug.

### PID 1 and graceful shutdown

Your process is PID 1 inside the container. PID 1 is special: the kernel doesn't apply default signal handlers to it, and it's expected to reap orphaned child processes. Practically, add this to your API:

```ts
// apps/api/src/index.ts
const server = serve({ fetch: app.fetch, port: env.PORT });

const shutdown = async (signal: string) => {
  console.log(`${signal} received, shutting down`);
  server.close();          // stop accepting new connections, finish in-flight ones
  await pool.end();        // close the database connection pool cleanly
  process.exit(0);
};

process.on("SIGTERM", () => shutdown("SIGTERM"));   // ← what `docker stop` and every
process.on("SIGINT",  () => shutdown("SIGINT"));    //   orchestrator sends
```

And run with `--init` (or `init: true` in compose) so a proper init process handles zombie reaping for you.

📖 [Docker: Node.js best practices](https://docs.docker.com/guides/nodejs/) · [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)

---

## 12.5 Compose: the whole stack

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: taskflow
    ports: ["5432:5432"]
    volumes: ["taskflow-data:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d taskflow"]
      interval: 5s
      timeout: 3s
      retries: 10

  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
    init: true
    environment:
      DATABASE_URL: postgres://postgres:devpassword@db:5432/taskflow
      WEB_ORIGIN: http://localhost:5173
      PORT: 3000
    ports: ["3000:3000"]
    depends_on:
      db:
        condition: service_healthy

volumes:
  taskflow-data:
```

Three things here are the actual lesson:

**1. `db:5432`, not `localhost:5432`.** Compose puts every service on a shared network and runs a DNS server that resolves **service names to container IPs**. Inside the `api` container, `localhost` means *the api container itself* — Postgres isn't there. This trips up everyone once, and the error (`ECONNREFUSED 127.0.0.1:5432`) is now one you'll recognize instantly.

Note the asymmetry: `psql` on your laptop still uses `localhost:5432` because of the published port. Same database, two different addresses depending on who's asking. That's Module 06's "an origin is where you're standing" made physical.

**2. `depends_on` alone is a trap.** Plain `depends_on: [db]` waits for the container to *start*, not for Postgres to be *ready to accept connections* — so your API races ahead, fails to connect, and crashes. The `condition: service_healthy` form plus a `healthcheck` is the fix. (Belt and braces: make your app retry its first connection anyway. In production nothing guarantees ordering.)

**3. Named volumes vs bind mounts.** `taskflow-data:/var/lib/...` is a **named volume** — Docker-managed storage that outlives the container (that's why your data survived the delete-and-recreate in Lab 00). A **bind mount** (`./src:/app/src`) maps a host folder in, which is what you want for hot-reload in development, not in production.

Useful commands:

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f api          # ← where your console.log goes now
docker compose exec api sh          # shell inside the running container
docker compose exec db psql -U postgres -d taskflow
docker compose down                 # stop + remove containers (volumes survive)
docker compose down -v              # ...and delete the volumes. Destroys your data.
```

`docker compose logs -f api` deserves emphasis: once your app is in a container, **`console.log` goes to the container's stdout**, not your terminal. Knowing where the logs went is half of container debugging. (This is also why Module 13 says log structured JSON to stdout and let the platform collect it — [12-Factor logs](https://12factor.net/logs).)

📖 [Compose file reference](https://docs.docker.com/reference/compose-file/) · [Compose networking](https://docs.docker.com/compose/how-tos/networking/)

---

## 12.6 Images as artifacts: tags and registries

A **registry** is where images live: [Docker Hub](https://hub.docker.com/), [GHCR](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) (GitHub's, free for public and private), or your cloud provider's.

```bash
docker build -t ghcr.io/yourname/taskflow-api:0.1.0 -f apps/api/Dockerfile .
echo $GITHUB_TOKEN | docker login ghcr.io -u yourname --password-stdin
docker push ghcr.io/yourname/taskflow-api:0.1.0
```

**Tags are mutable pointers; digests are not.**

```
ghcr.io/you/taskflow-api:0.1.0                          ← a tag. Can be moved.
ghcr.io/you/taskflow-api@sha256:9f2a...                 ← a digest. Immutable, forever.
```

> **`:latest` is not a version.** It's just the default tag name, and it means "whatever someone pushed most recently." Deploying `:latest` means you cannot answer "what is running in production right now?" — which is exactly the question you'll be asked during an incident. **Tag with the git SHA** (`:sha-a3f8b21`) and optionally also a semver tag. Then any running container traces back to an exact commit.

This is the point the whole module has been building to, and it's the concrete form of the 12-Factor claim in Module 13:

> **Build once, run everywhere.** The identical image — same bytes, same digest — is what CI tested, what ran in staging, and what serves production. The only difference between environments is the environment variables passed in. If you rebuild per environment, you never tested what you shipped.

---

## 12.7 CI in containers

Now CI can build the artifact, test *inside* it, and publish it.

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push: { branches: [main] }
  pull_request:

env:
  IMAGE: ghcr.io/${{ github.repository }}/api

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write            # ← needed to push to GHCR

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

      - uses: docker/setup-buildx-action@v3

      - name: Build image (load locally, don't push yet)
        uses: docker/build-push-action@v6
        with:
          context: .
          file: apps/api/Dockerfile
          load: true
          tags: ${{ env.IMAGE }}:${{ github.sha }}
          cache-from: type=gha          # ← reuse layer cache across CI runs
          cache-to: type=gha,mode=max

      - name: Run the test suite inside the image
        run: |
          docker run --rm --network host \
            -e DATABASE_URL=postgres://postgres:postgres@localhost:5432/taskflow_test \
            ${{ env.IMAGE }}:${{ github.sha }} \
            npm run test

      - name: Log in to GHCR
        if: github.ref == 'refs/heads/main'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push (only from main, only if tests passed)
        if: github.ref == 'refs/heads/main'
        run: |
          docker push ${{ env.IMAGE }}:${{ github.sha }}
          docker tag ${{ env.IMAGE }}:${{ github.sha }} ${{ env.IMAGE }}:latest
          docker push ${{ env.IMAGE }}:latest
```

What changed conceptually versus a plain `npm test` pipeline:

- **You test the artifact, not the source.** A test run on GitHub's Ubuntu with GitHub's Node proves your *code* works there. Running the suite inside the image proves the *thing you're about to deploy* works.
- **Only tested images get pushed**, and only from `main`. Nothing untested can reach the registry, which means nothing untested can reach production.
- **`cache-from/to: type=gha`** persists the layer cache between CI runs — without it every build is cold and your pipeline takes five minutes instead of one.

> ⚠️ The runtime image above ships with `--omit=dev`, so `vitest` isn't in it. Two clean options: add a `test` stage to the Dockerfile (`FROM build AS test`, with dev deps present) and `docker build --target test`, or keep testing from source and use the image build purely as an artifact check. **The first is more honest, the second is simpler** — pick one deliberately and write down why.

**Deploying an image instead of a repo.** Fly.io, Render, Railway, Cloud Run and Kubernetes all accept `ghcr.io/you/taskflow-api:<sha>` directly. Deployment becomes: point the platform at a digest, pass env vars, done. Rollback becomes: point it at the previous digest. That's a five-second recovery instead of a rebuild — the "time to restore service" metric from Module 13.

---

## 12.8 Security

Containers are isolation, not a sandbox. A few rules that cover most of the risk:

| Rule | Why |
|---|---|
| **Run as non-root** (`USER node`) | Limits the blast radius of any escape or RCE |
| **Never `COPY` secrets in** | Layers are permanent. A secret in layer 3 is still there after you `rm` it in layer 7 — exactly like a secret committed to git (Module 04). Anyone who pulls the image can extract it. |
| **Pass secrets as env vars at run time** | Or use build secrets (`RUN --mount=type=secret`) for build-time needs |
| **Pin base images**, ideally by digest | `node:22-slim` moves; `node:22-slim@sha256:…` doesn't |
| **Rebuild regularly** | Your base image accumulates CVEs. An image built six months ago is six months behind on patches. |
| **Scan** | `docker scout cves <image>`, or [Trivy](https://trivy.dev/) |
| **Minimal base** | Fewer packages, fewer vulnerabilities. Consider [distroless](https://github.com/GoogleContainerTools/distroless) once you're comfortable. |

Prove the secret-in-a-layer point to yourself — it's more convincing than being told:

```bash
docker build -t leaky - <<'EOF'
FROM alpine
RUN echo "SUPER_SECRET=hunter2" > /secret.txt
RUN rm /secret.txt
EOF
docker save leaky | tar -xO --wildcards '*/layer.tar' | strings | grep -a hunter2
```

The file is "deleted" and the secret is still in the image. 

📖 [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html) · [Docker build secrets](https://docs.docker.com/build/building/secrets/)

---

## 12.9 What this is *not*: orchestration

Docker runs containers on **one machine**. The moment you want many machines, automatic restarts, rolling deploys, service discovery across hosts and autoscaling, you need an **orchestrator** — almost always [Kubernetes](https://kubernetes.io/).

Kubernetes is a control loop: you declare the desired state ("5 replicas of this image, 512 MB each, reachable at this address") and it continuously works to make reality match. It's genuinely powerful and genuinely a large amount of complexity.

> **You do not need Kubernetes.** Not for TaskFlow, not for your next three projects, quite possibly not ever. A container image on Fly.io, Render, Railway, or Cloud Run gets you rolling deploys, health checks and autoscaling with none of the operational burden. Adopt an orchestrator when you have a problem it solves, not because it's on job descriptions.
>
> The good news: **the image is the interface.** Everything in this module transfers unchanged if you ever do move to Kubernetes. You'd be learning new deployment *config*, not new fundamentals.

📖 [Kubernetes concepts](https://kubernetes.io/docs/concepts/) — read the overview only, for vocabulary

---

## Lab 12 — Containerize TaskFlow

1. **Do all four experiments in 12.2** and write one sentence each on what they proved.

2. **Add graceful shutdown** to your API (`SIGTERM`/`SIGINT` handlers, close the server and the pool). Test it: `docker stop` and confirm you see your log line rather than a 10-second hang.

3. **Write `apps/api/Dockerfile`** and `.dockerignore`. Build it, run it against your existing Postgres container, and hit `/health` with curl.

4. **Measure the cache.** Time a cold build, a warm build, a build after editing one source line, and a build after changing `package.json`. Then deliberately move `COPY . .` above `RUN npm ci`, re-time it, and put it back. **Write the four numbers down** — this is the part people skip and then never really believe.

5. **Compare single-stage vs multi-stage.** Make a naive one-stage Dockerfile, build both, and compare `docker image ls`. Then `docker run --rm <single-stage> sh -c 'ls node_modules | wc -l'` to see what you'd have been shipping.

6. **Write `docker-compose.yml`** for db + api. Get the whole thing up with one command from a clean state:
   ```bash
   docker compose down -v && docker compose up --build
   ```
   It must work end to end with no manual steps. That's the bar.

7. **Break the networking on purpose.** Change `db:5432` to `localhost:5432` in the api service. Read the error. Now you own that one forever.

8. **Break the ordering on purpose.** Replace the healthcheck condition with plain `depends_on: [db]` and restart repeatedly until you catch the race. Then fix it *twice*: with the healthcheck, and with a connection retry in your app.

9. **Run migrations in the container.** Add a `migrate` service (or an entrypoint step) so a fresh `docker compose up` produces a working, migrated database. Decide — and justify in a comment — whether migrations should run on container start or as a separate deploy step. (Module 13 has opinions.)

10. **Push to GHCR** with a git-SHA tag. Then pull it on a clean state (`docker image rm` first) and run it. You just deployed to your own machine from a registry.

11. **Update CI** to build the image, run the suite inside it, and push only from `main`. Watch the second run be faster because of the layer cache.

12. **Prove the secret leak** with the `docker save | strings` trick in 12.8.

```text
Prompt for Claude Code:
Here's my Dockerfile, .dockerignore and docker-compose.yml: [paste all three].

Review them as a platform engineer. Specifically:
1. Layer ordering — where am I busting the build cache unnecessarily?
2. What am I shipping to production that shouldn't be there?
3. Security: user, secrets, base image, anything writable that shouldn't be
4. Will this shut down gracefully on SIGTERM? Walk me through the signal path.
5. What breaks if the database restarts while the API is running?
6. What's in my build context that shouldn't be?

Numbered list with line references. Don't rewrite the files — I'll fix them
and then you re-review.
```

---

## Understanding Gate

1. A container is not a VM. What are the two kernel features that make it work, and what does each one do?
2. You run `docker run` on a Mac. Where is your process actually running, and why does that matter?
3. Why does `COPY package.json` before `COPY .` make builds dramatically faster?
4. Why is `CMD node server.js` (shell form) a bug, and what's the symptom in production?
5. Inside the `api` container, why doesn't `localhost:5432` reach Postgres — even though it works from your laptop?
6. What's wrong with deploying `:latest`?
7. You `RUN rm /secret.txt` in a Dockerfile. Is the secret gone? Justify your answer.
8. Why run the test suite *inside* the image rather than on the CI runner?
9. What does exit code 137 mean, and what would you change in response?
10. Your colleague says you should move to Kubernetes. What do you ask them?

---

**Next:** [Module 13 — The Software Development Lifecycle](13-sdlc-teams-deploy.md)
