# Migrating iam.anudeep.pro from Python to Node.js

**Date:** 2026-09-05
**Source:** `iam.anudeep.pro_python` (branch `python`, `c815c41`) — FastAPI / SQLAlchemy / Python 3.13
**Target:** `iam.anudeep.pro` — Fastify / Drizzle / TypeScript on Node 24 LTS
**Revised:** 2026-09-05 after the nine-point review — see `migration-review-analysis-2026-09-05.md`
**Status:** Stage 1 complete (scaffold running, 7 tests green). Revised after the nine-point review; stage 1b applies its version decisions.

---

## a) What am I trying to do

Reproduce an identity service in Node.js, feature for feature, without losing the decisions
that make it correct.

**What exists today**

| | |
|---|---|
| Source | ~4,800 lines, 40 files |
| Tests | ~3,000 lines, **160 test cases** |
| Migrations | 4 Alembic revisions |
| HTTP surface | 14 JSON endpoints + 5 HTML screens (GET and POST each) |
| Datastores | Postgres (3 tables, `iam` schema) + Redis (4 key families) |
| Telemetry | 13 counters, 1 histogram, 13 span sites, correlated logs |
| Operational | CLI (2 commands), smoke driver, Grafana stack, Railway deploy |

**Two constraints set by the user**

1. **Side-by-side directories.** The Python moves to `iam.anudeep.pro_python` and keeps the
   `iam.anudeep.pro` name for Node. Both run at once, so parity is *demonstrated* per endpoint
   rather than asserted at the end.
2. **Greenfield data.** No requirement that existing argon2 hashes verify or that live Redis
   sessions resolve. This removes the largest single risk and lets the schema be created clean.

**What a successful migration means.** A client calling `/v1/validate` cannot tell which
implementation answered: same status codes, same JSON keys, same cookie names and flags, same
Problem Details shape, and the same *semantic* telemetry — counter names, attribute keys and
permitted values, hand-placed span names. Not the same literal OTLP output; `service.name` and
the SDK-generated spans differ by design. The port is finished when the black-box smoke driver
passes against the Node server with only its base URL and the renamed paths changed.

**The real risk is not syntax.** It is the handful of places where the Python is subtly correct
and an idiomatic Node translation is subtly wrong: constant-time comparison, the transaction
boundary that closes the registration race, case-insensitive email, and the attribute rule that
keeps credentials out of telemetry. Those are enumerated in (c.d).

---

## b) What are my options

### b.1 — Framework

#### Option 1 — Express

- **Pros:** ubiquitous; every developer knows it; largest middleware ecosystem.
- **Cons:** no schema validation or response serialization; callback-era ergonomics that fit
  poorly with the async service layer here; TypeScript support is bolted on. Every guarantee
  FastAPI gives for free would be hand-rolled.

#### Option 2 — NestJS

- **Pros:** batteries included; DI container; decorators map neatly onto FastAPI's `Depends`.
- **Cons:** imposes an opinionated module/provider architecture on a codebase that is
  deliberately explicit — `deps.py` is 150 readable lines, and replacing it with a DI framework
  adds indirection to the part that is currently easiest to follow.

#### Option 3 — Hono

- **Pros:** modern, small, superb TypeScript, edge-portable.
- **Cons:** edge-first; the Node-native story for `pg`, ioredis and OTel instrumentation is
  thinner. Portability to Workers is not a property this service needs.

#### Option 4 — Fastify  ✅ **selected**

- **Pros:** the closest structural analogue to FastAPI — schema-first with validation *and*
  response serialization, lifecycle hooks ≈ middleware, plugin encapsulation ≈ routers, and
  `app.inject()` gives in-process HTTP, which is exactly what `conftest.py` achieves with an
  in-loop ASGI transport. Mature OTel instrumentation. Fastest of the mainstream.
- **Cons:** smaller ecosystem than Express; hooks are less obvious than middleware at first read.

```
   FastAPI                          Fastify
   ─────────────────────────        ─────────────────────────
   APIRouter(prefix=…)         →    fastify.register(plugin, { prefix })
   Depends(...)                →    decorators + preHandler hooks
   BaseHTTPMiddleware          →    onRequest / onSend hooks
   response_model=X            →    schema.response + Zod type provider
   TestClient / ASGITransport  →    app.inject()
   exception_handler           →    setErrorHandler
```

### b.2 — Validation

| | Zod ✅ | TypeBox | Valibot |
|---|---|---|---|
| JSON Schema native | no (adapter) | **yes** | no |
| Serialization speed | good | **best** | good |
| Custom validators | **`.superRefine()`, `.transform()`** | awkward | good |
| Schema inheritance | **`.extend()`** | `Type.Composite` | `v.intersect` |
| Ecosystem | **largest** | growing | newest |

**Selected: Zod.** The deciding factor is that this port carries three pydantic features that
are not plain shape validation: `_parse_json_list` (accept a JSON array *or* a comma-separated
string), `_coerce_asyncpg` (rewrite a URL scheme), and `_ceiling_outlives_the_window` (a
cross-field check that refuses a bad TTL pair at boot). Zod expresses all three directly with
`.transform()` and `.superRefine()`; TypeBox would fight each one.

*A fourth reason has since evaporated and is recorded rather than quietly dropped.* This
originally also cited schema inheritance — `BrowserStartResponse` extending `StartResponse` so
the HTML and JSON surfaces could not drift. The review moved the UI to HTML-only, so those
subclasses are not ported and inheritance is no longer needed. The three validators are enough
on their own; had inheritance been the *only* argument, TypeBox would now be the better pick.

### b.3 — Database access

#### Option 1 — Prisma

- **Pros:** best-in-class migrations and DX; excellent generated types.
- **Cons:** its own schema DSL is a second language; raw SQL is an escape hatch rather than a
  first-class citizen — and this service needs `pg_advisory_xact_lock`, `hashtext`, `CITEXT`,
  `INET` and `Identity(always)`. Fighting the ORM on all five is the wrong trade.

#### Option 2 — Kysely

- **Pros:** the best types of any query builder; thin; raw SQL is natural.
- **Cons:** no schema-as-code, so the SQLAlchemy models have no counterpart; migrations are
  bring-your-own.

#### Option 3 — Drizzle  ✅ **selected**

- **Pros:** table definitions read like the declarative models they replace; `customType` covers
  CITEXT and INET; `sql\`\`` is first-class, not an escape hatch; `drizzle-kit` gives an Alembic
  analogue; thin layer over `pg`.
- **Cons:** younger; generated migrations are diffs, so they will be hand-written here.

### b.4 — Supabase (raised during review)

**As the database:** viable. It is Postgres; CITEXT and INET are available; and the registration
lock is `pg_advisory_xact_lock` — *transaction*-scoped, so it survives transaction-mode pooling
where a session-scoped lock would silently not.
**Rejected because** the service needs one small database with two extensions, while Supabase
brings PostgREST, GoTrue, Realtime, Storage and a dashboard that would go unused; self-hosting
is ~10 containers, which works against the single all-in-one image.

**As the auth system:** it would *replace* this service, not support it — and it contradicts
what this codebase decided on purpose.

| This service | Supabase Auth (GoTrue) |
|---|---|
| Opaque 256-bit tokens, meaningless without the store | JWTs with readable claims |
| Authoritative `/v1/validate`; revocation instant | Verified locally; valid until `exp` |
| Sliding idle window as a Redis TTL | Fixed expiry + refresh rotation |
| Fingerprint binding (IP `/24` + UA) | none |
| Second factor **mandatory** | MFA enrolment optional |
| `registration_mode=single` gate | custom hooks |

Migration `0002_sessions` moved *away* from exactly the refresh-token model GoTrue uses.
**Rejected.**

### b.5 — The whole stack, compared at a glance

```
   PYTHON                                 NODE
   ────────────────────────────────       ────────────────────────────────
   FastAPI ..........................→    Fastify 5
   Pydantic v2 / pydantic-settings ..→    Zod 4  (+ fastify-type-provider-zod)
   SQLAlchemy 2.0 async .............→    Drizzle ORM
   Alembic ..........................→    Drizzle Kit (hand-written SQL)
   asyncpg ..........................→    pg (node-postgres)
   redis-py asyncio .................→    ioredis
   argon2-cffi ......................→    @node-rs/argon2
   pyotp ............................→    otpauth
   qrcode ...........................→    qrcode
   structlog ........................→    pino (+ pino-pretty)
   contextvars ......................→    AsyncLocalStorage
   opentelemetry-python .............→    @opentelemetry/sdk-node
   pytest + httpx ASGI ..............→    Vitest + app.inject()
   ruff .............................→    Biome
   mypy .............................→    tsc --noEmit
   uv / pip .........................→    pnpm
   uvicorn ..........................→    Fastify's own listener
```

**Unchanged on purpose:** the Postgres schema, the Redis key layout and TTL semantics, cookie
names and flags, and the **semantic** telemetry — counter names, their attribute keys and
permitted values, the hand-placed span names, the histogram and the attribute rule. Not the
literal OTLP output: `service.name` differs deliberately so both services can report to one
collector, and SDK-generated spans differ (`SELECT` from SQLAlchemy versus `pg.query` from
`instrumentation-pg`).

**Changed on purpose, decided during review:** the HTTP surface is unified under `/v1`, and the
browser screens move to `/ui/v1/*`. Each surface has one content type — `/v1` answers JSON,
`/ui/v1` answers HTML — and both call one shared service layer. See the routing table below.

**Versions are pinned exactly**, not with `^`. The lockfile already pins the tree; pinning the
manifest as well removes the case where `pnpm update` silently takes a minor of
`@node-rs/argon2` or a Fastify plugin. For a service that issues credentials, that is worth the
friction.

---

## c) The selected design

### c.a) System architecture

```
                          ┌──────────────────────────────────────┐
   browser / service ─────▶│ Fastify 5                            │
                          │                                      │
                          │  onRequest hooks (outermost first):   │
                          │   1. originLock      (prod only)      │
                          │   2. trustedHost                      │
                          │   3. requestContext  ── ALS + pino     │
                          │   4. cors                             │
                          │   5. securityHeaders (CSP nonce)      │
                          │                                      │
                          │  plugins (≈ routers):                 │
                          │   health   /healthz /readyz           │
                          │   v1       /v1/*        JSON only     │
                          │   validate /v1/validate JSON only     │
                          │   ui       /ui/v1/*     HTML only     │
                          │                                      │
                          │            both call ↓                │
                          │        ┌──────────────────┐          │
                          │        │   AuthService    │          │
                          │        └────────┬─────────┘          │
                          └─────────────────┼────────────────────┘
                                  ┌─────────┴─────────┐
                          ┌───────▼────────┐  ┌───────▼────────┐
                          │ Drizzle + pg   │  │ ioredis        │
                          │ Postgres :5434 │  │ Redis :6381    │
                          │  iam.admin_user│  │ session:<sha>  │
                          │  iam.recovery… │  │ sessions:user:*│
                          │  iam.login_at… │  │ pending:<sha>  │
                          └────────────────┘  └────────────────┘
                                  │                   │
                                  └────────┬──────────┘
                                    OTLP :4318 (shared collector)
                                  service.name=iam.anudeep.pro-node
```

### c.b) Flow diagrams

#### The two-leg sign-in, unchanged in behaviour

```
  POST /v1/login  { email, password }
        │
        ├─ enforceRateLimit          span iam.rate_limit.check
        │    └─ SELECT login_attempt  (failures since last success, per IP and per account)
        ├─ usersByEmail               ← CITEXT: case-insensitive, security-relevant
        │    └─ not found → dummyVerify()  ← equal work, so timing cannot enumerate
        ├─ verifySecret(argon2id)     histogram iam.password.hash.duration
        ├─ not enrolled → status totp_enrolment_required
        └─ enrolled     → status totp_required
                └─ issuePending()     span iam.pending.issue
                     └─ HSET pending:<sha256(handle)>  EX 300
        ▼
  200 { status, email, pending_handle }        counter iam.login.attempts{outcome}

  POST /v1/login/totp  { pending_handle, totp_code }
        │
        ├─ loadPending()  → replayed? wrong purpose? fingerprint mismatch → burn + 401
        ├─ otpauth verify (window: 1)   counter iam.totp.verifications{outcome,stage}
        ├─ pending.spend() → boolean    MULTI[DEL, SETEX]; DEL's reply names the winner
        │     └─ false ⇒ a concurrent caller claimed it first → 401, no session
        └─ issueSession()               span iam.session.issue
             └─ HSET session:<sha256(token)> EX 1800 ; SADD sessions:user:<id>
        ▼
  200 TokenResponse + Set-Cookie ×2             counter iam.login.completed{method}
```

#### Registration, and the transaction that must not be broken

```
  ┌─ db.transaction() ──────────────────────────────────────────┐
  │                                                              │
  │  SELECT pg_advisory_xact_lock(hashtext('iam:registration'))  │
  │        │  serialises the check against the insert; released  │
  │        │  by COMMIT, so no path can leak it back to the pool │
  │        ▼                                                     │
  │  count(admin_user) ── mode=single and count>0 → 403          │
  │        ▼                                                     │
  │  INSERT admin_user                                           │
  │        ▼                                                     │
  │  INSERT login_attempt        ← MUST be inside this tx.       │
  │                                In Python this is the         │
  │                                _record(commit=False) case.   │
  └──────────────────────────────┬───────────────────────────────┘
                             COMMIT (lock released)
                                 ▼
                    issuePending(purpose=enrol, ttl=600)
```

#### The two surfaces, and how a screen is served

```
   ── without JavaScript ────────────────────────────────────────────
   GET  /ui/v1/login              → 200 HTML   (one response, nothing else loads)
   POST /ui/v1/login   form       → AuthService.login()
                                    ├─ ok       303 → /ui/v1/login/authenticator
                                    └─ refused  200 HTML, same screen, message inline

   ── with JavaScript ───────────────────────────────────────────────
   GET  /ui/v1/login              → 200 HTML + ~1.3 KB inline script
   POST /v1/login      json       → 200 StartResponse
                                    script maps status → next screen, location.assign()

   Both paths run the same AuthService method. Only the rendering differs.
```

`/v1` never emits HTML and `/ui/v1` never emits JSON. The screens are rendered server-side from
template literals — no framework, no bundler, no hydration — under
`default-src 'none'` with per-request nonces, so the page loads nothing at all from anywhere.

#### Request id ↔ trace id, via AsyncLocalStorage

```
  onRequest hook
     │  requestId = header X-Request-ID ?? randomUUID()
     ├─ als.run({ requestId, logger }, next)      ← contextvars analogue
     └─ reply.header('X-Request-ID', requestId)

  traced(name)                     pino processor
     │ span.setAttribute(            │ mixin() → { trace_id, span_id }
     │   'app.request_id', …)        ▼
     ▼                          {"event":"pending_issued","request_id":"abc",
   TraceQL:                       "trace_id":"fe27…","span_id":"d60e…"}
   { span.app.request_id = "abc" }
```

### c.c) Module map — subs and classes

Every Python module has one TypeScript counterpart. Paths are `src/` relative.

| Python | TypeScript | Exports |
|---|---|---|
| `settings.py` | `config.ts` | `Settings` (Zod schema + inferred type), `getSettings()` (memoised) |
| `main.py` | `app.ts` / `server.ts` | `buildApp(settings): FastifyInstance`, `start()` |
| `db.py` | `db/client.ts` | `getPool()`, `getDb()`, `withTransaction(fn)` |
| `models/base.py`, `models/user.py` | `db/schema.ts` | `adminUser`, `recoveryCode`, `loginAttempt`, `citext`, `inet` custom types |
| `redis_client.py` | `redis/client.ts` | `getRedis()` |
| `repositories/user_repo.py` | `repositories/*.ts` | `AdminUserRepository`, `RecoveryCodeRepository`, `LoginAttemptRepository` |
| `services/session.py` | `services/session-store.ts` | `SessionStore`, `SessionRecord`, `DEFAULT_*_TTL_SECONDS` |
| `services/pending.py` | `services/pending-store.ts` | `PendingAuthStore`, `PendingRecord`, `PendingReplayedError`, `PURPOSE_*` |
| `services/auth.py` | `services/auth-service.ts` | `AuthService` and its 20 methods; `IssuedSession`, `LoginOutcome`, `IntrospectionResult` |
| `security/tokens.py` | `security/tokens.ts` | `mintSessionToken()`, `mintPendingHandle()`, `hashToken()`, `cookieName()` |
| `security/passwords.py` | `security/passwords.ts` | `hashSecret()`, `verifySecret()`, `dummyVerify()`, `MIN_PASSWORD_LENGTH` |
| `security/fingerprint.py` | `security/fingerprint.ts` | `networkOf()`, `compute()`, `matches()` |
| `security/csrf.py` | `security/csrf.ts` | `newCsrfToken()`, `csrfTokenValid()`, `originAllowed()` |
| `security/cookies.py` | `security/cookies.ts` | `names()`, `setCookie()`, `applySession()`, `clearSession()` |
| `security/headers.py` | `plugins/security-headers.ts` | `securityHeaders`, `originLock`, the three CSP constants |
| `security/service_auth.py` | `security/service-auth.ts` | `credentialValid()`, `requireService`, `enforceServiceRateLimit()` |
| `schemas/v1.py` | `schemas/v1.ts` | Zod: `RegisterRequest`, `LoginRequest`, `LoginTotpRequest`, `StartResponse`, `TokenResponse`, `LoginStatus`. The two `Browser*` subclasses are **not** ported — `/ui/v1` returns HTML |
| `schemas/validate.py` | `schemas/validate.ts` | `ValidateResponse`, `HEADER_*` |
| `schemas/auth.py` | `schemas/auth.ts` | `MeResponse`, `RecoveryLoginRequest`, `PasswordChangeRequest` |
| `routers/v1.py`, `routers/auth.py` | `routes/v1.ts` | one plugin; `auth.py`'s four routes fold in as `/v1/me`, `/v1/logout/all`, `/v1/password/change`, `/v1/login/recovery` |
| `routers/validate.py` | `routes/validate.ts` | `/v1/validate` |
| `routers/login.py` | `routes/ui.ts` | the five screens under `/ui/v1`, HTML only |
| `routers/health.py` | `routes/health.ts` | unversioned probes |
| `templates.py` | `views/templates.ts` | `PAGE`, `CSS`, `JS`, `FORM_*` template literals |
| `qr.py` | `views/qr.ts` | `qrSvg()` |
| `errors.py` | `errors.ts` | `AppError` hierarchy, `problemDetails()`, `registerErrorHandler()` |
| `logging.py` | `logging.ts` | `configureLogging()`, `requestContext` hook, `als` |
| `observability.py` | `observability.ts` | `configureTelemetry()`, `traced()`, `annotate()`, the 13 instruments, `FORBIDDEN_ATTRIBUTE_HINTS` |
| `cli.py` | `cli.ts` | `create-user`, `clear-lockout` |

**`AuthService` — the surface that must be reproduced exactly**

```ts
class AuthService {
  register(...): Promise<[AdminUser, string]>            // @traced iam.register
  login(...): Promise<LoginOutcome>                       // @traced iam.login
  completeTotp(...): Promise<[IssuedSession, AdminUser]>  // @traced iam.login.totp
  loginWithRecoveryCode(...): Promise<IssuedSession>      // @traced iam.login.recovery
  pendingPurpose(handle): Promise<string | null>          // never throws
  ensureEnrolmentSecret(handle): Promise<[...]>           // idempotent
  beginTotpEnrolment(handle): Promise<[...]>              // @traced iam.enrolment.begin
  confirmTotpEnrolment(...): Promise<AdminUser>           // @traced iam.enrolment.confirm
  changePassword(...): Promise<void>                      // @traced iam.password.change
  logout(...) / logoutEverywhere(...)                     // @traced iam.logout.everywhere
  introspect(...): Promise<IntrospectionResult>           // @traced iam.introspect
  issueSession(...) / resolveSession(...) / revokeSession(...)
  private lockRegistration() / requireRegistrationOpen()
  private issuePending() / loadPending() / totpValid()
  private enforceRateLimit() / record(..., commit)
}
```

### c.d) The six places a naive port goes wrong

These are the reason this is a careful migration rather than a transliteration.

**1. Constant-time comparison.** `hmac.compare_digest` accepts unequal lengths;
`crypto.timingSafeEqual` **throws** on them. A direct port either leaks length through an
exception or, if wrapped in a length check that returns early, stops being constant-time.
→ Compare fixed-width digests (both sides are sha256 hex), and where lengths can differ, hash
both operands first and compare the digests.

**2. The registration transaction.** `_record(commit=False)` exists so the audit row shares the
transaction holding the advisory lock. In Drizzle, if the audit insert lands outside
`db.transaction()`, the lock is released early and the single-registration race reopens —
silently, and only under concurrency.

**3. Case-insensitive email.** `CITEXT` is doing security work: `byEmail` must stay
case-insensitive or two accounts can exist for one address. Drizzle needs a `customType`, and
the uniqueness test must assert `Me@X.pro` collides with `me@x.pro`.

**4. The telemetry attribute rule.** `FORBIDDEN_ATTRIBUTE_HINTS` raises on a
credential-shaped attribute name, and `service_id` is attached only *after* the credential
verifies — otherwise a caller mints unbounded time series. Both properties must survive, with
their tests.

**5. Check-then-act on the pending handle.** The Python reads the handle, verifies the code,
then spends it — and discards `DEL`'s reply, so a concurrent loser is never told it lost and
issues a session anyway. **This is a live bug being ported, not a translation risk**: two
callers with one handle and one valid code both get a session. The Node version must treat
`spend()` as a claim that returns a boolean, and refuse when it is false. No Lua — `DEL` inside
`MULTI/EXEC` already names exactly one winner.

**6. Check-then-act on recovery codes.** The same shape in Postgres: `SELECT … WHERE used_at IS
NULL`, verify, `UPDATE`. Two concurrent requests with one code both see it unused. The database
must pick the winner — `UPDATE … WHERE used_at IS NULL … RETURNING`, and no row returned means
someone else spent it.

### c.e) Docker

| File | Target | Contents |
|---|---|---|
| `Dockerfile` | `builder` | pnpm install, `tsc` build |
| | `runtime` | Node + built app, non-root — **what Railway deploys** |
| | `allinone` *(default)* | `runtime` + Postgres + Redis + supervisor + migrate-then-start entrypoint |
| `Dockerfile.dev` | — | `pnpm dev` under `tsx watch`, source bind-mounted |

`docker compose up` brings up **app + Postgres + Redis + collector + Tempo + Prometheus +
Grafana** by default, with healthchecks and `depends_on`.

The all-in-one contains what the app *cannot run without* — the two datastores. The
observability stack stays in compose: it would add over a gigabyte, and telemetry is off unless
`IAM_OTEL_ENABLED` is set.

### c.f) Sequence

Each stage ends with its ported tests passing.

| # | Stage | Gate |
|---|---|---|
| 0 | This document | ✅ done |
| 1 | Move Python; scaffold Node | ✅ done — `pnpm check` green, `/healthz` and `/readyz` verified live |
| 1b | **Apply the review**: Node 24.20.0, exact version pins, `.nvmrc` | `pnpm check` |
| 2 | Config + Drizzle schema + one initial migration | migration applies and reverses |
| 3 | Security primitives | unit tests, incl. the timing-safe port |
| 4 | Session and pending stores | Redis key/TTL tests |
| 5 | `AuthService` | registration race, **pending-handle race**, **recovery-code race**, recovery-before-enrolment |
| 6 | `/v1/*` (all ten, incl. the four renamed), `/v1/validate`, health | contract tests, JSON only |
| 7 | `/ui/v1` screens, GET and POST, HTML only | no-JS **and** scripted paths |
| 8 | Observability | span/counter/PII tests |
| 9 | CLI, scripts, **parity fixtures + differ** | byte-level diff against `:8001` |
| 10 | Docker, Railway, README | all-in-one image actually run |

### c.g) Verification

```sh
pnpm check && pnpm test                        # Biome + tsc + Vitest
docker compose up -d                           # app + stores + observability
docker build -t iam:allinone . && docker run -p 8000:8000 iam:allinone
docker build --target runtime -t iam:prod .

# parity, both servers up
python  :8001    node  :8000
```

A parity script drives an identical sequence against both and diffs status codes, JSON bodies
(modulo tokens and timestamps) and `Set-Cookie` flags. The existing `auth_flow.py smoke` is a
black-box HTTP client and must pass against the Node server **unchanged** — that is the
acceptance test for the whole migration.

Grafana carries both `service.name`s, so counters can be compared under identical traffic.
