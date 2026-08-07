# Module 01 — How a Computer Actually Runs Your Code

**Time:** ~4 hours
**Prereq:** [Module 00](00-setup.md)

---

## Why this module exists

You are about to spend 90 hours writing text files and asking a computer to do things with them. If you don't have a mental model of what happens between "I saved the file" and "something happened on screen", every bug will feel like sorcery.

This is the only module with no app code in it. It's the foundation everything else stands on. **Don't skip it** — this is precisely the layer that AI-assisted developers most often lack, and it's the layer that separates someone who can debug from someone who can only re-prompt.

---

## 1.1 The stack of abstractions

Everything in computing is layers, each pretending the layer below is simpler than it is.

```
   Your TypeScript source code           ← text you write
        ↓ compiled/transpiled
   JavaScript                            ← what the runtime understands
        ↓ parsed & JIT-compiled by V8
   Machine code (x86-64 / ARM64)         ← numbers the CPU executes
        ↓ scheduled by
   Operating system (kernel)             ← manages memory, files, network, CPU time
        ↓ runs on
   CPU + RAM + disk + network card       ← physical electronics
```

Every layer exists because the one below it is too tedious to use directly. And every layer **leaks** — occasionally a lower-layer detail (memory limits, network latency, integer precision) punches through and ruins your day. Knowing the layers is knowing where to look.

📺 Watch: [Crash Course Computer Science](https://www.youtube.com/playlist?list=PL8dPuuaLjXtNlUrzyH5r6jN9ulIgZBpdo), episodes 1–12. Fast, animated, genuinely good.
📖 Read: [What Every Programmer Should Know About Memory (intro only)](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) — read the first 5 pages, skip the rest. It's dense; the point is to see how deep the rabbit hole goes.

---

## 1.2 Bits, bytes, and why `0.1 + 0.2 !== 0.3`

A computer stores everything — text, images, your todo list — as numbers in binary. 8 bits = 1 byte.

- Integers are exact.
- **Decimals are not.** They're stored as [IEEE 754 floating point](https://floating-point-gui.de/): a sign, an exponent, and a fraction, in a fixed number of bits. `0.1` is not representable exactly in binary, exactly as `1/3` is not representable exactly in decimal.

Try it right now:

```bash
node -e 'console.log(0.1 + 0.2); console.log(0.1 + 0.2 === 0.3)'
```

```
0.30000000000000004
false
```

This is not a JavaScript bug. It happens in Python, Java, C, and your CPU. **Consequence for you:** never store money as a floating-point number. Store cents as an integer, or use a decimal type. Postgres has `numeric` for exactly this reason (Module 07).

- **Text** is bytes too. [Unicode](https://home.unicode.org/) assigns every character a number ("code point"); [UTF-8](https://en.wikipedia.org/wiki/UTF-8) encodes those numbers as 1–4 bytes. This is why `"héllo".length` can surprise you, and why emoji sometimes break string slicing.

📖 [The Absolute Minimum Every Developer Must Know About Unicode](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/) — Joel Spolsky, still the best explanation, 20 years on.

---

## 1.3 Programs, processes, and the operating system

- A **program** is a file on disk. Inert.
- A **process** is a program that's been loaded into memory and is being executed. It has its own memory space, its own file handles, its own ID (PID).
- The **operating system kernel** decides which process gets the CPU, for how long, and stops processes from reading each other's memory.

When you run `node server.js`, the OS creates a process, loads the Node binary into it, and Node then reads your file.

Try it:

```bash
node -e 'console.log("PID:", process.pid); setTimeout(()=>{}, 60_000)' &
ps aux | grep node        # see your process in the OS process table
```

**Concepts that will matter later:**
- **Ports.** A machine has one IP address but 65,535 TCP ports. A server process "listens" on one (your API will use 3000; Postgres uses 5432). Only one process can hold a port at a time — hence the extremely common error `EADDRINUSE: address already in use`, which means *you already have a server running, probably in another terminal you forgot about.*
- **Environment variables.** Key-value strings the OS hands to a process at startup. This is how you configure software without editing it — database passwords, API keys, `NODE_ENV=production`. Run `printenv` to see yours.
- **Signals.** `Ctrl+C` sends `SIGINT` to the foreground process, politely asking it to stop. `kill -9` sends `SIGKILL`, which cannot be caught or ignored.

---

## 1.4 Stack and heap

Two regions of a process's memory, with fundamentally different rules.

### The stack

Fast, small, automatic, and strictly ordered. Every time you call a function, a **stack frame** is pushed on: the function's arguments, its local variables, and where to return to when it's done. When the function returns, the frame pops off and its memory is instantly reclaimed.

```js
function a() { const x = 1; return b(x); }   //  frame for a() pushed
function b(y) { const z = 2; return c(y+z); } //  frame for b() pushed on top
function c(n) { return n * 10; }              //  frame for c() pushed on top
a();                                          //  frames pop as each returns
```

Because it's a stack, it's finite and it has an order. Which gives you the most famous error in programming:

```bash
node -e 'function boom(){ return boom() } boom()'
# RangeError: Maximum call stack size exceeded
```

Infinite recursion → frames pushed forever → stack overflow. (Yes, [that website](https://stackoverflow.com/) is named after this.)

The stack is also what a **stack trace** is: a printout of the frames that were live when an error occurred. Reading stack traces top-to-bottom tells you the exact call path that got you into trouble. This is the single most useful debugging skill there is.

### The heap

Large, unordered, manually or automatically managed. Anything whose size isn't known at compile time or whose lifetime outlives the function that made it lives here: objects, arrays, strings, closures.

```js
function makeUser() {
  const user = { name: "Ada" };  // the OBJECT lives on the heap
  return user;                    // the REFERENCE (a pointer) was on the stack
}
const u = makeUser();             // object survives — something still points at it
```

This is why the following behaves the way it does — and it trips up every beginner:

```js
const a = { count: 1 };
const b = a;        // copies the REFERENCE, not the object
b.count = 99;
console.log(a.count);  // 99  — a and b point at the SAME heap object

let x = 1;
let y = x;          // primitives are copied by VALUE
y = 99;
console.log(x);     // 1
```

**Value vs reference.** Burn this in. It explains React re-render bugs, "why did my array change when I only modified the copy", and half of all state management confusion.

### Garbage collection

In C, you free heap memory yourself, and forgetting is a memory leak while doing it twice is a crash. JavaScript has a **garbage collector**: periodically, the runtime finds every object still reachable from the running program and frees everything else.

You mostly don't think about it. When you do, it's because:
- A **memory leak** happens anyway — you kept a reference you forgot about (an event listener never removed, an ever-growing cache, a global array you keep pushing into). It's reachable, so the GC keeps it.
- GC pauses cost time, which matters at high scale.

📖 [MDN: Memory management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_management) · [V8's Trash Talk](https://v8.dev/blog/trash-talk) (how a real GC works)

---

## 1.5 How languages work: compiled, interpreted, and in between

**Compiled** (C, Rust, Go): a compiler translates your whole source to machine code *before* you run it. Fast to run, slower to iterate, errors caught at compile time.

**Interpreted** (classic Python, Ruby): a program reads your source and executes it statement by statement, every time. Slower, but instant feedback.

**JIT-compiled** (JavaScript, Java, C#): a hybrid. The runtime starts interpreting immediately, watches which functions run *hot*, and compiles those to optimized machine code on the fly. This is why modern JavaScript is thousands of times faster than 1998 JavaScript. V8 (in Node, Bun and Chrome) does this.

**Transpiled** — a source-to-source translation, not a compilation to machine code. This is what happens to your TypeScript: types are checked, then **erased**, and plain JavaScript comes out the other side.

> **Critical point about TypeScript that beginners miss:**
> TypeScript types do not exist at runtime. None of them. They are checked while you write, then deleted. If a malicious client POSTs `{ "title": 12345 }` to an endpoint you typed as `{ title: string }`, TypeScript will not stop it — the type was erased before the server ever started.
> This is *why* Module 08 validates every request body with Zod: you need a **runtime** check at the boundary. Types protect you from yourself; validation protects you from the world.

Play with it: [TypeScript Playground](https://www.typescriptlang.org/play) — write TS on the left, watch the emitted JS on the right. Notice the types vanish.

---

## 1.6 Concurrency and the event loop

Your server will handle many users at once on a single thread. Understanding how is essential.

**Blocking vs non-blocking.** Reading a file from disk takes ~10 milliseconds. In CPU terms that's an eternity — tens of millions of instructions. Two strategies:

1. **Threads** (Java, Go, Python): one thread per request; when a thread blocks on I/O, the OS runs another. Simple to write, but threads cost memory and coordinating shared data between them is where the hardest bugs in software live (race conditions, deadlocks).
2. **Event loop** (JavaScript, and hence you): a *single* thread. When you ask for I/O, you hand the runtime a callback and immediately move on. When the data arrives, the runtime queues your callback and runs it when the thread is free.

```js
console.log("1");
setTimeout(() => console.log("3"), 0);   // queued, runs when the stack is empty
Promise.resolve().then(() => console.log("2b"));  // microtask — higher priority
console.log("2a");
// prints: 1, 2a, 2b, 3
```

Run that. Then explain the order to yourself. If you can, you understand the event loop better than a lot of working developers.

**The consequence:** if you write a slow synchronous loop, *nothing else in your entire server runs* until it finishes. Every user waits. This is the one thing you must never do on the event loop.

**async/await** is syntax over this machinery. `await` doesn't block the thread; it suspends *your function* and returns control to the event loop, resuming when the promise settles.

📺 [Philip Roberts — What the heck is the event loop anyway?](https://www.youtube.com/watch?v=8aGhZQkoFbQ) — 26 minutes, a classic, watch it.
📺 [Jake Archibald — In The Loop](https://www.youtube.com/watch?v=cCOL7MC4Pl0) — the sequel, covers microtasks vs macrotasks.
🔧 [Loupe](http://latentflip.com/loupe/) — visualize the event loop as your code runs.

---

## 1.7 What actually happens when you load a web page

Tie it all together. You type `https://taskflow.app` and press Enter:

1. **DNS** — the browser asks a name server: what IP address is `taskflow.app`? Gets back e.g. `203.0.113.42`.
2. **TCP** — opens a connection to that IP on port 443, via a three-way handshake (SYN → SYN-ACK → ACK).
3. **TLS** — negotiates encryption; the server proves its identity with a certificate signed by a trusted authority. This is the S in HTTPS.
4. **HTTP request** — sends `GET / HTTP/1.1` plus headers over the encrypted connection.
5. **Server** — a process listening on port 443 receives it, runs your code, maybe queries a database, builds a response.
6. **HTTP response** — status code, headers, and a body of HTML comes back.
7. **Parse** — the browser builds the **DOM** (a tree of the HTML) and the **CSSOM** (a tree of the styles), and fetches every `<script>`, `<link>`, and `<img>` it finds — each one a whole new request.
8. **Execute** — JavaScript runs and may modify the DOM.
9. **Render** — layout (where does everything go?) → paint (what color is each pixel?) → composite (stack the layers) → photons.

Module 06 unpacks steps 1–6 in detail. For now, just internalize: **the network is in the middle, and it is slow and unreliable.** Every design decision in web development is downstream of that fact.

📖 [High Performance Browser Networking](https://hpbn.co/) — free online book by Ilya Grigorik. Chapters 1–2 are the definitive explanation.
🎨 [How DNS works](https://howdns.works/) — a comic. Genuinely the clearest DNS explainer.

---

## Lab 01 — Observe each layer

No app code. Just look at the machine.

**A. Floating point**
```bash
node -e 'console.log(0.1 + 0.2, 0.1 + 0.2 === 0.3, (0.1+0.2).toFixed(20))'
```
Then figure out and write down: how *should* you store a price of $19.99?

**B. Blow the stack**
```bash
node -e 'let d=0; function f(){ d++; return f() } try{ f() }catch(e){ console.log(e.constructor.name, "depth:", d) }'
```
Note the depth. Run it again — is it the same? Why might it not be?

**C. Reference vs value**
Write a file `refs.js` that demonstrates the `a`/`b` object aliasing above, then fix it so `b` is an independent copy. Find at least two ways to copy (`{...a}` and `structuredClone(a)`), and then find a case where one of them is *not* enough (hint: nested objects).

**D. Block the event loop**
```js
// blocking.js
const start = Date.now();
setTimeout(() => console.log("timer fired after", Date.now() - start, "ms"), 100);
while (Date.now() - start < 3000) {} // busy-wait 3 seconds
console.log("loop done");
```
Run it. The timer asked for 100ms. When did it actually fire? **Explain why in one sentence.** This is the single most important runtime lesson in this module.

**E. Watch the network**
Open any website, press F12, go to the **Network** tab, and reload. Look at: the number of requests, the waterfall of timings, and the DNS/TCP/TLS/TTFB breakdown when you click a request. You are looking at steps 1–6 above, measured.

---

## Understanding Gate

1. What is the difference between a program and a process?
2. Where does an object live in memory, and where does the variable that names it live?
3. Why does `0.1 + 0.2 !== 0.3`, and what should you use for money?
4. TypeScript says a value is a `string`. Give a scenario where, at runtime, it isn't.
5. What does "blocking the event loop" mean, and why is it catastrophic on a server?
6. Explain a stack trace to someone who's never seen one.
7. What's a memory leak in a garbage-collected language, if the GC frees everything unused?

```text
Prompt for Claude Code:
I'm learning the fundamentals of how code executes. Give me five short
JavaScript snippets — each 3–8 lines — where the output is surprising for a
reason connected to: (a) the event loop, (b) value vs reference, (c)
floating point, (d) closures capturing variables, (e) the call stack.

Show me each snippet WITHOUT the output. I'll predict what it prints, then
you tell me if I'm right and explain what actually happens at the runtime
level. Do them one at a time.
```

---

**Next:** [Module 02 — Programming Fundamentals with TypeScript](02-programming-fundamentals.md)
