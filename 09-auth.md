# Module 09 — Authentication & Authorization

**Time:** ~6 hours
**Prereq:** [Module 06](06-client-server-http.md), [Module 08](08-backend-api.md)

---

## Why this module exists

Auth is the area where beginners cause the most damage, because the code *appears* to work whether or not it's secure. A login form that lets anyone in looks identical to one that doesn't, until it doesn't.

The rule that governs this entire module: **do not build your own authentication.** Use a well-audited library. But you must *understand* what the library is doing — otherwise you'll misconfigure it, and a misconfigured auth library is just as broken as a hand-rolled one.

---

## 9.1 The two words

- **Authentication (authN)** — *who are you?* Proving identity. Login. → HTTP **401**.
- **Authorization (authZ)** — *are you allowed to do this?* Checking permissions. → HTTP **403**.

They're separate systems. You can be perfectly authenticated and still forbidden. Most real-world breaches are **authorization** failures — the login worked fine, but the app forgot to check that task #42 belonged to the person asking. That's [OWASP #1: Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/).

---

## 9.2 Passwords

If you store passwords, here's the non-negotiable minimum:

- **Never store the password.** Store a **hash** — a one-way function. Even you must not be able to recover it.
- **Never use a general-purpose hash** (MD5, SHA-256). Those are designed to be *fast*, which is exactly wrong: a modern GPU computes billions of SHA-256 hashes per second.
- **Use a password hashing function designed to be slow and memory-hard:** [Argon2id](https://en.wikipedia.org/wiki/Argon2) (current best), [scrypt](https://en.wikipedia.org/wiki/Scrypt), or [bcrypt](https://en.wikipedia.org/wiki/Bcrypt).
- **Salt** — a unique random value per password, stored alongside the hash. It means two users with the same password get different hashes, which defeats precomputed [rainbow tables](https://en.wikipedia.org/wiki/Rainbow_table). Modern libraries do this automatically; you should know it's happening.
- **Compare in constant time.** A naive string comparison returns faster on an early mismatch, which leaks information about the correct value. Libraries handle this.
- **Rate limit login attempts.** Otherwise the strongest hash in the world just slows down an attacker who has all night.
- Check candidate passwords against [Have I Been Pwned's password API](https://haveibeenpwned.com/API/v3#PwnedPasswords) rather than imposing arcane composition rules. "Must contain a special character" is [not what modern guidance recommends](https://pages.nist.gov/800-63-4/sp800-63b.html).

📖 [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — read this one properly.

---

## 9.3 Sessions vs tokens

Once someone proves who they are, how does the *next* request know? HTTP is stateless (Module 06), so each request must carry proof.

### Server-side sessions (the traditional approach)

1. User logs in successfully.
2. Server generates a long random session ID, stores `session_id → user_id, expiry` in the database.
3. Server sends `Set-Cookie: session=<random-id>; HttpOnly; Secure; SameSite=Lax`.
4. Browser sends that cookie automatically on every subsequent request.
5. Server looks up the session, finds the user.

✅ **Revocable instantly** — delete the row and the session is dead.
✅ The cookie is an opaque random string; it carries no data to leak.
❌ Requires a lookup per request (cheap: it's an indexed primary-key hit, and you were querying the DB anyway).

### JWT (JSON Web Tokens)

A JWT is a **self-contained, signed** token. Three base64url-encoded parts separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9 . eyJzdWIiOiJ1c2VyXzEyMyIsImV4cCI6MTc2N30 . dBjftJeZ4CV...
└─────── header ────────┘   └─────────── payload (claims) ──────────┘   └── signature ──┘
```

- **Header** — the signing algorithm.
- **Payload** — claims: `sub` (subject/user id), `exp` (expiry), `iat` (issued at), plus whatever you add.
- **Signature** — HMAC or a public-key signature over header+payload, using a secret only the server knows.

The server can verify a JWT **without any database lookup**: recompute the signature, and if it matches, the payload wasn't tampered with.

> ### 🚨 The two things everyone gets wrong about JWTs
>
> **1. A JWT is signed, not encrypted.** The payload is base64, not ciphertext. Anyone holding the token can read every claim in it. Paste one into [jwt.io](https://jwt.io/) and see. **Never put anything sensitive in a JWT payload.**
>
> **2. A JWT cannot be revoked.** That's the whole point of statelessness — the server doesn't track it. If a token is stolen, it stays valid until it expires. There is no "log out everywhere" without adding a server-side denylist… at which point you're doing a database lookup per request and you've rebuilt sessions, with extra steps and worse properties.

**When JWTs genuinely fit:** short-lived access tokens (5–15 minutes) paired with a long-lived, revocable **refresh token**; service-to-service auth; distributed systems where the verifying service can't reach the auth database.

**When sessions are better:** a normal web app with a browser and one backend. Which is you.

📖 [Stop using JWT for sessions](https://web.archive.org/web/20250113005244/http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/) — a decade old, still the clearest statement of the argument. *(Archived copy: the original is served over plain HTTP only, which — given §6.2 — would be a poor link to hand you.)*
📖 [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519) · [jwt.io](https://jwt.io/) (decode tokens, safely — it's client-side)

**Where to store a token in a browser:** in an `HttpOnly` cookie (Module 06). Not `localStorage`, which any XSS or compromised dependency can read. This is the consensus position and it's why sessions-in-cookies remains the default for web apps.

---

## 9.4 OAuth 2.0 and OpenID Connect

**OAuth 2.0 is a *delegated authorization* protocol.** It answers: *how can this app get permission to act on my behalf at another service, without me giving it my password?*

That last clause is the entire point. Before OAuth, apps asked for your Gmail password.

The four roles:
- **Resource owner** — you
- **Client** — TaskFlow, the app wanting access
- **Authorization server** — GitHub's login system
- **Resource server** — GitHub's API

The **Authorization Code flow with PKCE** (the only one you should use for new work):

```
1. TaskFlow redirects your browser to GitHub:
   /login/oauth/authorize?client_id=…&redirect_uri=…&scope=read:user
   &state=<random>&code_challenge=<hash>&code_challenge_method=S256

2. You log in TO GITHUB (TaskFlow never sees your password) and approve the scopes.

3. GitHub redirects back:  https://taskflow.app/callback?code=<auth-code>&state=<random>

4. TaskFlow's SERVER exchanges that code, plus its client_secret and the
   original code_verifier, for an access token. This happens server-to-server —
   the token never touches the browser.

5. TaskFlow calls GitHub's API with the access token.
```

Why the extra steps:
- **`state`** is a random value you generate and check on the way back. It proves the callback belongs to a flow *you* started — that's CSRF protection.
- **PKCE** (`code_challenge`/`code_verifier`) proves that whoever redeems the code is the same party that started the flow, so an intercepted code is useless. Originally for mobile apps; now recommended universally.
- **`scope`** is least privilege: ask for `read:user`, not everything.

**OpenID Connect (OIDC)** is a thin identity layer *on top of* OAuth 2.0. OAuth alone gives you an access token for an API; it deliberately says nothing about *who the user is*. OIDC adds:
- an **ID token** — a JWT with standard claims about the user (`sub`, `email`, `name`, `iss`, `aud`, `exp`)
- a **`/userinfo`** endpoint
- **discovery** at `/.well-known/openid-configuration`

> **The one-line summary to remember:** OAuth 2.0 is for *authorization* (can this app access that API?). OIDC is for *authentication* (who is this person?). "Sign in with Google" is OIDC; "let this app read your Google Calendar" is OAuth.

Go look at a real one right now:
```bash
curl -s https://accounts.google.com/.well-known/openid-configuration | head -40
```

📖 [OAuth 2.0 Simplified](https://www.oauth.com/) — free book by Aaron Parecki, the best resource on this
📖 [OIDC: How it works](https://openid.net/developers/how-connect-works/) · [oauth.net/2](https://oauth.net/2/)
📺 [OAuth 2.0 and OpenID Connect in plain English](https://www.youtube.com/watch?v=996OiexHze0) — Nate Barbettini's talk; an hour that will make it click

---

## 9.5 Authorization models

Once you know who someone is, decide what they can do.

| Model | How it works | Fits |
|---|---|---|
| **Ownership** | "You may edit rows where `user_id` = you" | TaskFlow. Most small apps. |
| **RBAC** — role-based | Users get roles (`admin`, `member`); roles get permissions | Most business apps |
| **ABAC** — attribute-based | Rules over attributes: "editors may edit drafts in their own department during business hours" | Complex policy |
| **ReBAC** — relationship-based | Permissions derive from a graph of relationships (Google Docs sharing) | Collaboration products |

Whatever the model, one rule dominates:

> **Check authorization on the server, on every request, as close to the data as possible.**
>
> Hiding a Delete button in the UI is not authorization — it's decoration. The endpoint is still one `curl` away. Every single endpoint must independently verify permission, because every single endpoint is directly reachable.

📖 [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

---

## 9.6 Better Auth

[Better Auth](https://www.better-auth.com/) is a framework-agnostic TypeScript auth library: email/password, OAuth providers, sessions, and the database schema, with sensible secure defaults.

```bash
cd apps/api
npm install better-auth
```

```ts
// apps/api/src/lib/auth.ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { db } from "../db/client";
import { env } from "../config";

export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: "pg" }),
  secret: env.BETTER_AUTH_SECRET,
  baseURL: env.API_ORIGIN,
  emailAndPassword: { enabled: true },
  socialProviders: {
    github: { clientId: env.GITHUB_CLIENT_ID, clientSecret: env.GITHUB_CLIENT_SECRET },
  },
  session: { expiresIn: 60 * 60 * 24 * 7 },   // 7 days
  trustedOrigins: [env.WEB_ORIGIN],
});
```

Mount its handler, then add middleware that resolves the session for your own routes:

```ts
// apps/api/src/index.ts
import { Hono } from "hono";
import { auth } from "./lib/auth";
import { UnauthorizedError } from "./lib/errors";

type User = typeof auth.$Infer.Session.user;

// Declaring Variables makes c.get("user") typed everywhere downstream
const app = new Hono<{ Variables: { user: User | null } }>();

// Better Auth serves the whole /api/auth/* surface: signin, signout, OAuth callbacks…
// c.req.raw is the underlying web-standard Request — the object from Module 08.1.
app.on(["GET", "POST"], "/api/auth/*", (c) => auth.handler(c.req.raw));

// Runs on every /api request: read the session cookie, resolve the user (or null)
app.use("/api/*", async (c, next) => {
  const session = await auth.api.getSession({ headers: c.req.raw.headers });
  c.set("user", session?.user ?? null);
  await next();
});

// A guard that short-circuits: if there's no user, downstream never runs.
// Mount it on everything that requires a login — and nothing else.
const requireAuth = async (c, next) => {
  if (!c.get("user")) throw new UnauthorizedError();   // → 401 via onError (Module 08.6)
  await next();
};

app.use("/api/tasks/*", requireAuth);
app.use("/api/lists/*", requireAuth);

app.get("/api/me", requireAuth, (c) => c.json(c.get("user")));
```

Notice the middleware **onion** from Module 08.4 doing real work: the session lookup runs first and attaches the user; `requireAuth` either calls `next()` or throws; your route handler only ever runs for an authenticated request. That's authentication in one place instead of copy-pasted into twelve handlers — and a handler you *forget* to protect is the single most common way this goes wrong.

Now replace every hardcoded `userId` from Module 08 with `c.get("user").id`, and make **every** task/list query filter by it.

**Read Better Auth's generated tables.** Run its migration, then `\d user`, `\d session`, `\d account` in psql. You'll see: hashed passwords, session tokens with expiries, and one `account` row per OAuth provider link. That's sections 9.2–9.4 made concrete — go look at them and connect each column to a concept above.

📖 [Better Auth docs](https://www.better-auth.com/docs) · [Drizzle adapter](https://www.better-auth.com/docs/adapters/drizzle) · [Hono integration](https://www.better-auth.com/docs/integrations/hono) · [Social providers](https://www.better-auth.com/docs/authentication/github)

### Making cookies work across origins in development

Your frontend is `http://localhost:5173`; your API is `http://localhost:3000`. Different ports = different origins (Module 06). For the session cookie to work you need:

- Server: CORS with a **specific** origin (never `*`) and `credentials: true`
- Client: `fetch(url, { credentials: "include" })` on every request
- Cookie: `SameSite=Lax` works for same-site-different-port in most browsers; genuinely cross-*site* needs `SameSite=None; Secure`, which requires HTTPS

**Expect to lose an hour here, and treat it as a lesson rather than an obstacle.** When it fails, open devtools → Network → the request → Cookies tab, and see whether the cookie was sent, and Application → Cookies to see whether it was ever stored. Debug from evidence, not from guessing.

---

## Lab 09 — Ship real auth

1. **Decode a JWT by hand.** Get one from [jwt.io](https://jwt.io/), then in a terminal:
   ```bash
   echo '<payload-segment>' | base64 -d
   ```
   Read the claims. Now change one character in the payload and try to verify it. **Write down in one sentence why you couldn't forge it, and why you could still read it.**

2. **Install and configure Better Auth.** Run its migration; inspect the generated tables in psql.

3. **Email/password signup and login** via `curl`. Save the cookie jar (`-c`/`-b`, Module 06) and use it to hit `/api/me`. Watch the `Set-Cookie` header — check every attribute: is it `HttpOnly`? `Secure`? What's the `SameSite` value?

4. **Protect your routes.** Every `/api/tasks` and `/api/lists` endpoint requires a session; every query filters by the session's user id.

5. **Attack yourself.** This is the most important step in the module.
   - Create two users, A and B. As A, create a task and note its id.
   - As B, try `GET /api/tasks/<A's task id>`. Then `PATCH` it. Then `DELETE` it.
   - Try with no cookie at all. Try with a made-up cookie value. Try with A's cookie after deleting A's session from the database.
   - Try `PATCH /api/tasks/<B's own task>` with a body containing `{"listId": "<one of A's lists>"}` — can B move their task into A's list? **This one catches almost everyone.**

   Every one of these must fail with the right status. Fix any that don't.

6. **Add GitHub OAuth.** Register an OAuth app at [github.com/settings/developers](https://github.com/settings/developers) with callback `http://localhost:3000/api/auth/callback/github`. Then run the flow with the Network tab open and **trace every redirect**: write down each URL, its query parameters, and what each one is for. Find the `state` parameter and confirm it comes back unchanged.

7. **Rate limit the login endpoint.** Then prove it works by hammering it in a loop.

```text
Prompt for Claude Code:
I've implemented auth with Better Auth in my TaskFlow API: [paste auth
config, the session guard, and 2-3 protected routes].

Act as a penetration tester reviewing this. Give me a numbered list of every
way an attacker could access or modify data they shouldn't. Include:
- IDOR / broken object-level authorization on each endpoint
- Anything I can set via a request body that I shouldn't (e.g. moving a
  resource to another user's parent)
- Cookie attributes that are wrong or missing
- Session fixation, session lifetime, logout behaviour
- Information leaked by my error responses or status codes
- Anything missing: rate limiting, CSRF, email verification, password reset

For each, tell me how to TEST it with curl. Don't write fixes yet.
```

---

## Understanding Gate

1. Authentication vs authorization — one sentence each, plus the status code for each failure.
2. Why is a JWT payload readable by anyone holding the token?
3. Why can't you revoke a JWT, and what do people do about it?
4. Why is `HttpOnly` on the session cookie more important than the token format?
5. In OAuth's authorization code flow, what is `state` for and what breaks without it?
6. What does OIDC add to OAuth 2.0, and why was it needed?
7. Why is hiding the Delete button not authorization?
8. Why can't you store passwords with SHA-256 — it's a secure hash, isn't it?
9. A user's laptop is stolen. What do you do, and does your answer depend on sessions vs JWTs?

---

**Next:** [Module 10 — Building the Frontend](10-frontend.md)
