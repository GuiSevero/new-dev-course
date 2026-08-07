# Module 04 — Git and Version Control

**Time:** 5–6 hours
**Prereq:** [Module 00](00-setup.md)

---

## Why this module exists

Git is the safety net that makes everything else possible. It's what lets you say "I'll try something risky" without fear, lets five people edit the same codebase without chaos, and lets you answer "when did this break, and what changed?"

It is also famously confusing, for one specific reason: **most people learn Git as a list of commands to memorize instead of as a data structure to understand.** We're going to do it the other way round. Twenty minutes on the model will save you years of `git`-flavored anxiety.

---

## 4.1 The mental model

Git is a **content-addressed graph of snapshots.**

- A **commit** is a full snapshot of your project at a moment, plus metadata (author, date, message) and a pointer to its **parent** commit.
- Each commit has a **hash** (`a3f8b21…`) derived from its contents. Change anything, and you get a different hash. This is why Git history is tamper-evident.
- Commits form a **directed graph** pointing backwards through time.
- A **branch** is not a copy of anything. It is a *sticky note with a commit hash on it.* That's the whole thing. Creating a branch is instant because it writes 41 bytes to a file.
- `HEAD` is another sticky note saying "which branch am I currently on."

```
        A ─── B ─── C ─── D   ← main
                     \
                      E ─── F   ← feature/add-tasks  ← HEAD (you are here)
```

Once you believe "a branch is a pointer," `merge`, `rebase`, and `reset` stop being magic and become obvious operations on a graph.

### The three areas

```
 working directory  →  staging area (index)  →  repository (.git)
   your edits            git add                  git commit
```

- **Working directory** — the files you actually edit.
- **Staging area** — a scratch space where you assemble the *next* commit. This is Git's most distinctive idea and the one beginners skip. It lets you make five changes and commit them as three logically separate commits.
- **Repository** — the permanent, immutable-ish history in `.git/`.

📖 [Pro Git](https://git-scm.com/book/en/v2), chapters 1–3 — free, official, excellent. Chapter 10 ("Git Internals") is optional but genuinely enlightening.
🎮 [Learn Git Branching](https://learngitbranching.js.org/) — **do this one.** Interactive, visual, ~2 hours, and it's the single best Git resource that exists. Complete "Main" and "Remote" tracks.
🎮 [Oh My Git!](https://ohmygit.org/) — a card game that teaches Git. Surprisingly good.
📖 [Think Like (a) Git](https://think-like-a-git.net/) — for when you're stuck on the graph model.

---

## 4.2 The everyday loop

```bash
git status                 # ← run this constantly. It tells you where you are.
git add src/tasks.ts       # stage a specific file
git add -p                 # stage HUNK BY HUNK, interactively — learn this one
git commit -m "feat: add task completion"
git log --oneline --graph --all --decorate   # see the graph
git diff                   # unstaged changes
git diff --staged          # what you're about to commit
```

`git status` is not a formality. It answers "what branch am I on, what's changed, what's staged, am I ahead of the remote" — the four questions behind most Git confusion. Run it before and after everything for the first month.

`git add -p` is transformative once you get it: it walks you through each chunk of your changes and asks whether to include it. It forces you to *read your own diff before committing*, which catches an astonishing number of bugs and stray `console.log`s.

### Writing commit messages

A commit message explains **why**, since the diff already shows what. Use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add due dates to tasks
fix: prevent duplicate task titles within a list
refactor: extract task validation into its own module
docs: explain the auth flow in the README
test: cover the empty-list case in summarize()
chore: bump drizzle-orm to 0.44
```

Format: `type: short imperative summary`, ideally under 72 characters, and a body if there's a *why* worth recording.

```
fix: prevent duplicate task titles within a list

Users were double-clicking Save and getting two identical tasks. Adds a
unique constraint on (list_id, title) plus a friendlier 409 response.

Closes TASK-42
```

**One logical change per commit.** A commit that says "various fixes" and touches 30 files is useless — it can't be reverted, can't be reviewed, and can't be understood in six months.

---

## 4.3 Branching and merging

```bash
git switch -c feature/task-filters    # create + switch (modern; `git checkout -b` is the old form)
# ...work, commit, commit...
git switch main
git merge feature/task-filters
```

**Merge** creates a new commit with two parents, preserving the exact history of what happened.

```
A ─ B ─ C ─────── M   ← main   (M has parents C and F)
     \           /
      D ─ E ─ F
```

**Rebase** rewrites your commits so they appear to have been made on top of the latest `main` — producing a straight line.

```bash
git switch feature/task-filters
git rebase main
```

```
A ─ B ─ C ─ D' ─ E' ─ F'   ← linear, but D' E' F' are NEW commits (different hashes)
```

**When to use which:**
- **Rebase your own unpushed feature branch** onto main to keep history clean and avoid pointless merge commits.
- **Merge** when integrating a finished branch into a shared one, or when the history of "these things happened in parallel" matters.
- 🚫 **Never rebase a branch other people are working on.** You'd be rewriting commits they already have, and their next pull becomes a disaster. The rule: *don't rewrite public history.*

### Conflicts

A conflict happens when two branches changed the same lines. Git can't guess; it asks you.

```
<<<<<<< HEAD
const PAGE_SIZE = 20;
=======
const PAGE_SIZE = 50;
>>>>>>> feature/task-filters
```

Top is what's on your current branch, bottom is what's coming in. **Delete the markers and write the correct final code** — which is sometimes neither side. Then:

```bash
git add src/config.ts
git commit           # (or `git rebase --continue` if you were rebasing)
```

Conflicts are not errors, and they are not your fault. They're a normal consequence of parallel work. Fear of conflicts is what makes people avoid branching — get over it early by causing a few on purpose in the lab.

---

## 4.4 Remotes and GitHub

A **remote** is another copy of the repository, usually on a server. `origin` is the conventional name for "the one I cloned from."

```bash
git remote add origin git@github.com:you/taskflow.git
git push -u origin main       # -u sets up tracking so future `git push` needs no args
git pull                      # fetch + merge from the remote
git fetch                     # download remote changes WITHOUT touching your files
git clone git@github.com:you/taskflow.git
```

`git fetch` then `git log origin/main` is the safe way to see what changed upstream before you integrate it.

### SSH keys

HTTPS auth means typing a token constantly. Set up SSH once:

```bash
ssh-keygen -t ed25519 -C "you@example.com"     # press Enter through the prompts
cat ~/.ssh/id_ed25519.pub                       # copy this
```

Paste it into [GitHub → Settings → SSH and GPG keys](https://github.com/settings/keys). Test with `ssh -T git@github.com`.

📖 [GitHub's SSH setup guide](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)

### Pull requests

A **PR** is a request to merge one branch into another, plus a place to discuss the change. It's the fundamental unit of collaboration in modern software. Even solo, use PRs — they force you to read your own diff.

The [GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow):

1. `git switch -c feature/x` off `main`
2. Commit small, coherent changes
3. `git push -u origin feature/x`
4. Open a PR; describe **what changed and why**, and how to test it
5. CI runs (Modules 12–13); reviewers comment
6. Address feedback with new commits
7. Merge (squash-merge is common: the branch's commits collapse into one on `main`)
8. Delete the branch

The `gh` CLI ([install](https://cli.github.com/)) makes this fast:

```bash
gh auth login
gh repo create taskflow --private --source=. --push
gh pr create --fill
gh pr view --web
gh pr checks
```

---

## 4.5 Undoing things — the actual reason to trust Git

Print this section out.

```bash
# Discard unstaged changes to a file (DESTRUCTIVE — the edits are gone)
git restore src/tasks.ts

# Unstage a file but keep the edits
git restore --staged src/tasks.ts

# Amend the last commit (fix the message, or add a forgotten file)
git commit --amend

# Undo the last commit but KEEP the changes as unstaged edits — the safe default
git reset --soft HEAD~1

# Undo the last commit and THROW AWAY the changes (destructive)
git reset --hard HEAD~1

# Create a NEW commit that undoes an old one — safe on shared branches
git revert <hash>

# Park your work temporarily
git stash
git stash pop

# See the state of a file at an old commit
git show <hash>:src/tasks.ts

# Who changed this line, and in which commit?
git blame src/tasks.ts

# Find the commit that introduced a bug, by binary search
git bisect start; git bisect bad; git bisect good <old-hash>
```

**`git reflog` is the undo button for your undo button.** It records every position `HEAD` has been in, including commits you "lost" with a bad reset or rebase:

```bash
git reflog                    # find the hash you were at before the disaster
git reset --hard HEAD@{3}     # go back there
```

> **The one true rule:** if a commit exists, it is almost impossible to truly lose it for ~30 days (until garbage collection). Uncommitted work, on the other hand, is genuinely fragile. **Commit early and often.** You can always tidy history later.

`revert` vs `reset`: `reset` rewrites history (fine locally, dangerous when shared). `revert` adds a new commit that undoes the old one (always safe). On `main`, use `revert`.

---

## 4.6 `.gitignore` and secrets

Some files must never be committed: dependencies (huge, reproducible), build output (generated), and **secrets** (catastrophic).

Create `.gitignore` at the repo root:

```gitignore
node_modules/
dist/
build/
.env
.env.local
*.log
.DS_Store
coverage/
.vite/
```

> ### Read this twice
> **A secret committed to git is compromised, permanently.** Deleting it in a later commit does not help — it's still in history, still on GitHub, and bots scan public repos for API keys within *seconds* of a push.
>
> If it happens: **rotate the secret immediately** (revoke the old key, generate a new one). That's the actual fix. Scrubbing history with [`git filter-repo`](https://github.com/newren/git-filter-repo) is secondary cleanup, not a substitute.
>
> Commit a `.env.example` with the *keys* and dummy values so teammates know what config they need, and gitignore the real `.env`.

Consider installing [gitleaks](https://github.com/gitleaks/gitleaks) as a pre-commit hook so this can't happen to you.

---

## Lab 04 — Cause and fix real problems

Set up your actual project repo, then deliberately create every situation you're afraid of.

```bash
mkdir -p ~/projects/taskflow && cd ~/projects/taskflow
git init
printf 'node_modules/\ndist/\n.env\n' > .gitignore
echo "# TaskFlow" > README.md
git add . && git commit -m "chore: initial commit"
gh repo create taskflow --private --source=. --push   # or create it on github.com and add the remote
```

Now, one at a time:

1. **Feature branch → PR.** Branch, add a `docs/architecture.md` sketching the three-tier diagram from the course README, push, open a PR, review your own diff on GitHub, merge, delete the branch, pull `main` locally.

2. **Cause a merge conflict on purpose.** Create two branches from `main` that each change the same line of `README.md` differently. Merge one, then merge the other. Resolve it. Do this twice so it stops being scary.

3. **Rebase.** Branch, make three commits. Meanwhile add a commit to `main`. Rebase your branch onto `main`. Run `git log --graph --all --oneline` before and after and describe the difference in your own words.

4. **Interactive rebase.** `git rebase -i HEAD~3` — squash your three commits into one and rewrite the message. (You'll need this before every real PR.)

5. **Break it and recover.** Make a commit. Then `git reset --hard HEAD~1`. Panic appropriately. Then recover it with `git reflog`. **Do not skip this step** — the confidence it produces is worth the whole module.

6. **Stash.** Start editing a file, then get "urgently interrupted": stash, switch branches, come back, pop.

7. **Bisect.** Make 8 commits where commit #5 introduces a bug (e.g. changes a function to return the wrong value). Use `git bisect` to find it without looking at the log.

8. **Archaeology.** `git log --oneline`, `git show <hash>`, `git blame README.md`. Answer: who changed line 1, when, and in which commit?

---

## Understanding Gate

1. What *is* a branch, physically, in the `.git` directory?
2. What's the difference between `git fetch` and `git pull`?
3. When must you use `revert` instead of `reset`?
4. You committed an API key an hour ago and pushed it. What do you do — in order?
5. Why is rebasing a shared branch antisocial?
6. What does the staging area buy you that "commit everything" doesn't?
7. You made a change and can't find it. What's the first command you run?

```text
Prompt for Claude Code:
Act as a Git instructor. Describe 8 realistic "oh no" scenarios one at a
time — committed to the wrong branch, need to undo a pushed commit,
accidentally deleted a branch, rebase went wrong mid-way, committed a
secret, need to split one commit into two, etc.

For each: I'll tell you the commands I'd run. Don't give me the answer
first. Tell me if my approach would work, whether it's destructive, and
what a safer alternative would be.
```

---

**Next:** [Module 05 — Project Organization](05-project-organization.md)
