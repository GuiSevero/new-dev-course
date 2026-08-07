# Module 00 — Setting Up Your Machine

**Time:** 3–4 hours (most of it waiting for downloads)
**Assumes:** a brand-new computer with nothing installed.

---

## Why this module exists

Setting up a development environment is the first real test of the skill you're here to build: following precise instructions, reading error messages, and understanding *what* you just installed rather than just that it worked.

Every tool below is explained. Do not install anything you can't answer "what is this for?" about.

---

## 0.1 The terminal — your primary instrument

Graphical interfaces are optimized for discovery. Terminals are optimized for **repetition, precision, and automation**. Software development involves a great deal of all three. Also: Claude Code lives in a terminal.

### Open one

| OS | How |
|---|---|
| **macOS** | `Cmd+Space` → type "Terminal" → Enter. (Later, install [iTerm2](https://iterm2.com/) or [Ghostty](https://ghostty.org/) if you want a nicer one.) |
| **Windows** | Install [Windows Terminal](https://apps.microsoft.com/detail/9n0dx20hk701) from the Microsoft Store, then see the WSL section below. |
| **Linux** | `Ctrl+Alt+T` on most desktops. |

### Windows users: read this first

Most server software in the world runs on Linux. Windows is a fine machine to develop *on*, but you want a Linux environment *inside* it. That's **WSL2** (Windows Subsystem for Linux) — a real Linux kernel running alongside Windows, sharing your files.

Open PowerShell **as Administrator** and run:

```bash
wsl --install -d Ubuntu
```

Reboot, let Ubuntu finish setting up, create your Linux username and password. From now on, **every command in this course runs inside the Ubuntu/WSL terminal**, not PowerShell. Keep your project files under the Linux home directory (`~/`), not `/mnt/c/` — it's dramatically faster.

📖 [Microsoft's WSL install docs](https://learn.microsoft.com/en-us/windows/wsl/install) · [Set up a WSL dev environment](https://learn.microsoft.com/en-us/windows/wsl/setup/environment)

### The 12 commands that cover 90% of your terminal use

```bash
pwd                  # print working directory — "where am I?"
ls -la               # list files, including hidden ones, with details
cd projects          # change directory (cd ..  = up one,  cd ~ = home)
mkdir -p a/b/c       # make directories, including parents
touch file.txt       # create an empty file
cat file.txt         # dump a file's contents to the screen
less file.txt        # page through a long file (q to quit)
cp a.txt b.txt       # copy
mv a.txt b.txt       # move / rename
rm file.txt          # delete a file (rm -r dir  for a directory) — NO undo
grep -r "todo" .     # search for text recursively from here
curl https://example.com   # make an HTTP request
```

Two more that will save you constantly:

```bash
history | grep docker    # what was that docker command I ran yesterday?
which node               # where is the program that runs when I type `node`?
```

**Concepts to hold onto:**
- **Working directory** — every terminal has a "you are here." Most mistakes come from being in the wrong one.
- **`PATH`** — a list of folders the shell searches when you type a command name. Run `echo $PATH`. When something says `command not found`, it means: *the program is not in any folder on this list* (either not installed, or installed somewhere unlisted).
- **stdin / stdout / stderr** — every program has one input stream and two output streams (normal output, error output). The `|` pipe connects one program's stdout to the next one's stdin. `>` redirects stdout to a file.
- **Exit codes** — every command returns a number. `0` means success; anything else means failure. Run `echo $?` right after a command to see it. This is how scripts and CI decide whether a step passed.

🎓 **Do this**: [MIT's *The Missing Semester of Your CS Education*](https://missing.csail.mit.edu/) — lectures 1 and 2 (shell, shell tools). It's ~2 hours and it is the single highest-value thing in this module.

---

## 0.2 A package manager for your operating system

Installing software by downloading `.dmg`/`.exe` files by hand doesn't scale and can't be scripted. Use a package manager.

**macOS — [Homebrew](https://brew.sh/):**

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Follow the "Next steps" it prints at the end — it will tell you to add Homebrew to your `PATH`. That's the concept from above, in the wild. Verify with `brew --version`.

**Linux / WSL Ubuntu** — you already have `apt`:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential curl git unzip
```

**Windows (native, outside WSL)** — [winget](https://learn.microsoft.com/en-us/windows/package-manager/winget/) is built in.

---

## 0.3 Git

The tool that records the history of your code. Module 04 is entirely about it; for now, just install and identify yourself.

```bash
# macOS
brew install git
# Ubuntu / WSL — already installed above
```

```bash
git --version
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

Those last two settings: new repositories start on a branch called `main` (an industry-wide convention), and `git pull` will merge rather than rebase by default (both are explained in Module 04 — this just picks the less surprising one for now).

---

## 0.4 Node.js — via a version manager

**What Node.js is:** JavaScript was invented to run inside web browsers. Node.js is that same language engine (Google's V8, extracted from Chrome) packaged as a standalone program, plus APIs the browser deliberately doesn't give you: reading files, opening network sockets, spawning processes. It's how JavaScript escaped the browser and became a server language.

**Why a version manager:** different projects need different Node versions, and installing Node globally makes switching painful. [`fnm`](https://github.com/Schniz/fnm) is a fast, simple one.

```bash
# macOS
brew install fnm

# Linux / WSL
curl -fsSL https://fnm.vercel.app/install | bash
```

Then close and reopen your terminal, and:

```bash
fnm install --lts        # install the current Long Term Support version
fnm use --lts
fnm default $(fnm current)
node --version           # expect v22.x or v24.x
npm --version
```

If `fnm` isn't found after reopening, its installer needs a line added to your shell config (`~/.zshrc` on macOS, `~/.bashrc` on Ubuntu). The installer prints it — read the output.

**What `npm` is:** the package manager that ships with Node. It downloads libraries from the [npm registry](https://www.npmjs.com/) — a public repository of ~3 million JavaScript packages. Module 05 covers how it works and why `node_modules` is so large.

---

## 0.5 A note on other runtimes (nothing to install)

Node is not the only way to run JavaScript outside a browser any more. You'll hear about:

- **[Bun](https://bun.sh/)** — a newer runtime that's also a package manager, bundler and test runner. Much faster installs and startup, and it runs TypeScript directly with no build step.
- **[Deno](https://deno.com/)** — secure-by-default (a script can't read your files or hit the network unless you allow it), TypeScript-first.

You do **not** need either for this course, and installing things you don't need is how environments rot. Mentioning them because of a design choice you'll meet in Module 08:

> The backend framework we use, [Hono](https://hono.dev/), is built on the **web-standard `Request` and `Response` objects** — the same ones the browser's `fetch` uses. That means the identical application code runs on Node, Bun, Deno, or Cloudflare Workers; only the one line that binds it to a port changes.
>
> That portability isn't a marketing feature, it's a *teaching* one: it forces the framework to stay a thin, visible layer over the actual HTTP primitives, instead of inventing its own request object you'd have to learn separately. You'll see exactly what that means in Module 08.

Curious later? `curl -fsSL https://bun.sh/install | bash` and run your API on it — should be a one-line change. That's a good experiment for Module 14, not for today.

---

## 0.6 Docker — for the database

**What it is:** Docker runs software in **containers** — isolated packages that include the program *and* everything it needs to run (libraries, config, file layout). A container behaves the same on your laptop and on a production server, which eliminates the entire genre of bug called "works on my machine."

**Why you want it here:** installing PostgreSQL natively means installing a database server, configuring it, and having it running in the background forever. With Docker, your database is one command to start, one command to throw away, and identical to everyone else's.

Install **[Docker Desktop](https://www.docker.com/products/docker-desktop/)** (macOS, Windows, Linux). On Windows, enable the WSL2 integration in Settings → Resources → WSL Integration so `docker` works from your Ubuntu terminal.

Verify:

```bash
docker --version
docker run --rm hello-world
```

That second command downloaded a tiny image, ran it in a container, printed a message, and deleted the container (`--rm`). You just ran software you didn't install.

**The four words you need today** — [Module 12](12-containers.md) is entirely about what's actually going on underneath, so take these on faith for now:

- **Image** = a frozen filesystem snapshot + a startup command. Like a class.
- **Container** = a running instance of an image. Like an object.
- **Volume** = storage that survives when the container is destroyed. Your database's real data lives here — without one, deleting a container deletes your data.
- **Port mapping** (`-p 5432:5432`) = "make the container's port 5432 reachable at localhost:5432 on my machine."

📖 [Docker's getting-started guide](https://docs.docker.com/get-started/) — the first three pages, no more. Resist the urge to go deeper now; you'll get the real thing in Module 12, once you have an app worth containerizing.

*Alternatives if Docker won't work on your machine:* [Postgres.app](https://postgresapp.com/) (macOS, drag-and-drop simple), the [official Windows installer](https://www.postgresql.org/download/windows/), or a free hosted database from [Neon](https://neon.com/) or [Supabase](https://supabase.com/) — those require no local install at all and give you a connection string.

---

## 0.7 An editor

**[Visual Studio Code](https://code.visualstudio.com/)** — free, dominant, excellent TypeScript support.

After installing, open the Command Palette (`Cmd/Ctrl+Shift+P`), run **"Shell Command: Install 'code' command in PATH"** so you can type `code .` to open the current folder.

Extensions worth having on day one:

| Extension | Why |
|---|---|
| ESLint | Underlines likely bugs and style violations as you type |
| Prettier | Formats code automatically so nobody argues about it |
| Tailwind CSS IntelliSense | Autocompletes Tailwind class names |
| Error Lens | Puts error messages *inline* instead of hiding them in a panel |
| GitLens | Shows who changed each line and when |
| Docker | Browse and manage containers from the sidebar |

WSL users: also install **WSL** (Microsoft) and open your project with `code .` *from inside the Ubuntu terminal* — VS Code will attach to Linux properly.

---

## 0.8 Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Then `cd` into a project folder and run `claude`. Sign in when prompted.

📖 [Claude Code documentation](https://docs.claude.com/en/docs/claude-code/overview) · [Common workflows](https://docs.claude.com/en/docs/claude-code/common-workflows)

Two features you'll use constantly in this course:
- **`CLAUDE.md`** — a file in your repo whose contents are given to Claude automatically every session. This is where the "act as a tutor" instructions live. Copy the [CLAUDE.template.md](CLAUDE.template.md) from this course folder into your project.
- **Plan mode** — ask Claude to plan before it edits. Perfect when you want the *reasoning* rather than the diff.

---

## 0.9 An HTTP client

For poking at APIs by hand, which you'll do a lot in Modules 06–09.

- **[curl](https://curl.se/)** — command line, already installed, universal. Learn it.
- **[Bruno](https://www.usebruno.com/)** — a GUI client that stores requests as plain files in your repo (unlike Postman, which pushes you toward its cloud). Recommended.
- **[HTTPie](https://httpie.io/)** — a friendlier curl. `brew install httpie`.

---

## 0.10 Verify everything

```bash
echo "--- versions ---"
git --version
node --version
npm --version
docker --version
code --version | head -1
claude --version
```

All six should print a version. If one says `command not found`, remember what that means: *the program isn't on your `PATH`.* Either it didn't install, or your shell config needs a line the installer told you to add. Re-read that installer's output before searching the web.

---

## Lab 00 — Prove the whole toolchain works

1. Make a folder and put it under version control:

   ```bash
   mkdir -p ~/projects/taskflow && cd ~/projects/taskflow
   git init
   ```

2. Start a PostgreSQL database in Docker:

   ```bash
   docker run --name taskflow-db \
     -e POSTGRES_PASSWORD=devpassword \
     -e POSTGRES_DB=taskflow \
     -p 5432:5432 \
     -v taskflow-data:/var/lib/postgresql/data \
     -d postgres:17
   ```

   Then `docker ps` to confirm it's running, and:

   ```bash
   docker exec -it taskflow-db psql -U postgres -d taskflow -c "SELECT version();"
   ```

   **Before you run each of these, say what you think it will do.** Then read the flags: what does `-e` do? `-p`? `-v`? `-d`? Ask Claude if you're unsure — but guess first.

3. Write and run a program:

   ```bash
   echo 'console.log("the toolchain works")' > hello.js
   node hello.js
   ```

   Then prove Node can serve HTTP with nothing installed — this is the entire idea of Module 08 in four lines:

   ```bash
   node -e 'require("node:http").createServer((q,s)=>s.end("hello from node")).listen(3000)' &
   curl -v localhost:3000
   ```

   Read the `curl -v` output. You just made an HTTP request to a server you wrote. Kill it with `kill %1` when you're done.

4. Commit it:

   ```bash
   git add .
   git commit -m "chore: verify toolchain"
   ```

5. Stop and remove the database container, then start it again and confirm your data survived (it will — that's what the volume is for):

   ```bash
   docker stop taskflow-db && docker rm taskflow-db
   # ...now re-run the docker run command from step 2
   ```

---

## Understanding Gate

Answer out loud, without looking anything up:

1. You type `node` and get `command not found`. Give two different reasons this could happen.
2. What is the difference between a Docker **image** and a Docker **container**?
3. In `-p 5432:5432`, which number is your machine and which is the container?
4. Why is deleting the container *not* the same as deleting your database data, in the setup above?
5. What does `echo $?` tell you, and when would you care?
6. Node is a "JavaScript runtime." What does that phrase mean, and what does Node give you that a browser doesn't?

```text
Prompt for Claude Code:
I just finished setting up my dev environment (git, fnm/Node, Docker,
VS Code, Claude Code). Quiz me with 6 questions about what each tool does
and how they relate to each other. Ask them ONE AT A TIME and wait for my
answer. Don't accept vague answers — if I say something fuzzy, ask a
follow-up that forces me to be precise. At the end, tell me which concepts
I should re-read before moving on.
```

---

**Next:** [Module 01 — How a Computer Actually Runs Your Code](01-how-computers-run-code.md)
