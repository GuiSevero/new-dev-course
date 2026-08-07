# Appendix A — Prompting Claude Code to Teach You

The default behaviour of an AI coding assistant is to *solve your problem*. For a working engineer that's exactly right. For a learner it's the worst possible outcome: your problem goes away and you don't change.

This appendix is a set of prompt patterns that redirect that energy toward understanding. Steal them, adapt them, keep the ones that work.

---

## Set it up once

Put [CLAUDE.template.md](CLAUDE.template.md) in your project root. It's loaded automatically every session and turns the tutor behaviour on by default, so you don't have to remember to ask.

Also useful:
- **Plan mode** — ask for a plan before any edits. Perfect when you want the reasoning, not the diff.
- Ask it to **explain the codebase** at the start of a session: `/init` generates a `CLAUDE.md` describing the project, which is itself a good summary to read.

📖 [Claude Code docs](https://docs.claude.com/en/docs/claude-code/overview) · [Common workflows](https://docs.claude.com/en/docs/claude-code/common-workflows) · [Memory & CLAUDE.md](https://docs.claude.com/en/docs/claude-code/memory)

---

## Pattern 1 — Explain before implementing

```text
I want to add [feature] to my app. Do NOT write code yet.

1. Explain the approach in plain language: what files change, what each
   piece does, and how data flows through it.
2. Give me two alternative approaches and the tradeoffs of each.
3. Tell me which you'd pick and why.
4. List what could go wrong.

I'll pick an approach and then implement it myself. Stay available for
questions while I do.
```

The habit this builds: *design before typing*. It's the difference between engineering and flailing.

---

## Pattern 2 — Socratic mode

```text
I don't understand [concept]. Instead of explaining it, ask me questions
that lead me to work it out. Start from what I probably already know. If I
give a wrong answer, don't correct it directly — ask a question that makes
the contradiction obvious.
```

Slower than being told. Sticks far longer.

---

## Pattern 3 — Explain at a level below

```text
Explain [thing] to me, but go one layer deeper than the question.

If I ask about a React hook, explain the render cycle underneath it.
If I ask about an ORM method, show me the SQL it generates.
If I ask about a framework feature, show me what it does with the raw
request and response.

I want to understand the machinery, not just the API.
```

This is the highest-value prompt in this appendix. Abstractions are only safe when you know what's underneath.

---

## Pattern 4 — Predict the output

```text
Give me 5 short code snippets related to [topic] where the behaviour is
surprising. Show me the code but NOT the output. I'll predict what happens,
then you tell me if I'm right and explain what actually occurs at runtime.
One at a time.
```

Prediction-then-correction is one of the best-supported learning techniques there is. Being wrong in a low-stakes way is exactly what you want.

---

## Pattern 5 — Adversarial review

```text
Here's my [code]. Act as a hostile reviewer whose job is to find problems.
Assume it's broken and find out how.

Check specifically for: [security / edge cases / performance / accessibility].
Give me a numbered list with file:line references and a concrete failure
scenario for each — actual inputs that produce actual wrong behaviour.

Don't fix anything. I'll fix them and then you re-review.
```

The "don't fix anything" clause is doing the work. Without it you get a diff and learn nothing.

---

## Pattern 6 — Inject a bug

```text
Take my [file] and introduce ONE subtle bug — the kind that passes a casual
read and might pass my tests. Show me the modified file without telling me
what you changed.

I'll find it. If I can't in 10 minutes, give me a hint about which function,
not which line.
```

Trains the skill that matters most for AI-assisted work: **reading code critically**.

---

## Pattern 7 — Quiz me

```text
Quiz me on [topic] from the code we just wrote. Rules:
- One question at a time; wait for my answer
- Start easy, get harder
- If I'm vague, ask a follow-up that forces precision
- Don't accept "it handles X" — make me say HOW
- After 8 questions, tell me which concepts I don't actually understand
  and what to re-read
```

---

## Pattern 8 — The three-way comparison

```text
I'm choosing between [A], [B] and [C] for [purpose].

Give me: what each is actually doing differently under the hood; when each
is the right choice; what each costs; what a team of 5 would regret in a
year. Then tell me what you'd pick for MY situation, which is: [context].
```

Comparison teaches more than description, because it surfaces the tradeoff axis.

---

## Pattern 9 — Debug with me, not for me

```text
I'm getting this error: [paste the FULL error and stack trace].
Here's what I've tried: [list].

Don't tell me the fix. Instead:
1. What is this error message literally saying?
2. Which part of it is the useful clue?
3. What are the 3 most likely causes, most likely first?
4. What's the cheapest experiment to distinguish between them?
```

This teaches a repeatable *procedure*, which is worth vastly more than one fixed bug.

---

## Pattern 10 — Rate my understanding

```text
I'm going to explain [concept] to you as if you were a new teammate. Listen,
then tell me: what I got right, what was imprecise, what was wrong, and what
I left out that matters.

[your explanation]
```

The [Feynman technique](https://fs.blog/feynman-technique/). Explaining is the only reliable way to find the holes in your own understanding, and this gives you an audience that won't politely nod.

---

## Pattern 11 — Archaeology

```text
Walk me through this codebase. Start with the entry point and trace what
happens on a single request. At each hop, tell me which file, which
function, and what decision is made there. Stop after each layer and ask if
I want to go deeper.
```

Great in an unfamiliar repo — including on day one of a new job.

---

## Pattern 12 — Constrain the answer

```text
Answer in under 100 words. No code. No bullet lists longer than 3 items.
```

Verbose explanations feel thorough and teach less. Forcing compression forces the model to identify the actual point.

---

## Anti-patterns

| Don't | Because |
|---|---|
| "Build me a todo app" | You get an app and no understanding. This is the whole thing the course exists to prevent. |
| "Fix this" (pasting an error, nothing else) | You skip diagnosis, which is the skill. |
| Accepting code you can't explain | It compounds. In three weeks you can't debug your own project. |
| "Make it better" | Vague in, vague out. Name the dimension: faster? clearer? safer? |
| Asking before trying | The 30 minutes of being stuck *is* the learning. |
| Never verifying against docs | Models are confidently wrong about library APIs. Check primary sources. |
| Letting it make architecture decisions | It optimizes for the current prompt, not for your system in six months. |

---

## A weekly check

Once a week, build something small — 50 lines, an hour — with **no AI at all**. A script, a small component, a query.

If that feels hard in a way it didn't the week before, you've been outsourcing more than you thought. Adjust.
