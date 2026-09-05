# admin.anudeep.pro — Authenticated React Admin SPA

**Date:** 2026-09-05
**Repo:** `github.com/AnudeepChPaul/admin.anudeep.pro` (empty at time of writing)
**Depends on:** `iam.anudeep.pro` @ `8e926f9`, `api.anudeep.pro` (to be rewritten in Node.js)
**Companion documents:**
`../iam.anudeep.pro/architecture-and-auth-flow-reference-2026-09-04.md`,
`../api.anudeep.pro/delegate-identity-graphql-and-ingest-2026-09-04.md`

---

## a) What am I trying to do

Build a CDN-hosted, statically served React SPA at `admin.anudeep.pro` that administers the
**Resume** application only — bio, experiences, projects, technologies — through a
draft/validate/publish/rollback lifecycle, with Markdown editing, sanitized preview, immutable
version history, diff, and asset uploads.

Two hard constraints shape every decision below:

1. **The initial shell must be 20–30 KB compressed JavaScript**, and must be interactive before
   any Resume data arrives. That budget is roughly *React + React DOM alone* on a good day
   (~45 KB brotli for React 18 + DOM), so the shell's dependency list is not a preference — it is
   the whole design. This is addressed in §c.a.3.
2. **The token must never touch a query parameter, a log, `localStorage`, or `IndexedDB`**, and
   IAM service credentials must never exist in frontend code. Token validation is server-to-server
   at `api.anudeep.pro`.

### a.1 One correction to the brief, found in the IAM source

The brief states:

> IAM redirects back to admin with `{token, token_type, email, token_issued_at, csrf_token, expires_in}`.

That object is IAM's `TokenResponse` — the **JSON body** of `POST /v1/login` and
`POST /v1/login/totp`. It is *not* what the redirect carries. What IAM actually does today
(`architecture-and-auth-flow-reference-2026-09-04.md`, Flow 4, `_finish`) is:

```
POST /login  ──▶ 303 Location: <safe_next>
                 Set-Cookie: __Host-anudeep_session=anut_…; HttpOnly; Secure; SameSite=Lax; Path=/
                 Set-Cookie: __Host-anudeep_csrf=…;          Secure; SameSite=Lax; Path=/
```

Both cookies are `__Host-`-prefixed, therefore **host-only to `iam.anudeep.pro`**. A page on
`admin.anudeep.pro` cannot read them, and the browser will not attach them to a request to
`api.anudeep.pro`. The 303 to `safe_next` carries **no credential at all**. So there is presently
no mechanism by which a browser at `admin.anudeep.pro` obtains the token from a redirect — and
IAM has no authorization-code endpoint either (route inventory, §c.c.1 of that document: there is
`/v1/login`, `/v1/login/totp`, `/v1/validate`, and nothing code-shaped).

**Every option in §b is therefore also a statement about what has to change in IAM.** This is the
one decision that gates work; the rest of the build does not depend on it, which is why §c.a.6
isolates it behind a single module.

### a.2 The second constraint the brief did not account for: cookie site-ness

The brief's "preferred final browser model" is:

> API exchanges IAM token for a host-only HttpOnly Secure **SameSite=Lax** cookie.

`admin.anudeep.pro` and `api.anudeep.pro` are different hosts. A cookie set by `api.anudeep.pro`
is **third-party** relative to a document at `admin.anudeep.pro`, and a `SameSite=Lax` cookie is
not attached to cross-site `fetch()`. The combination as written cannot work. There are exactly
three ways out, and only one of them keeps `Lax`:

| Way out | Cookie | Cost |
|---|---|---|
| `SameSite=None; Secure` on the API cookie | works cross-site | ambient authority cross-site; CSRF now rests entirely on the double-submit token + Origin check; blocked outright by third-party-cookie phase-out in Safari/Brave today, Chrome eventually |
| **Same-site the API**: serve it at `admin.anudeep.pro/api/*` via a Cloudflare route/Worker in front of `api.anudeep.pro` | `__Host-` + `Lax` works exactly as the brief wants | one Cloudflare route; the API must accept the proxied `Host`/forwarded headers |
| No cookie at all — bearer token held in JS memory | n/a | token is XSS-reachable for the tab's lifetime; no CSRF surface at all (no ambient authority) |

This mirrors IAM's own reasoning, quoted in its reference document: *"CSRF applies to the cookie
path only. A bearer token carries no ambient authority."*

---

## b) What are my options

The options concern **authentication only**. Frontend stack, module layout, GraphQL/REST split,
draft-revision rules, and the publish state machine are settled by the brief and are not
re-litigated here; they appear in §c as design, not as choices.

### b.0 System context, common to all options

```
                 ┌──────────────────────────── browser ────────────────────────────┐
                 │  admin.anudeep.pro   (static, Cloudflare Pages, brotli, immutable)│
                 │    shell.js 20–30 KB ──lazy──▶ resume · editor · preview · diff  │
                 └───────┬──────────────────────────────────┬──────────────────────┘
                         │ GraphQL /graphql                 │ REST /v1/assets/*
                         ▼                                  ▼
                 ┌───────────────────── api.anudeep.pro (Node) ─────────────────────┐
                 │  resolvers · field authz · persisted ops · upload intents        │
                 └───────┬─────────────────────────────────────────┬───────────────┘
                         │ POST /v1/validate                       │ presign / verify
                         │ Bearer <user token> + service credential│
                         │ X-Pro-Client-Ip / -User-Agent           ▼
                         ▼                                  object storage
                 ┌──────────────────┐
                 │ iam.anudeep.pro  │  passwords · TOTP · sessions · introspection
                 └──────────────────┘
```

Invariant across all three: **the browser never talks to IAM's `/v1/validate`, and never holds an
IAM service credential.** Only `api.anudeep.pro` does, server-to-server.

---

### Option 1 — Fragment hand-off, bearer token in memory

IAM's `_finish` gains one behaviour: when `next` resolves to an admin origin, append the
`TokenResponse` fields to the redirect as a **URL fragment** rather than relying on its own
host-only cookies.

```
IAM 303 ──▶ https://admin.anudeep.pro/auth/callback#token=anut_…&csrf_token=…
                                                    &token_issued_at=…&expires_in=1800&email=…
CallbackPage:  parse location.hash
            →  tokenStore.set(...)             (module-scoped variable; never persisted)
            →  location.hash = ''  via history.replaceState()
            →  navigate('/app', { replace: true })
Every API call:  Authorization: Bearer anut_…
                 X-Pro-Token-Issued-At: <issued_at>
```

```
browser ──#fragment──▶ [ tokenStore : in-memory ] ──Bearer──▶ api ──validate──▶ IAM
```

**Pros**
- Smallest change to IAM: one function, `_finish`, gains a fragment branch. No new table, no new route.
- The fragment is never sent to any server, never appears in `Referer`, never reaches an access
  log or a CDN log. Cleared from the address bar before the first render commits.
- No ambient authority anywhere ⇒ **no CSRF surface**, and the CSRF token becomes vestigial for
  the admin path (still echoed, since the API may accept cookie-auth from elsewhere).
- CORS stays simple and strict: `Access-Control-Allow-Origin: https://admin.anudeep.pro`, no
  `Allow-Credentials` required for a bearer flow.

**Cons**
- The token lives in JS memory ⇒ any XSS in the admin reads it. Mitigated but not eliminated by
  strict CSP + no third-party scripts + sanitized preview in a sandboxed iframe.
- **Full page reload loses the session** — the fragment is gone by then. Either accept a silent
  re-bounce through IAM (which, holding `__Host-anudeep_session`, returns instantly without a
  password prompt — a redirect flicker, not a login), or fall back to `sessionStorage`, which the
  brief explicitly deprecates because XSS reads it.
- A copy-pasted URL from the address bar *before* `replaceState` runs would leak the token; the
  window is one tick, but non-zero.

---

### Option 2 — One-time authorization code, exchanged for a same-site API session cookie *(recommended)*

IAM gains a genuine authorization-code leg. `api.anudeep.pro` is additionally routed at
`admin.anudeep.pro/api/*` by a Cloudflare route, which makes its cookies **same-site** and lets
`__Host-` + `SameSite=Lax` work as the brief intends.

```
IAM 303 ──▶ https://admin.anudeep.pro/auth/callback#code=anuc_<32B>       (≤60 s, single use)
CallbackPage ─POST /api/v1/session/exchange {code}, credentials:'same-origin'──▶ API
    API ──POST iam /v1/authorize/exchange {code} + service credential──▶ IAM
        ← {token, token_issued_at, email, expires_in}
    API stores the IAM token *server-side* (Redis, keyed by its own session id)
    ← 200 {email, permissions[], csrf_token}
      Set-Cookie: __Host-admin_session=…; HttpOnly; Secure; SameSite=Lax; Path=/
      Set-Cookie: __Host-admin_csrf=…;              Secure; SameSite=Lax; Path=/
Every mutation:  X-CSRF-Token: <read from the readable csrf cookie>   + Origin check server-side
```

```
browser ──#code──▶ API ──exchange (service cred)──▶ IAM
   ▲                │
   └── HttpOnly ────┘   IAM token never enters the browser at all
```

**Pros**
- **The IAM token never reaches the browser.** XSS can issue requests as the user while the tab is
  open, but cannot exfiltrate a bearer token for later or lateral use against other estate services.
- Survives reload, and survives a new tab, with no re-bounce and nothing in web storage.
- `SameSite=Lax` + `__Host-` + double-submit CSRF + `Origin` check is exactly the model IAM already
  implements and documents — one security model across the estate rather than two.
- A code is single-use and ≤60 s; a leaked address bar is worthless within a minute, and a replay
  is detectable and auditable.
- Revocation becomes real: the API holds the session and can drop it without waiting for IAM TTL.

**Cons**
- Most work, and it is work in **three** repos: IAM (code table + issue + exchange route), API
  (session store, exchange endpoint, CSRF middleware), admin (trivial — one POST).
- Adds a Cloudflare route in front of the API and a same-origin path prefix; the Node API must
  trust and honour forwarded headers, which is its own small hardening exercise.
- The API becomes stateful for sessions (Redis). It arguably already is, or will be.

---

### Option 3 — Admin renders the login form, calls IAM by XHR

No redirect at all. Admin ships an email/password/TOTP form, `POST`s to IAM with
`credentials:'include'`, reads `TokenResponse` from the JSON body.

**Pros**
- Zero IAM changes — `/v1/login` and `/v1/login/totp` already return exactly the object in the brief.
- No redirect flicker; the whole flow is one SPA.

**Cons**
- **Puts password and TOTP entry inside the admin bundle.** That is credential surface in the
  application least able to justify it, and it duplicates IAM's login pages, its rate limiting, its
  enrolment flow, its recovery-code path, and its CSP-nonced form-token defence.
- Blows the shell budget: a form, validation, TOTP step, error states, and enrolment redirect are
  not free, and they are needed on first paint.
- Token in memory, same as Option 1, with none of Option 1's savings.
- Directly contradicts "IAM owns login, password, TOTP, and session creation."

Listed for completeness. **Do not choose this.**

---

### b.1 Decision taken — 2026-09-05

**Option 2, built directly.** No Option 1 interim cut. The IAM token never enters the browser.

Consequences accepted:

- Admin cannot authenticate until **IAM gains an authorization-code leg** and **the API gains a
  session store + exchange endpoint**. Those become prerequisites of step 3, not parallel work.
  Steps 1, 2, 5 and 15 proceed now; steps 6–14 proceed against the API schema and a stubbed
  session (`VITE_AUTH_STUB=1`, dev-only, refused at build time when `MODE === 'production'`).
- **`api.anudeep.pro` is routed at `admin.anudeep.pro/api/*`** by a Cloudflare route, making the
  API same-site. `__Host-admin_session` (HttpOnly) + `__Host-admin_csrf` (readable) + `SameSite=Lax`
  + `Origin` check + double-submit header — identical to the model IAM already runs.
- `auth/tokenStore.ts` holds **no token**. It holds `{email, permissions[], csrfToken, expiresAt}`
  only; the credential is the cookie, which JS cannot read. The seam in §c.a.6 survives, narrowed.

**Shell budget: Preact + `preact/compat`,** aliased in `vite.config.ts` and mirrored in
`vitest.config.ts`. Target shell ≈ 21 KB brotli, budget held at 30 KB in `.size-limit.json`.

---

## c) Selected option — Option 2 target, Option 1 first cut

### c.a) System design

#### c.a.1 Module and bundle map

```
                          ┌──────────────────── shell.js  (20–30 KB br) ───────────────────┐
                          │  react · react-dom · react-router  · AppShell · Navigation     │
   entry index.html ─────▶│  CallbackPage · tokenStore · apiAuth · restClient(fetch)       │
   <link modulepreload>   │  ErrorBoundary · LoadingShell · CommandPalette(minimal)        │
                          └───┬───────────────────────────────────────────────────────────┘
                              │ React.lazy(() => import(...))   — route level only
        ┌─────────────────────┼─────────────────────┬──────────────────┬─────────────────┐
        ▼                     ▼                     ▼                  ▼                 ▼
  resume-overview.js    editor.js            preview.js          diff.js          uploads.js
   10–20 KB              50–100 KB            20–40 KB            20–50 KB         10–30 KB
   ResumeModule          CodeMirror 6         sandboxed iframe    @codemirror/     presign +
   list · summary        RHF + Zod            + server HTML       merge            multipart
   TanStack Query ──────▶ shared chunk (query.js) pulled in by the first feature that needs it
   urql + generated ────▶ shared chunk (gql.js)   likewise
```

The shell contains **no** TanStack Query, **no** urql, **no** Zod, **no** CodeMirror. This is the
only way the 30 KB ceiling holds; see §c.a.3.

#### c.a.2 Runtime sequence — cold load of `/app/resume`

```
t0  HTML (≈1 KB) + <link rel=modulepreload href=shell.js>
t1  shell.js parses → ReactDOM.createRoot → router
t2  ── SHELL PAINTS ── nav, skeleton, error boundary.  INTERACTIVE.  No API call yet.
t3  SessionBootstrap fires (idle callback), and in parallel:
        import('./features/resume')            ← network for resume-overview.js + query.js + gql.js
t4  SessionMe resolves → nav shows email, permissions gate menu items
t5  ResumeModule mounts → useQuery(ResumeSummary) → skeleton → content
    ── editor.js is NOT fetched.  Reloading /app never fetches CodeMirror. ──
t6  user clicks a document → import('./editor') → editor.js → ContentItem(type,slug)
```

`t2` before `t3` is an acceptance criterion, not an aspiration: `SessionBootstrap` is invoked from
a `useEffect` on the mounted shell, never from a module-scope await, and the router renders its
skeleton without waiting on it.

#### c.a.3 How the 30 KB budget is actually met

React 18 + React DOM is ~45 KB brotli. React Router 6 (`react-router-dom`) adds ~8 KB. That is
already 53 KB — the budget is **already blown before a single line of application code**. Three
ways to close it, in order of preference:

1. **Preact + `preact/compat` aliased at build time.** ~5 KB brotli replacing 45 KB. Every library
   in the stack (urql, TanStack Query, RHF, CodeMirror bindings) works against `preact/compat`.
   Cost: `useSyncExternalStore` edge cases, and Vitest must alias identically.
   Net shell: ~5 (preact) + 6 (`react-router` core only, no `dom` extras) + ~10 (app) ≈ **21 KB**.
2. **React 19 + a hand-rolled 1 KB router** (`history` + `useSyncExternalStore` + a path matcher).
   Seven routes with two dynamic segments do not need React Router. Net ≈ **48 KB** — still over.
3. **Accept 45–55 KB and renegotiate the budget.** Honest, and defensible: the budget's *purpose*
   is "no feature dependency in the shell," and that is satisfiable at 50 KB.

**Recommendation: (1), and state it explicitly in the CI budget file.** If Preact is unacceptable
for reasons outside this document, then (3) — because (2) trades a well-tested router for a bespoke
one and still misses. This is the second decision that needs your call, and unlike the auth
decision it is cheap to reverse later (a Vite alias and a test-config alias).

The CI gate is `size-limit`, run on every PR, failing the build:

```
shell.js ≤ 30 KB br | resume-overview ≤ 20 | editor ≤ 100 | preview ≤ 40 | diff ≤ 50 | uploads ≤ 30
```
plus an **import-graph assertion** — a Vitest test that walks the shell chunk's module list and
fails if it contains `codemirror`, `urql`, `@tanstack`, `zod`, or `marked`. Byte budgets drift;
this one catches the actual regression the acceptance criteria care about.

#### c.a.4 Data flow — server state

```
component ──useQuery(key)──▶ TanStack Query cache ──miss──▶ urql exchange ──▶ POST /graphql
                                   │                                            (persisted op id)
                                   └── mutation ──▶ saveDraft(expectedRevision)
                                                       │ ok    → queryClient.setQueryData(item, result)
                                                       │          ← surgical; NEVER invalidate the list
                                                       └ CONFLICT → conflict dialog (§c.b.2)
```

TanStack Query owns cache/staleness/retry; urql is a thin typed transport (`fetchExchange` only —
**no `cacheExchange`**, which would be a second, disagreeing cache). This is why urql over Apollo:
Apollo's normalized cache is dead weight when TanStack Query is already the cache.

#### c.a.5 Preview isolation

```
editor ──markdown──▶ POST /v1/preview (debounced 400 ms, AbortController)
                     API renders with the SAME renderer+sanitizer as production, returns HTML + ETag
                  ──▶ <iframe sandbox="allow-same-origin" srcdoc=…
                         CSP: default-src 'none'; img-src https: data:; style-src 'unsafe-inline'>
```

Never `dangerouslySetInnerHTML`. `preview.js` therefore contains no Markdown parser at all — which
is how it stays inside 20–40 KB and how "preview and published output use the same renderer" is
guaranteed structurally rather than by discipline.

#### c.a.6 The auth seam

```
auth/tokenStore.ts     ── the ONLY module that knows where a credential lives
auth/apiAuth.ts        ── the ONLY module that turns that into request headers/credentials mode
api/graphqlClient.ts   ── calls apiAuth.decorate(init); knows nothing else
api/restClient.ts      ── ditto
```
Option 1 → Option 2 migration = rewrite these two files. Nothing else imports a token.

---

### c.b) Flow diagrams

#### c.b.1 Callback and session bootstrap

```
 /auth/callback
      │
      ├─ read location.hash ─── empty? ──▶ /auth/failed  ("start again from IAM")
      │
      ├─ Option 1: parse token, token_issued_at, csrf_token, expires_in, email
      │            tokenStore.set(...)                       (in-memory only)
      │  Option 2: parse code ──▶ POST /api/v1/session/exchange {code}
      │                            └─ 4xx ──▶ /auth/failed
      │
      ├─ history.replaceState(null, '', '/auth/callback')     ← fragment gone
      ├─ navigate('/app', { replace: true })
      │
      └─ never console.log the URL; error paths log a code, never the location

 /app  ── render shell immediately ──┐
                                     ├─ effect: SessionMe → { email, permissions[] }
                                     │     401/inactive ──▶ tokenStore.clear() ──▶ IAM /login?next=…
                                     └─ idle: prefetch import('./features/resume')
```

#### c.b.2 Draft save and the revision conflict

```
Cmd/Ctrl+S (or 2 s debounce)
   │
   ├─ IndexedDB: put {type, slug, markdown, ts}     ← local recovery, before the network
   ├─ saveDraft(id, markdown, expectedRevision = cached.revision)
   │      │
   │      ├─ 200 → setQueryData(item, {revision: r+1, errors})  → "Saved" · clear dirty flag
   │      │        → IndexedDB delete
   │      │
   │      └─ STALE_REVISION → conflict dialog, four buttons:
   │             keep local          → refetch item, retry with the server's revision
   │             load server         → discard local, hydrate editor, clear IndexedDB
   │             show diff           → import('./diff') → local vs server
   │             manual merge        → @codemirror/merge, three-way, then save as above
   │
   └─ AbortController cancels the in-flight save if another Cmd+S arrives
```

Validation (`validateDraft`) is debounced separately at 800 ms and its result renders beside the
field, keyed by the server's field path. It is **re-run server-side before publish regardless** —
client validation is convenience only.

#### c.b.3 Publish

```
Cmd/Ctrl+Enter ──▶ confirm dialog ──▶ publishResume(idempotencyKey = uuid stored in component state)
                                          │  the SAME key on a double-click ⇒ same jobId, one deploy
                                          ▼
                              { jobId, state: PENDING }
                                          │
                     poll PublishStatus(jobId): 1s → 2s → 4s → 8s → 15s (cap), jitter ±20 %
                                          │
   DRAFT → VALIDATING ─fail─▶ VALIDATION_FAILED (per-item errors, deep-linked to the editor)
                        │
                        └ok─▶ BUNDLED ─▶ QUEUED ─▶ TRIGGERED ─▶ SUCCEEDED
                                                        └─────▶ FAILED ─▶ rollbackPublish(toBundleId)
                        (UNKNOWN on 3 consecutive poll errors: stop, show "status unavailable",
                         offer manual refresh — never silently retry forever)
```

Publishing is all-or-nothing over a bundle; there is no per-record publish in the UI, by design.

#### c.b.4 Upload

```
choose file ──▶ client sniff (magic bytes, size)          ← convenience only
            ──▶ POST /v1/assets/intent {filename, size, contentType, sha256}
                     ← {uploadUrl, assetId, headers}       (presigned; API is authoritative)
            ──▶ PUT uploadUrl (progress via XHR)           browser → object storage, direct
            ──▶ POST /v1/assets/{assetId}/confirm
                     API re-verifies: content-sniffed MIME, size, ZIP entry count/ratio/paths,
                     rejects traversal + symlinks, generates the server-side filename, attaches
            ← {url, width, height}  ──▶ insert `![](url)` at the editor cursor
```

---

### c.c) Subs and classes

#### c.c.1 Routes

| Path | Chunk | Loads |
|---|---|---|
| `/auth/callback` | shell | fragment parse, exchange, replaceState, redirect |
| `/app` | shell | `AppShell` + `SessionMe`; idle-prefetches resume |
| `/app/resume` | `resume-overview` | `ResumeSummary`, `ContentList(type, status, cursor)` |
| `/app/resume/content/:type/:slug` | `editor` (+`preview` on toggle) | `ContentItem`, `ContentValidation` |
| `/app/resume/history/:type/:slug` | `diff` | `ContentHistory` — on open only |
| `/app/resume/publish` | `resume-overview` | `PublishStatus` — after publish only |
| `/app/settings` | shell | session, permissions, sign-out |

#### c.c.2 Signatures

```ts
// auth/tokenStore.ts  — Option 1. No export ever returns the raw token to a component.
type Credential = { token: string; issuedAt: string; csrfToken: string;
                    email: string; expiresAt: number };
let credential: Credential | null = null;                 // module scope; never persisted
export function setCredential(c: Credential): void;
export function clearCredential(reason: 'logout'|'expiry'|'iam-rejected'): void;
export function isAuthenticated(): boolean;               // expiresAt > now
export function subscribe(fn: () => void): () => void;    // for useSyncExternalStore
/** @internal — apiAuth only */ export function readForTransport(): Credential | null;

// auth/apiAuth.ts  — the one seam between auth model and transport
export function decorate(init: RequestInit): RequestInit;
//   Option 1 → Authorization: Bearer, X-Pro-Token-Issued-At
//   Option 2 → credentials: 'same-origin', X-CSRF-Token on unsafe methods
export function onUnauthorized(): void;                   // clear + bounce to IAM /login?next=
export function loginUrl(next: string): string;

// auth/CallbackPage.tsx
export function CallbackPage(): JSX.Element;              // parse → store/exchange → replaceState → navigate

// shell/SessionBootstrap.ts
export function useSessionBootstrap(): { status: 'idle'|'loading'|'ready'|'anonymous';
                                         session: SessionMe | null };
export function can(p: Permission): boolean;              // content:read|write|publish,
                                                          // asset:write, history:read, rollback:write

// api/graphqlClient.ts
export const client: Client;                              // urql: fetchExchange only, no cacheExchange
export function gqlFetch<T, V>(op: TypedDocumentNode<T,V>, vars: V,
                               signal?: AbortSignal): Promise<T>;

// api/errors.ts
export type ApiError =
  | { kind: 'unauthorized' }
  | { kind: 'forbidden'; permission: Permission }
  | { kind: 'stale-revision'; serverRevision: number }
  | { kind: 'validation'; issues: { path: string; message: string }[] }
  | { kind: 'rate-limited'; retryAfter: number }
  | { kind: 'network' | 'server' | 'aborted'; message: string };
export function toApiError(e: unknown): ApiError;

// features/resume/hooks
export function useContentItem(type: ContentType, slug: string);
export function useSaveDraft(type: ContentType, slug: string);   // expectedRevision + conflict
export function useValidateDraft(type: ContentType, slug: string); // debounced 800 ms
export function usePublish();                                     // idempotency key + backoff poll
export function useLocalRecovery(type: ContentType, slug: string); // IndexedDB, one store, keyPath type+slug

// editor/shortcuts.ts
export function useEditorShortcuts(h: {
  onSave(): void; onPublish(): void; onTogglePreview(): void;
  onPalette(): void; onEscape(): void;
}): void;   // Cmd/Ctrl+S · Cmd/Ctrl+Enter · Cmd/Ctrl+P · Cmd/Ctrl+K · Esc
            // preventDefault on all; ignores repeat; skips when a dialog owns focus
```

#### c.c.3 GraphQL discipline

- Codegen `client-preset`, `TypedDocumentNode`, fragments colocated with components.
- **Persisted operations in production**: codegen emits a hash manifest at build; the client sends
  `{id, variables}` only. Unknown id ⇒ 400. This also removes query text from the bundle.
- Server-side (API repo, listed here so it is not forgotten): depth ≤ 10, complexity ≤ 1000,
  `first` ≤ 50, variables ≤ 64 KB, 10 s resolver timeout, mutation rate limit, field-level authz in
  resolvers (never in the gateway), introspection off in production.
- `ContentHistory` and `PublishedBundle` are never in the initial load path.

#### c.c.4 Repository layout

Exactly the structure given in the brief, plus:
`src/features/resume/hooks/`, `src/api/persisted.json` (generated),
`.size-limit.json`, `tests/shell-imports.test.ts` (the import-graph assertion).

---

## Measured, 2026-09-05 (steps 1–2, 5–7, 9–11 built)

Budgets are now enforced **per route, not per file** — see the correction below. Figures are
the transitive cost of opening a route with only the shell already loaded.

| route | budget | measured (brotli) |
|---|---|---|
| shell | 30 kB | **22.1 kB** |
| resume-overview | 20 kB | 13.1 kB |
| editor | 110 kB (was 100) | **107.1 kB** |
| preview | 40 kB | 12.3 kB |
| diff (cold) | 95 kB (was 50) | **92.2 kB** |
| diff (warm, after the editor) | 15 kB | 8.8 kB |
| uploads | 30 kB | 13.8 kB |

### Uploads: two things the design above did not say

- **The presigned `PUT` must set `withCredentials = false`.** A presigned URL carries its own
  authorisation; sending ours would hand the admin session to a third-party storage origin on
  every upload. Tested.
- **The CSP had to widen for object storage.** `connect-src` and `img-src` now name the
  storage host explicitly (`assets.anudeep.pro` as a placeholder — narrow it to whatever
  storage actually serves). A wildcard there would undo the point of the policy.

SVG is rejected outright rather than sanitised, and the client checks sniff magic bytes
rather than trusting extensions — but all of it is convenience. The API re-derives every
check and is the sole authority.

### The per-file budget was measuring the wrong thing

`size-limit` matched built chunks by filename glob. That works only while each route's code
lives in one file — and it stopped being true the moment the diff viewer arrived. Rollup
hoisted CodeMirror into a chunk shared by the editor and the diff, and the "editor chunk"
fell from 90.7 kB to 23.3 kB **with no change whatsoever to the cost of opening the editor**.
The 70.7 kB shared chunk was in no budget at all.

`scripts/route-budget.mjs` replaces it, and `size-limit` has been removed so there is one
source of truth. From Vite's build manifest it walks each route's transitive static imports,
subtracts everything the shell already loaded, and sums the brotli sizes. It also lists any
chunk no budgeted route reaches — a chunk nobody is measuring.

### Two budgets had to move, with reasons

- **editor 100 → 110 kB.** `@codemirror/view` is ~70 kB of the total and is irreducible for
  a real editor; the remaining pieces (state, commands, language, `@lezer/markdown`) are
  already the minimum set. The original 100 kB was an estimate made before measurement, and
  it was optimistic by 7 %.
- **diff 50 → 95 kB cold, plus a 15 kB warm budget.** 50 kB was right for the path the brief
  had in mind — a diff opened from an editor conflict, where CodeMirror is already loaded;
  that path measures **8.8 kB**. But `/app/resume/history/:type/:slug` is a real URL that can
  be opened cold, and then it pays for CodeMirror itself. Both are now enforced: the cold
  figure is honest about the worst case, and the warm figure is what actually catches a
  regression in the diff's own code, which the cold figure would hide behind CodeMirror.

**`@codemirror/lang-markdown` must not be used.** It depends on `@codemirror/lang-html`
for embedded HTML blocks, which drags in the full HTML, CSS and JavaScript Lezer grammars
plus `@codemirror/autocomplete`: the editor chunk measured **152 kB**, 52 kB over budget,
to highlight fenced code this admin does not need highlighted. Wrapping `@lezer/markdown`
directly in a `Language` gives the same Markdown highlighting for **90.7 kB**. For the same
reason the editor is assembled from individual CodeMirror packages rather than `basicSetup`,
which bundles autocomplete, lint, search and fold.

**The GraphQL contract now lives in `schema.graphql` in the admin repo** and should be
treated as the source of truth by the API rewrite. Codegen runs with
`documentMode: 'string'` and `persistedDocuments: replaceDocumentWithHash`, so a generated
document is *nothing but its sha256 hash*: no query text and no graphql-js runtime reaches
any bundle, verified against `dist/`. The API loads
`src/api/generated/persisted-documents.json` as its allowlist — which also delivers the
depth and complexity guarantees for free, since an operation the build did not emit cannot
be executed at all. This applies in development too, so the API must be started with the
same manifest. CI regenerates and fails on a diff.

One environment note, recorded because it cost real time: under Vitest, `react-router` left
externalised resolves `react` through Node to preact's **CJS** build while the renderer uses
the **ESM** build — two hook instances, and every hook call inside a router component throws
`Cannot read properties of undefined (reading '__H')`. Neither `resolve.alias`,
`resolve.dedupe`, nor `server.deps.inline` fixes it. The fix is
`test.deps.optimizer.web.include: ['react-router']`, which pre-bundles it with esbuild so the
aliases apply to its own imports. `vitest.config.ts` also deliberately omits
`@preact/preset-vite`, whose prefresh plugin is unnecessary in tests.

---

## Implementation order

Follows the brief's 1–15. Steps 1, 2, 5, 15 are unblocked now. Step 3 waits on the §b decision;
step 4 follows it. Steps 6–14 depend only on the API's schema, not on the auth model.

## Prerequisites in other repos

These now block `/auth/callback` end-to-end. Both are small, and both belong to their own repo.

**`iam.anudeep.pro`**
- `authorization_codes` table: `code_hash` (sha256), `user_id`, `session_id`, `client_id`,
  `redirect_uri`, `expires_at` (≤60 s), `consumed_at`. Single use, enforced by a conditional
  `UPDATE … WHERE consumed_at IS NULL RETURNING`.
- `_finish` gains a branch: when `next` resolves to an admin origin, issue a code and redirect to
  `<next>#code=anuc_…` instead of relying on its host-only cookies.
- `POST /v1/authorize/exchange` — service credential + code → `TokenResponse`. Same
  service-credential requirement as `/v1/validate`. Audited.
- Note the open finding in `hardening-recommendations-analysis-2026-09-04.md` item 9: there is
  currently **no way to identify a trusted service caller** since `5016e17` removed service tokens.
  `/v1/authorize/exchange` cannot be safely shipped until that is resolved — it is a stronger
  dependency than `/v1/validate`'s, because a code exchange mints authority rather than reading it.

**`api.anudeep.pro`**
- Session store (Redis), keyed by an opaque id, holding the IAM token, `issued_at`, email,
  permissions, idle + absolute expiry.
- `POST /v1/session/exchange` (code → cookies), `POST /v1/session/logout`, `GET /v1/session/me`.
- CSRF middleware: `Origin` ∈ {`https://admin.anudeep.pro`} **and** `X-CSRF-Token` matches the
  cookie, on every unsafe method.
- Honour `X-Forwarded-*` from the Cloudflare route only, and forward the **end user's** IP and UA
  to IAM as `X-Pro-Client-Ip` / `X-Pro-User-Agent` — not the proxy's.

## Open decisions

None blocking. Revisit if IAM's service-credential gap (above) proves slow to close: that would
reopen Option 1 as an interim, which the seam in §c.a.6 still permits at two files' cost.
