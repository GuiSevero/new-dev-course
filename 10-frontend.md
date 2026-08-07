# Module 10 — Building the Frontend

**Time:** ~12 hours. The largest module.
**Prereq:** [Module 02](02-programming-fundamentals.md), [Module 06](06-client-server-http.md), [Module 08](08-backend-api.md)

---

## Why this module exists

The frontend is where most beginners start, and it's where they build the deepest misconceptions — because a UI framework hides an enormous amount, and it's easy to make things appear on screen without knowing why.

We're arriving here last, on purpose. You now know what an HTTP request is, what a JSON response contains, and what's in your database. React is going to look much smaller than it would have in week one.

---

## 10.1 The platform underneath: HTML, CSS, the DOM

Before React, understand what React produces.

**HTML** is a tree of elements describing *structure and meaning*. Use semantic tags (`<button>`, `<nav>`, `<main>`, `<label>`, `<ul>`) — they carry behaviour and accessibility for free. A `<div onClick>` is not a button: it isn't focusable, doesn't respond to Enter or Space, and is invisible to a screen reader.

**CSS** styles it. Three things to actually understand (the rest you can look up):
- **The box model** — content, padding, border, margin. Everything is a box.
- **Flexbox and Grid** — how modern layout works. [Flexbox Froggy](https://flexboxfroggy.com/) and [Grid Garden](https://cssgridgarden.com/) teach both in about 40 minutes of games. Do them.
- **The cascade & specificity** — which rule wins when several apply.

**The DOM** is the browser's live, in-memory tree of objects representing the page. JavaScript modifies the DOM; the browser re-renders. Open the console on any page and run:

```js
document.querySelectorAll("a").length
document.querySelector("h1").textContent = "I changed this"
```

You just did, by hand, what React does for you. **React's entire job is: keep the DOM in sync with your data, so you never write that second line again.**

📖 [MDN: Learn web development](https://developer.mozilla.org/en-US/docs/Learn_web_development) — the best free curriculum for the platform
🎮 [Flexbox Froggy](https://flexboxfroggy.com/) · [Grid Garden](https://cssgridgarden.com/)
📖 [web.dev: Learn CSS](https://web.dev/learn/css) · [Learn Accessibility](https://web.dev/learn/accessibility)

---

## 10.2 Vite

A **build tool**. Browsers can't run TypeScript, JSX, or (efficiently) hundreds of small module files. Vite:

- **In dev:** serves your files as native ES modules with instant Hot Module Replacement — you save, the browser updates the changed module without a reload or losing state.
- **In build:** bundles, minifies, tree-shakes (drops unused code), and fingerprints filenames (`index-a3f8b21.js`) so they can be cached forever (Module 06's `Cache-Control: immutable`).

```bash
cd apps
npm create vite@latest web -- --template react-ts
cd web && npm install && npm run dev
```

Then **look at what it produced** before touching it: `index.html` (the single HTML file — note there's almost nothing in it), `src/main.tsx` (the entry point that mounts React into that empty div), `vite.config.ts`.

Run `npm run build` and inspect `dist/`. Open the built `index.html`. Notice the hashed filenames. **That's what actually gets deployed** — everything else is development machinery.

📖 [Vite guide](https://vite.dev/guide/) · [Why Vite](https://vite.dev/guide/why)

---

## 10.3 React's mental model

Three ideas. Everything else follows.

### 1. UI is a function of state

```
UI = f(state)
```

You never say "add a row to the table." You change the data and describe what the UI looks like *for that data*. React figures out the DOM operations.

```tsx
function TaskList({ tasks }: { tasks: Task[] }) {
  return (
    <ul>
      {tasks.map(task => <li key={task.id}>{task.title}</li>)}
    </ul>
  );
}
```

**`key` is not optional.** It tells React which item is which across renders. Without stable keys (never the array index, if the list can reorder or have items removed) you get bizarre bugs: checkbox states attaching to the wrong row, inputs losing focus, animations firing on the wrong element.

### 2. Components compose

A component is a function returning JSX. **JSX is not HTML** — it's syntax that compiles to `React.createElement(...)` calls. Hence `className` instead of `class`, `onClick` instead of `onclick`, `{}` for JavaScript expressions.

Props flow **down**; events flow **up** via callbacks. Data has one direction, which is what makes a large UI tractable.

### 3. Render → reconcile → commit

When state changes, React re-runs your component function, producing a new tree description. It **diffs** that against the previous one and applies only the minimal real DOM changes.

**Consequence that trips up everyone:** your component function runs *many times*. It must be a pure function of props and state — no side effects, no mutation, no DOM manipulation in the body. Side effects go in event handlers or `useEffect`.

📖 [react.dev/learn](https://react.dev/learn) — the official tutorial, genuinely excellent, **work through all of it**
📖 [Thinking in React](https://react.dev/learn/thinking-in-react) — read this twice
📖 [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) — read it *before* you write your first `useEffect`. It will save you from the most common category of React bug.

---

## 10.4 Hooks

```tsx
const [count, setCount] = useState(0);           // local state
useEffect(() => { /* sync with something external */ }, [deps]);
const value = useMemo(() => expensive(a, b), [a, b]);   // cache a computed value
const fn = useCallback(() => doThing(id), [id]);        // cache a function identity
const ref = useRef<HTMLInputElement>(null);             // a mutable box that doesn't cause re-renders
const theme = useContext(ThemeContext);                 // read shared context
```

**Rules of Hooks:** only call them at the top level of a component (never in a loop, condition, or nested function), and only from components or other hooks. React tracks hooks *by call order*; a conditional hook shifts the order and corrupts state. The ESLint plugin enforces this — install it and obey it.

### `useEffect` is not "run this when the component loads"

It's for **synchronizing with something outside React** — a subscription, a timer, a browser API, an imperative library. If you're reaching for `useEffect` to fetch data or to compute a value from other state, there's almost always a better answer:

```tsx
// ❌ derived state in an effect: an extra render, and a chance to go stale
const [visible, setVisible] = useState<Task[]>([]);
useEffect(() => { setVisible(tasks.filter(t => t.status === filter)) }, [tasks, filter]);

// ✅ just compute it during render
const visible = tasks.filter(t => t.status === filter);
```

```tsx
// ❌ data fetching by hand: no caching, no dedup, races, no retry, no loading/error discipline
useEffect(() => { fetch("/api/tasks").then(r => r.json()).then(setTasks) }, []);

// ✅ use a data-fetching library (next section)
```

### The stale closure

Module 02's closures, in the wild:

```tsx
useEffect(() => {
  const id = setInterval(() => console.log(count), 1000);  // captures `count` from THIS render
  return () => clearInterval(id);
}, []);   // ← empty deps: the effect never re-runs, so it logs 0 forever
```

The fix is either honest dependencies or the updater form `setCount(c => c + 1)`. Recognizing this shape is a genuine milestone in learning React.

---

## 10.5 State: the four kinds

Most React confusion comes from treating all state the same. It isn't.

| Kind | Example | Where it goes |
|---|---|---|
| **Local UI state** | is this dropdown open, what's in this input | `useState` in the component |
| **Shared UI state** | theme, sidebar collapsed, current modal | Context, or a store like [Zustand](https://zustand.docs.pmnd.rs/) |
| **URL state** | current filter, page number, selected task | The URL itself, via a router |
| **Server state** | your tasks, the current user | **[TanStack Query](https://tanstack.com/query/latest)** — never `useState` |

> **Server state is not your state.** It lives in the database. Your copy is a *cache* that can be stale, that several components need, that must be invalidated when you mutate, and that needs loading/error/refetch handling. Trying to manage that with `useState` + `useEffect` means hand-rolling a cache library, badly. This distinction is the single biggest leap in frontend competence.

**URL state deserves a mention too:** if the filter lives in `useState`, the user can't bookmark it, share it, or use the back button. Putting it in the query string is usually the right call.

---

## 10.6 TanStack Query

```bash
npm install @tanstack/react-query
```

```tsx
// src/api/tasks.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

const API = import.meta.env.VITE_API_URL;

async function request<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${API}${path}`, {
    ...init,
    credentials: "include",                     // ← send the session cookie (Modules 06 & 09)
    headers: { "Content-Type": "application/json", ...init?.headers },
  });
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new Error(body?.error?.message ?? `Request failed: ${res.status}`);
  }
  return res.status === 204 ? (undefined as T) : res.json();
}

export function useTasks(listId: string, status?: TaskStatus) {
  return useQuery({
    queryKey: ["tasks", listId, status],       // the cache key — changes here trigger a refetch
    queryFn: () => request<Task[]>(`/api/tasks?listId=${listId}${status ? `&status=${status}` : ""}`),
  });
}

export function useCreateTask() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (input: CreateTaskInput) =>
      request<Task>("/api/tasks", { method: "POST", body: JSON.stringify(input) }),
    onSuccess: (_task, input) => {
      // Tell the cache this data is stale; Query refetches what's on screen.
      qc.invalidateQueries({ queryKey: ["tasks", input.listId] });
    },
  });
}
```

```tsx
function TaskList({ listId }: { listId: string }) {
  const { data: tasks, isPending, isError, error } = useTasks(listId);

  if (isPending) return <Spinner />;
  if (isError) return <ErrorMessage error={error} />;      // ← handle this. Always.
  if (tasks.length === 0) return <EmptyState />;           // ← and this.

  return <ul>{tasks.map(t => <TaskRow key={t.id} task={t} />)}</ul>;
}
```

You get for free: caching, deduplication of identical in-flight requests, background refetching, retry with backoff, refetch on window focus, and race-condition safety. Writing that by hand is hundreds of lines and you'd get the races wrong.

**Every screen has four states: loading, error, empty, and content.** Beginners build the fourth and ship. Build all four, every time — the empty state is often the most important screen a new user sees.

📖 [TanStack Query docs](https://tanstack.com/query/latest/docs/framework/react/overview) · [Practical React Query](https://tkdodo.eu/blog/practical-react-query) — TkDodo's blog series is the best writing on this library

---

## 10.7 Tailwind CSS and Radix UI

**Tailwind** is utility-first CSS: compose styles from small single-purpose classes in your markup instead of writing separate stylesheets.

```tsx
<button className="rounded-md bg-blue-600 px-4 py-2 text-sm font-medium text-white
                   hover:bg-blue-700 focus-visible:ring-2 focus-visible:ring-blue-400
                   disabled:opacity-50">
  Add task
</button>
```

It looks wrong at first. The arguments for it: styles are colocated with markup (no hunting for what `.card-header--compact` does), there's no dead CSS, no naming bikeshed, and the design tokens (spacing, colors) are constrained by default so your UI stays consistent.

When markup gets repetitive, **that's the signal to extract a component** — not to write a CSS class.

📖 [Tailwind docs](https://tailwindcss.com/docs) · [Install with Vite](https://tailwindcss.com/docs/installation/using-vite)

**Radix UI Primitives** are unstyled, accessible components: dialog, dropdown, checkbox, tooltip, popover. They handle the genuinely hard parts — focus trapping, keyboard navigation, ARIA attributes, screen-reader announcements, click-outside, scroll locking — and give you zero styling opinions.

```tsx
import * as Dialog from "@radix-ui/react-dialog";

<Dialog.Root>
  <Dialog.Trigger className="rounded bg-blue-600 px-3 py-1.5 text-white">New task</Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay className="fixed inset-0 bg-black/50" />
    <Dialog.Content className="fixed left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2
                               rounded-lg bg-white p-6 shadow-xl">
      <Dialog.Title className="text-lg font-semibold">Create a task</Dialog.Title>
      <TaskForm />
      <Dialog.Close className="absolute right-4 top-4">✕</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

> **Do not build your own modal, dropdown, or combobox.** Accessible versions of these are *much* harder than they look — focus management alone is a genuine engineering problem, and getting it wrong locks out keyboard and screen-reader users entirely. Use primitives.

📖 [Radix Primitives](https://www.radix-ui.com/primitives/docs/overview/introduction) · [shadcn/ui](https://ui.shadcn.com/) — Radix + Tailwind, pre-styled, copied into your repo rather than installed as a dependency. Excellent, and worth reading the source of the components you use.

---

## 10.8 Forms: React Hook Form + Zod

```bash
npm install react-hook-form zod @hookform/resolvers
```

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

// Put this in packages/shared and the API validates against the SAME schema.
export const CreateTaskSchema = z.object({
  title: z.string().min(1, "Title is required").max(200, "Too long"),
  description: z.string().max(2000).optional(),
  dueDate: z.coerce.date().min(new Date(), "Due date must be in the future").optional(),
  priority: z.number().int().min(0).max(3).default(0),
});
export type CreateTaskInput = z.infer<typeof CreateTaskSchema>;   // ← type derived from the schema

function TaskForm({ listId, onDone }: { listId: string; onDone: () => void }) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<CreateTaskInput>({ resolver: zodResolver(CreateTaskSchema) });
  const createTask = useCreateTask();

  return (
    <form onSubmit={handleSubmit(async (values) => {
      await createTask.mutateAsync({ ...values, listId });
      onDone();
    })}>
      <label htmlFor="title" className="block text-sm font-medium">Title</label>
      <input id="title" {...register("title")}
             aria-invalid={!!errors.title}
             className="w-full rounded border px-3 py-2" />
      {errors.title && <p role="alert" className="text-sm text-red-600">{errors.title.message}</p>}

      <button type="submit" disabled={isSubmitting}
              className="rounded bg-blue-600 px-4 py-2 text-white disabled:opacity-50">
        {isSubmitting ? "Saving…" : "Add task"}
      </button>
    </form>
  );
}
```

Three ideas doing the work here:

1. **Zod is a runtime schema that also produces a TypeScript type** (`z.infer`). One declaration, both checks. Same principle as Drizzle's `$inferSelect` in Module 08 — **define once, derive everywhere.**
2. **This is literally the same schema the API validates against.** `zValidator("json", CreateTaskSchema)` in Module 08 and `zodResolver(CreateTaskSchema)` here import the identical object from `packages/shared`. Change `max(200)` to `max(300)` in one place and both ends move together — there is no way for them to drift, because there's only one of them.
   That doesn't make the server check redundant. Client-side validation is for *user experience* — instant feedback, no round trip. Server-side validation is for *correctness and security* — the client can be bypassed with one `curl`. **You need both; sharing the schema just means you only write the rules once.**
3. **React Hook Form keeps inputs uncontrolled**, so typing in one field doesn't re-render the whole form. It also handles touched/dirty state, submission state, and focus management on error.

Accessibility, briefly but non-negotiably: every input has a `<label htmlFor>`, errors use `role="alert"` and `aria-invalid`, and the form must be completable with the keyboard alone. Try it — tab through your form and submit with Enter.

📖 [React Hook Form](https://react-hook-form.com/get-started) · [Zod](https://zod.dev/) · [MDN: Form validation](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Forms/Form_validation)

---

## 10.9 Routing

```bash
npm install react-router
```

You need at minimum: `/login`, `/lists`, `/lists/:listId`. Put filters in the query string so they're shareable.

📖 [React Router](https://reactrouter.com/) · [TanStack Router](https://tanstack.com/router/latest) (fully type-safe routes — worth a look once you're comfortable)

---

## Lab 10 — Build the TaskFlow UI

Build it **one working screen at a time**, running in the browser after each step. Resist scaffolding everything first.

1. Scaffold with Vite. Read the generated files and delete the demo content.
2. Install and configure Tailwind. Get one styled button on screen.
3. `apiClient` with `credentials: "include"` and typed error handling. Wire up React Query's provider.
4. **Login page.** RHF + Zod, calling Better Auth's endpoints. Get a real session cookie set from the browser. *(This is where CORS will bite. Read the Network tab, not Stack Overflow.)*
5. **Protected route wrapper** — redirect to `/login` when there's no session. Handle the "still loading the session" state, or you'll flash the login page on every refresh.
6. **Lists page** — fetch and display. Build the loading, error, and empty states first, *before* the success state.
7. **Task list** — with a status filter kept in the URL query string.
8. **Create task** — Radix Dialog + RHF/Zod form + `useMutation` + cache invalidation. Watch the list update.
9. **Toggle complete** — a checkbox with an [optimistic update](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates). Then throttle your network to "Slow 3G" in devtools and watch it feel instant. Then make the API return an error and confirm it rolls back.
10. **Delete** — with a Radix confirmation dialog.
11. **Polish:** keyboard navigation end-to-end, focus states, a loading skeleton, and a sensible empty state with a call to action.

**Then break it deliberately:**
- Stop the API. Does the UI show a useful error, or a blank white page?
- Throttle to Slow 3G. Are the loading states actually visible and sensible?
- Unplug your mouse and complete the entire flow with the keyboard.
- Run [axe DevTools](https://www.deque.com/axe/devtools/) or Chrome's Lighthouse accessibility audit. Fix what it finds.
- Remove the `key` prop from your task list, add reordering, and watch what happens.

```text
Prompt for Claude Code:
Here's my React TaskFlow UI: [paste 2-3 components].

Review it as a senior frontend engineer. Focus on:
1. State that's in the wrong place — server state in useState, UI state that
   belongs in the URL, state that should be derived instead of stored
2. useEffect that shouldn't exist
3. Missing loading / error / empty states
4. Accessibility problems: labels, focus management, keyboard traps, ARIA
5. Re-render problems and stale closures
6. Anywhere I've duplicated a validation rule instead of sharing the schema

Point at specific lines. Don't rewrite the components.
```

---

## Structured practice

- 🎓 [react.dev Learn](https://react.dev/learn) — the official course. Do all of it, including "Escape Hatches."
- 🎓 [Full Stack Open](https://fullstackopen.com/en/) (University of Helsinki, free) — parts 1–2 for React, part 5 for testing. Rigorous and well-regarded.
- 🎓 [The Odin Project](https://www.theodinproject.com/paths/full-stack-javascript) — free, project-heavy, good on the HTML/CSS foundations.
- 📖 [TkDodo's blog](https://tkdodo.eu/blog/) — for React Query and React patterns generally.
- 📖 [Josh Comeau's CSS for JS Developers](https://css-for-js.dev/) (paid) — if CSS keeps feeling like guesswork, this fixes it permanently.

---

## Understanding Gate

1. What does "UI is a function of state" rule out that jQuery allowed?
2. Why does `key` matter, and why is the array index usually wrong?
3. Name three things your component function must never do, and why.
4. Server state vs client state — why does that distinction justify a whole library?
5. When *should* you use `useEffect`? Give one legitimate example.
6. Explain a stale closure and how to fix one.
7. You have client-side Zod validation. Why does the server still need to validate?
8. Why use Radix for a dropdown instead of writing 30 lines yourself?
9. A user reports "the page is blank." What are the first three things you check?

---

**Next:** [Module 11 — Testing](11-testing.md)
