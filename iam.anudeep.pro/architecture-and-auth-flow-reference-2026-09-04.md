# iam.anudeep.pro — Architecture & Auth Flow Reference

**Date:** 2026-09-04
**Repo:** `github.com/AnudeepChPaul/iam.anudeep.pro`
**Branch at capture:** `main` @ `8e926f9`
**Stack:** Python 3.13 · FastAPI · SQLAlchemy 2.0 (async / asyncpg) · Alembic · PostgreSQL 16 · Redis 7 · argon2-cffi · pyotp · structlog · uvicorn · Docker/Railway/Cloudflare

---

## a) What am I trying to do

Document, end to end and at implementation depth, what this repository is and how it works — so that anyone picking it up can answer, without reading the source:

1. **What it is.** A standalone identity provider for the `anudeep.pro` estate. It owns admin accounts, passwords, TOTP second factors, recovery codes, live sessions, the interactive login/register HTML pages, and one introspection endpoint (`POST /v1/validate`) that every other service on the estate calls to resolve a token into a user.
2. **What the fundamental design decisions are** and *why* they were made — opaque sessions in Redis rather than JWTs, fingerprint binding rather than refresh-token rotation, a two-leg login rather than one, dual cookie+bearer delivery on every response.
3. **The exact wire contract** — every route, request body, response body, header and status code, with runnable dummy request/response pairs.
4. **The data model, the runtime topology, and the operational surface** (migrations, env vars, CLI, health probes, deploy).

The intent is a reference document, not a change proposal. Section (b) reconstructs the design space the codebase has already chosen from, so the "why" behind the current shape is legible; section (c) is the full technical description of what is actually built.

---

## b) What are my options

The repo has already resolved four architectural forks. Each is recorded here with the trade it made, because the current code only makes sense against the alternatives it rejected.

### Option 1 — Stateless signed tokens (JWT)

The service signs a JWT carrying `sub`, `exp`, `epoch`. Every downstream service verifies the signature locally with a shared public key. No lookup, no session store.

```
┌──────────┐   login    ┌───────────┐
│ Browser  │───────────▶│    IAM    │──── signs JWT (RS256)
└──────────┘◀───────────└───────────┘
     │  Bearer eyJhbGciOi…
     ▼
┌──────────────┐  verifies signature locally, no network call
│ api.anudeep  │  (holds JWKS)
└──────────────┘
```

**Pros**
- Zero-latency validation; downstream services never call IAM, so IAM is not on the hot path and not a single point of failure for reads.
- Horizontally trivial — no shared state at all.
- Standard, well-tooled (`pyjwt` is already a dependency, a leftover from this era).

**Cons**
- **Revocation is a lie.** A signed token is valid until `exp` no matter what the server thinks. Logout-everywhere, a password change, or a detected theft cannot take effect until expiry — which forces short expiries plus a refresh-token dance to make them tolerable.
- **An idle window is impossible** without reissuing on every single request, because the expiry is baked inside the signature.
- **The claims are readable** by whoever holds the token. `sub` and `email` leak from any log line, proxy cache, or browser extension that sees the header.
- Key distribution and rotation become an operational surface of their own.

### Option 2 — Opaque token in Postgres

A random token whose sha256 is a primary key in a `session` table, with an `expires_at` column.

```
┌──────────┐   login    ┌───────────┐   INSERT session
│ Browser  │───────────▶│    IAM    │──────────────▶ ┌──────────┐
└──────────┘◀── anut_…  └───────────┘                │ Postgres │
                             ▲   UPDATE expires_at   └──────────┘
                             │   on every request
                        (+ a cron to sweep dead rows)
```

**Pros**
- Fully authoritative and instantly revocable — delete the row.
- One datastore; no second piece of infrastructure to run, back up, or lose.
- Session history is queryable with ordinary SQL.

**Cons**
- **A write per read.** Sliding the idle window means `UPDATE session SET expires_at = …` on *every* authenticated request. That is a row lock and a WAL write for something worthless in thirty minutes.
- Needs a sweeper job for expired rows, i.e. a cron for garbage.
- Table bloat and vacuum pressure on the hottest path in the system.

### Option 3 — Opaque token in Redis, fingerprint-bound  ✅ **selected**

A 256-bit opaque token; its sha256 is a Redis hash key. The **idle window *is* the key's TTL**. An `absolute_expiry` timestamp inside the record caps total lifetime independently of activity. The session is additionally bound to a digest of (IP network, User-Agent).

```
                       ┌──────────────────────────────────────┐
   login               │  Redis                               │
┌──────────┐  anut_…   │  session:<sha256>  HASH  TTL=1800s   │
│ Browser  │◀──────────│    user_id, epoch, fingerprint,      │
└──────────┘           │    issued_at, absolute_expiry        │
      │ Bearer/Cookie  │  sessions:user:<uuid>  SET           │
      ▼                │  pending:<sha256>  HASH  TTL=300s    │
┌──────────────┐       └──────────────────────────────────────┘
│ api.anudeep  │──── POST /v1/validate (service + Bearer) ──▶ IAM
└──────────────┘◀─── {"active": true, "sub": …, "epoch": …}
                                    │
                              ┌──────────┐  admin_user, recovery_code,
                              │ Postgres │  login_attempt  (schema "iam")
                              └──────────┘
```

**Pros**
- **The idle window costs one `EXPIRE`**, not a row update. Expiry is the datastore's own semantics; nothing has to sweep.
- **Authoritative and instant** — revocation is a `DEL`; a logout-everywhere is a pipelined delete over the `sessions:user:<id>` set (no `SCAN` across a shared keyspace).
- **The token carries nothing.** It is random bytes; only its digest is stored. A dump of Redis is not replayable, and a leaked token reveals no identity.
- **Fingerprint binding** catches a credential lifted from a log on its *first* use elsewhere, which is what refresh-token rotation's reuse detection was trying to approximate on a race.
- Both TTLs are per-deployment configuration, validated at boot.

**Cons**
- IAM is on the hot path: every downstream authorisation is a network call to `/v1/validate`.
- Redis becomes availability-critical — losing it signs everyone out (mitigated with `appendonly yes` / `appendfsync everysec`).
- Two datastores to operate instead of one.

### Option 4 — Delegate entirely to Cloudflare Access

No local identity at all; Access is the gate.

**Pros:** almost no code; MFA, device posture and audit come free.
**Cons:** a misconfigured Access policy is then the *only* thing between the internet and the admin; no local notion of "who" for authorship/attribution; the estate cannot run without the vendor.

**The codebase takes both:** the login page lives *behind* Access, not instead of it — "two independent gates, so a misconfigured Access policy is not on its own enough to hand someone the admin" (`routers/login.py`). `OriginLockMiddleware` enforces that traffic arrived through the proxy.

### Decision summary

| Fork | Chosen | Rejected | Because |
|---|---|---|---|
| Token format | Opaque, 256-bit, `anut_` prefix | JWT | Instant revocation + a real idle window + no readable claims |
| Session store | Redis (TTL = idle window) | Postgres rows + cron | A write per read, and a sweeper, for data worthless in 30 min |
| Theft defence | Fingerprint binding (IP `/24` or `/64` + UA) | Refresh-token rotation & reuse detection | Catches theft on first use, not on a race; `refresh_token` was dropped in migration `0002` |
| Login shape | Two legs (password → handle → TOTP) | One leg carrying all three fields | Leg two never sees the password; the handle is single-use, revocable and fingerprint-bound |
| Delivery | Cookies **and** bearer on every response | A `mode` request field | A caller could ask for the browser-hostile shape by accident |
| Perimeter | Cloudflare Access **+** local login | Either alone | Two independent gates |

---

## c) The selected design, in full

### c.a) System design

#### c.a.1 Runtime topology

```
                    Internet
                        │
              ┌─────────▼──────────┐
              │  Cloudflare        │  WAF · rate limiting · Access policies
              │  (proxy + Access)  │  injects: CF-Connecting-IP, cf-origin-auth
              └─────────┬──────────┘
                        │ HTTPS
              ┌─────────▼──────────────────────────────────────┐
              │ Railway container                              │
              │  uvicorn --workers 2 --proxy-headers           │
              │  ┌──────────────────────────────────────────┐  │
              │  │ FastAPI app (anudeep.iam.main:app)       │  │
              │  │                                          │  │
              │  │  Middleware (outermost → innermost):     │  │
              │  │   1. OriginLockMiddleware   (prod only)  │  │
              │  │   2. TrustedHostMiddleware               │  │
              │  │   3. RequestContextMiddleware (req id)   │  │
              │  │   4. CORSMiddleware                      │  │
              │  │   5. SecurityHeadersMiddleware (CSP…)    │  │
              │  │                                          │  │
              │  │  Routers:                                │  │
              │  │   health   /healthz /readyz              │  │
              │  │   v1       /v1/register /v1/login …      │  │
              │  │   auth     /api/v1/auth/*                │  │
              │  │   login    /login /register (HTML)       │  │
              │  │   validate /v1/validate                  │  │
              │  │   docs     /docs        (non-prod only)  │  │
              │  └───────┬──────────────────────┬───────────┘  │
              └──────────┼──────────────────────┼──────────────┘
                         │                      │
                ┌────────▼────────┐    ┌────────▼─────────┐
                │ PostgreSQL 16   │    │ Redis 7          │
                │ schema "iam"    │    │ AOF everysec     │
                │ admin_user      │    │ session:<digest> │
                │ recovery_code   │    │ sessions:user:*  │
                │ login_attempt   │    │ pending:<digest> │
                └─────────────────┘    │ pending:spent:*  │
                                       └──────────────────┘
```

> **Middleware ordering note.** Starlette runs `add_middleware` registrations bottom-up, so the *last* registered runs *first*. `OriginLockMiddleware` is added last precisely so it "must reject before anything else does work" (`main.py:50`).

#### c.a.2 Layered code architecture

```
 HTTP boundary          routers/        v1.py  auth.py  login.py  validate.py  health.py  docs.py
      │                                 (own the commit; own cookies; own HTML)
      ▼
 Wiring                 deps.py         SettingsDep · SessionDep · AuthServiceDep
      │                                 CallerIdentity · BearerToken · CurrentUser
      ▼
 Domain logic           services/       auth.py (AuthService)  session.py (SessionStore)
      │                                 pending.py (PendingAuthStore)
      ▼
 Data access            repositories/   AdminUserRepository · RecoveryCodeRepository
      │                                 LoginAttemptRepository
      ▼
 Persistence            models/         Base · AdminUser · RecoveryCode · LoginAttempt
                        db.py           async engine + per-request AsyncSession
                        redis_client.py Redis singleton

 Cross-cutting          security/       tokens · passwords · fingerprint · cookies · csrf · headers
                        errors.py       RFC 9457 Problem Details
                        logging.py      structlog + X-Request-ID
                        settings.py     pydantic-settings, IAM_ prefix
                        templates.py    inline HTML/CSS/JS for the login page
                        qr.py           inline SVG QR for the otpauth URI
```

**Transaction rule:** routers own `await session.commit()`. The single deliberate exception is `AuthService._record()`, which commits the `login_attempt` row immediately — "a failed login raises, the request session rolls back, and the attempt row goes with it — leaving the rate limiter counting to zero forever."

#### c.a.3 Data model (PostgreSQL, schema `iam`)

```
┌──────────────────────────────────────────────┐
│ iam.admin_user                               │
├──────────────────────────────────────────────┤
│ id             UUID  PK  gen_random_uuid()   │
│ email          CITEXT   UNIQUE  NOT NULL     │◀── case-insensitive by type
│ password_hash  TEXT     NOT NULL             │    argon2id PHC string
│ totp_secret    TEXT     NULL                 │    NULL = not enrolled
│ totp_opt_in    BOOL     NOT NULL  DEFAULT t  │    distinguishes "wants one,
│ token_epoch    INT      NOT NULL  DEFAULT 0  │    hasn't set it up" from "declined"
│ last_login_at  TIMESTAMPTZ NULL              │
│ created_at     TIMESTAMPTZ NOT NULL now()    │
└───────┬──────────────────────────────────────┘
        │ 1:N  ON DELETE CASCADE
┌───────▼──────────────────────────────────────┐
│ iam.recovery_code                            │
├──────────────────────────────────────────────┤
│ id         UUID  PK                          │
│ user_id    UUID  FK → admin_user.id          │
│ code_hash  TEXT  NOT NULL   (argon2, like a password)
│ used_at    TIMESTAMPTZ NULL (single use)     │
│ created_at TIMESTAMPTZ NOT NULL              │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│ iam.login_attempt        (no FK — audit)     │
├──────────────────────────────────────────────┤
│ id          BIGINT  PK  IDENTITY ALWAYS      │
│ email       CITEXT  NULL                     │
│ ip          INET    NOT NULL                 │
│ success     BOOL    NOT NULL                 │
│ user_agent  TEXT    NULL                     │
│ fingerprint VARCHAR(64) NULL  (success only) │◀── the ONLY place the readable
│ created_at  TIMESTAMPTZ NOT NULL             │    ip/agent are kept beside the digest
├──────────────────────────────────────────────┤
│ INDEX login_attempt_ip_idx    (ip, created_at DESC)    │
│ INDEX login_attempt_email_idx (email, created_at DESC) │
└──────────────────────────────────────────────┘
```

**Extensions required:** `pgcrypto` (for `gen_random_uuid()`), `citext`. Created `IF NOT EXISTS` in `0001`, so they are no-ops when the content API already made them.

**Why a dedicated `iam` schema:** production shares one Postgres instance with the content API. A schema plus a role per service keeps "sharing an instance" a hosting decision rather than a merged database.

**Migration history**

| Revision | Down-revision | What it does |
|---|---|---|
| `0001_iam` | — | Creates schema `iam`, extensions, and `admin_user`, `refresh_token`, `recovery_code`, `login_attempt` |
| `0002_sessions` | `0001_iam` | **Drops `refresh_token` entirely**; adds `login_attempt.fingerprint VARCHAR(64)`. Sessions move to Redis; rotation is replaced by fingerprint binding |
| `0003_totp_opt_in` | `0002_sessions` | Adds `admin_user.totp_opt_in BOOL NOT NULL DEFAULT true`. Existing rows default `true` — they were created under the mandatory regime |

#### c.a.4 Redis keyspace

| Key | Type | TTL | Fields |
|---|---|---|---|
| `session:{sha256(token)}` | HASH | `idle_ttl` (default 1800s), re-`EXPIRE`d on every use | `user_id`, `epoch`, `fingerprint`, `issued_at`, `absolute_expiry` |
| `sessions:user:{user_id}` | SET | `absolute_ttl` (default 1 209 600s = 14d) | member = each live session digest |
| `pending:{sha256(handle)}` | HASH | 300s (login) / 600s (enrolment) | `user_id`, `purpose`, `attempts`, optional `totp_secret`, `fingerprint` |
| `pending:spent:{sha256(handle)}` | STRING `"1"` | 900s | replay marker — lets a *replay* be distinguished from ordinary *expiry* |

**Why the absolute cap is a stored timestamp, not a TTL:** a TTL would be pushed out by every touch, which is exactly what the cap exists to survive.

#### c.a.5 Credential inventory

| Credential | Prefix | Entropy | Stored as | Lifetime |
|---|---|---|---|---|
| Session token | `anut_` | 32 bytes → `secrets.token_urlsafe(32)` | sha256 hex, Redis key | idle 30 min sliding, hard cap 14 d |
| Pending handle | `anup_` | 32 bytes | sha256 hex, Redis key | 300s (login) / 600s (enrol), single use, 5 wrong codes burns it |
| CSRF token | — | `token_urlsafe(32)` | cookie (readable) + echoed header | = idle TTL |
| Login form token | — | `token_urlsafe(24)` | httpOnly cookie, double-submit | 900s |
| Recovery code | — | `xxxx-xxxx-xxxx` grouped hex | argon2 in Postgres | single use, 10 per user |
| Password | — | ≥ 12 chars | argon2id `t=3, m=64 MiB, p=4` | rehashed on login when params change |
| TOTP secret | — | `pyotp.random_base32()` | plaintext `admin_user.totp_secret` | until re-enrolment; `valid_window=1` (±30s) |
| Build token | `anub_` | — | prefix constant only; the feature was removed | — |

**sha256, not argon2, for tokens and handles:** the input already carries 256 bits of entropy, so there is nothing to slow a guesser down for, and the second leg should not pay a KDF cost per attempt.

---

### c.b) Flow diagrams

#### Flow 1 — Two-leg sign-in (enrolled account, API client)

```
Client                    /v1/login              AuthService              Postgres        Redis
  │                          │                        │                      │              │
  │ POST {email,password}    │                        │                      │              │
  ├─────────────────────────▶│                        │                      │              │
  │                          ├─ _enforce_rate_limit ─▶│─ recent_failures ───▶│              │
  │                          │                        │◀── (by_ip, by_acct) ─┤              │
  │                          │                        │  429 if either trips │              │
  │                          ├─ users.by_email ──────▶│─────────────────────▶│              │
  │                          │        (None → dummy_verify(), then 401)      │              │
  │                          ├─ verify_secret(argon2) │                      │              │
  │                          │        (rehash if params moved on)            │              │
  │                          ├─ totp_enrolled? yes ──▶│                      │              │
  │                          ├─ _record(success) ────▶│─ INSERT + COMMIT ───▶│              │
  │                          ├─ _issue_pending ──────▶│                      │              │
  │                          │    handle=anup_…, digest=sha256(handle)       │              │
  │                          │    fingerprint = sha256(net(ip)\0 ua)         │              │
  │                          │                        ├─ HSET pending:… TTL 300 ───────────▶│
  │◀─ 200 {status:           │                        │                      │              │
  │    "totp_required",      │                        │                      │              │
  │    pending_handle}       │                        │                      │              │
  │                          │                        │                      │              │
  │ POST /v1/login/totp      │                        │                      │              │
  │  {pending_handle,        │                        │                      │              │
  │   totp_code}             │                        │                      │              │
  ├─────────────────────────▶│                        │                      │              │
  │                          ├─ _load_pending ───────▶│─ HGETALL pending:… ───────────────▶│
  │                          │    • missing + spent marker → PendingReplayedError → 401     │
  │                          │    • wrong purpose            → 401                          │
  │                          │    • fingerprint mismatch     → SPEND handle, then 401       │
  │                          ├─ pyotp.verify(window=1)│                      │              │
  │                          │    wrong → record_attempt(); ≥5 burns handle  │              │
  │                          ├─ pending.spend ───────▶│─ DEL + SETEX spent:… 900 ─────────▶│
  │                          ├─ users.mark_login ────▶│                      │              │
  │                          ├─ issue_session ───────▶│                      │              │
  │                          │    token=anut_…  digest=sha256(token)         │              │
  │                          │                        ├─ HSET session:… ; EXPIRE 1800 ─────▶│
  │                          │                        ├─ SADD sessions:user:<id> ──────────▶│
  │◀─ 200 {token, token_type,│                        │                      │              │
  │   email, token_issued_at,│  + Set-Cookie: __Host-anudeep_session  (httpOnly)            │
  │   csrf_token,expires_in} │  + Set-Cookie: __Host-anudeep_csrf     (readable)            │
```

#### Flow 2 — Registration → enrolment → signed in

```
POST /v1/register {email, password, totp_opt_in}
        │
        ├─ registration_mode gate:
        │     "open"   → allow
        │     "single" → allow only while users.count() == 0
        │     "closed" → 403 Forbidden
        ├─ len(password) < 12 → 422
        ├─ email exists → 409 "That address cannot be registered."   (no enumeration)
        │
        ├── totp_opt_in = false ──▶ INSERT user; mark_login; issue_session
        │                           200 {status:"ok", token, csrf_token, …}
        │
        └── totp_opt_in = true ───▶ INSERT user (totp_secret NULL)
                                    _issue_pending(purpose="enrol", ttl=600)
                                    Set-Cookie: anudeep_totp_pending
                                    200 {status:"totp_enrolment_required", pending_handle}
                                          │
        GET /v1/register/totp  (handle from body / X-Pro-Pending-Handle / cookie)
                                          │
                                    ensure_enrolment_secret(handle)
                                      • record.totp_secret set?  → return it unchanged (idempotent)
                                      • otherwise: pyotp.random_base32()
                                                   pending.set_secret(digest, secret)   ← server-side only
                                                   recovery.replace_all(10 argon2 hashes)
                                    200 {secret, otpauth_uri, recovery_codes[10]}
                                          │
        POST /v1/register/totp {totp_code, pending_handle?}
                                          │
                                    confirm_totp_enrolment
                                      • record.totp_secret is None → 401 "Start enrolment again."
                                      • pyotp verify fails → record_attempt() → 401
                                      • success: user.totp_secret = record.totp_secret
                                                 pending.spend(digest)
                                    issue_session(...)
                                    200 TokenResponse  + session/csrf cookies, pending cookie deleted
```

> The secret reaches `admin_user` **only** after a code generated from it verifies, so an abandoned enrolment leaves nothing half-configured. The candidate seed is parked in Redis, never handed to the browser as a cookie.

#### Flow 3 — Session resolution (every authenticated call, and `/v1/validate`)

```
                       resolve_session(token, ip, user_agent)
                                     │
                    digest = sha256(token)
                                     │
                        HGETALL session:{digest}
                          │                  │
                       empty              found
                          │                  │
                       return None           ├─ fingerprint.matches(record.fingerprint, ip, ua)?
                                             │       no → log "session_fingerprint_mismatch"
                                             │            REVOKE the session, return None
                                             │       yes ↓
                                             ├─ sessions.touch(digest, record)
                                             │       remaining = absolute_expiry - now
                                             │       ≤ 0 → revoke, return None
                                             │       > 0 → EXPIRE key min(idle_ttl, remaining)
                                             ├─ users.by_id(record.user_id)
                                             │       None → revoke, return None
                                             ├─ user.token_epoch == record.epoch?
                                             │       no → revoke, return None   (logout-all / pw change)
                                             └─ return AdminUser
```

**Deliberately one answer for every failure.** Unknown, idled out, past its ceiling, revoked, and replayed-from-elsewhere are indistinguishable to a caller, so the endpoint cannot classify a stolen token. Only the fingerprint mismatch is logged, because that one is a theft signal rather than ordinary expiry.

**Fingerprint computation** (`security/fingerprint.py`):
```
network_of(ip)  →  IPv4 /24   ("203.0.113.7"  → "203.0.113.0/24")
                   IPv6 /64   ("2001:db8::1"  → "2001:db8::/64")
                   unparseable → the stripped string itself

compute(ip, ua) = sha256( network_of(ip).encode() + b"\x00" + ua.strip().encode() ).hexdigest()
matches(stored, ip, ua) = hmac.compare_digest(stored, compute(ip, ua))
```
The prefix (not the exact address) is bound because phones move between cell and wifi and IPv6 privacy extensions rotate hourly; exact binding turns those into surprise logouts. The `\x00` separator prevents `ip="1.2.3.4" + ua="0Mozilla"` colliding with `ip="1.2.3.40" + ua="Mozilla"`.

#### Flow 4 — Interactive login page (works with JS disabled)

```
GET /login ──▶ pending cookie present and resolvable?
                 │ yes → 303 to /login/authenticator or /login/setup
                 │ no  → render step "password"
                          • mint CSP nonce → request.state.csp_nonce
                          • mint/reuse form_token cookie (__Host-anudeep_login_form, 900s, Path=/)
                          • Cache-Control: no-store
                          • CSP: default-src 'none'; style-src 'nonce-…'; script-src 'nonce-…';
                                 connect-src 'self'; form-action 'self'; frame-ancestors 'none'

POST /login  ──▶ _check_form_token:
                   1. origin_allowed(request, admin_origins ∪ base_url)  else 403
                   2. if X-Requested-With: fetch  → skip double-submit (a custom header
                      already proves it was not a cross-site form post)
                      else compare_digest(form_token cookie, posted form_token) else 403
                 ──▶ safe_next(next): a path matching ^/[A-Za-z0-9._~!$&'()*+,;=:@/?%#\[\]-]*$,
                     or an absolute URL whose origin ∈ admin_origins; anything else →
                     settings.login_redirect_to.  "//evil.example" is rejected explicitly.
                 ──▶ service.login(...)
                       session issued  → _finish: 303 (or JSON {status,redirect}) + cookies
                       totp_required   → set pending cookie (300s), 303 → /login/authenticator
                       enrolment req.  → set pending cookie (600s), 303 → /login/setup
                       AuthError/429   → re-render the same step with exc.detail, status = exc.status_code

Steps: /login (password) · /register · /login/authenticator (code) · /login/setup (QR + recovery codes)
Each step is its own URL — reloadable, bookmarkable, and the address bar tracks where you are.
Only that step's form is emitted; the others are absent, not hidden.
```

#### Flow 5 — Service-to-service introspection

```
End user ──(browser, holds anut_… )──▶ api.anudeep.pro
                                            │
                                            │ POST https://iam.anudeep.pro/v1/validate
                                            │ Authorization: Bearer anut_<the user's token>
                                            │ X-Pro-Client-Ip:      <THE END USER's ip>
                                            │ X-Pro-User-Agent:     <THE END USER's UA>
                                            │ X-Pro-Token-Issued-At: <optional pin>
                                            ▼
                                          IAM
                                            ├─ require_bearer_token → 401 if absent
                                            ├─ introspect → resolve_session (Flow 3)
                                            ├─ inactive → 200 {"active": false}
                                            ├─ issued_at pin mismatch (>1s) → 200 {"active": false}
                                            └─ 200 {active, sub, email, epoch,
                                                    issued_at, expires_at, totp_enrolled}
```

**The user token identifies the subject; the service credential authenticates the caller.** The
backend sends `X-IAM-Service-Id` and `X-IAM-Service-Token`, which are stored only in its server
environment. IAM rejects unregistered callers and applies a Redis-backed rate limit per service.
The service credential is never exposed to browser clients.

**200 `{"active": false}` rather than 401** for a dead token: that is the answer to the question asked, not a failure to ask it. 401 is reserved for a request presenting no bearer token at all.

**Sending the *service's* own IP/UA is the classic misuse** — every session then looks alike and every answer comes back inactive.

#### Flow 6 — Rate limiting / lockout

```
recent_failures(email, ip, window=900s)  →  (by_ip, by_account)

  for each of {ip, email}:
      last_success = MAX(created_at) WHERE col=value AND success AND created_at >= now-900s
      floor        = max(now-900s, last_success)
      count        = COUNT(*)  WHERE col=value AND NOT success AND created_at > floor

  trip if  by_account >= IAM_LOGIN_MAX_ATTEMPTS      (default 5)
        or by_ip      >= IAM_LOGIN_MAX_ATTEMPTS_PER_IP (default 20)
  → 429 + Retry-After: 900
```

Two thresholds, not one: per-IP alone is defeated by a botnet; per-account alone lets an attacker lock a user out by guessing at their email; one shared limit punishes a single person on one machine trying two addresses — and behind Cloudflare or a NAT, one clumsy colleague would lock out everyone.

**A success clears the slate** (the `floor` is the last success), so an occasional typo does not accumulate across a whole window, and a repeated automated check cannot lock itself out.

---

### c.c) Subs and classes — the complete API surface

#### c.c.1 Route inventory

| Method | Path | Router | Auth | Purpose |
|---|---|---|---|---|
| GET | `/healthz` | health | none | Liveness. Touches nothing else |
| GET | `/readyz` | health | none | Readiness: `SELECT 1` + Redis `PING`; 503 `{"status":"degraded"}` if either fails |
| POST | `/v1/register` | v1 | none (mode-gated) | Create an account |
| GET | `/v1/register/totp` | v1 | enrol handle | Fetch secret + otpauth URI + recovery codes (idempotent) |
| POST | `/v1/register/totp` | v1 | enrol handle | Confirm the code, activate TOTP, sign in |
| POST | `/v1/login` | v1 | none | Leg one: email + password |
| POST | `/v1/login/totp` | v1 | pending handle | Leg two: handle + 6-digit code |
| POST | `/v1/logout` | v1 | bearer or cookie | Drop the presented session. 204, silent |
| POST | `/v1/validate` | validate | `Bearer` + service credential | RFC 7662-shaped introspection |
| POST | `/api/v1/auth/login/recovery` | auth | none | Sign in with a printed recovery code |
| POST | `/api/v1/auth/logout-all` | auth | `CurrentUser` | Bump epoch + revoke every session. 204 |
| GET | `/api/v1/auth/me` | auth | `CurrentUser` | Own identity |
| GET/POST | `/login` | login | form token | Password step (HTML) |
| GET/POST | `/login/authenticator` | login | form token + pending cookie | TOTP code step (HTML) |
| GET/POST | `/login/setup` | login | form token + pending cookie | Enrolment step: QR, key, recovery codes (HTML) |
| GET/POST | `/register` | login | form token | Registration form (HTML) |
| GET | `/docs`, `/openapi.json` | docs | — | **Non-production only.** In prod `docs_url`, `redoc_url`, `openapi_url` are all `None` |

#### c.c.2 Classes and functions, by module

**`settings.py`**
```python
class Settings(BaseSettings):                    # env_prefix "IAM_", .env file, extra="ignore"
    environment: Literal["dev","prod"] = "dev"
    log_level: str = "INFO"
    database_url: str                            # required
    redis_url: str = "redis://localhost:6379/0"
    db_pool_size: int = 5;  db_max_overflow: int = 5;  db_echo: bool = False
    session_idle_ttl_seconds: int     = 1800      (ge=60)
    session_absolute_ttl_seconds: int = 1_209_600 (ge=300)
    session_secret: SecretStr                    # required
    admin_origins: list[AnyHttpUrl] = []
    trusted_hosts: list[str] = ["*"]
    origin_auth_header: SecretStr | None = None
    login_max_attempts: int = 5
    login_max_attempts_per_ip: int = 20
    login_window_seconds: int = 900
    login_redirect_to: str = "/docs"
    registration_mode: Literal["single","open","closed"] = "single"

    @model_validator(mode="after")  _ceiling_outlives_the_window()
        # absolute < idle would make the idle window unreachable → refuse at boot
    @field_validator("database_url")  _coerce_asyncpg()
        # Railway hands out postgres://  →  postgresql+asyncpg://
    @field_validator("admin_origins","trusted_hosts", mode="before")  _parse_json_list()
        # accepts a JSON array OR a comma-separated string
    @property is_prod

@lru_cache(maxsize=1) def get_settings() -> Settings
```

**`main.py`**
```python
@asynccontextmanager async def lifespan(app)     # logs startup; disposes engine + redis on shutdown
def create_app(settings: Settings | None = None) -> FastAPI
app = create_app()
```

**`db.py` / `redis_client.py`**
```python
@lru_cache def get_engine() -> AsyncEngine         # pool_pre_ping=True
@lru_cache def get_sessionmaker() -> async_sessionmaker[AsyncSession]   # expire_on_commit=False, autoflush=False
async def get_session() -> AsyncIterator[AsyncSession]   # one per request; rollback on any exception
@lru_cache def get_redis() -> redis.Redis          # decode_responses=True, 3s timeouts, health_check_interval=30
```

**`deps.py`**
```python
SettingsDep    = Annotated[Settings,     Depends(get_settings)]
SessionDep     = Annotated[AsyncSession, Depends(get_session)]
AuthServiceDep = Annotated[AuthService,  Depends(get_auth_service)]
CallerIdentity = Annotated[tuple[str, str|None], Depends(caller_identity)]
BearerToken    = Annotated[str,          Depends(require_bearer_token)]
CurrentUser    = Annotated[AdminUser,    Depends(current_user)]

def get_auth_service(session, settings) -> AuthService     # constructs the whole graph
def client_ip(request) -> str                              # CF-Connecting-IP wins, else request.client.host
def caller_identity(request) -> tuple[str, str|None]       # X-Pro-Client-Ip / X-Pro-User-Agent win
def bearer_token(request) -> str | None
def require_bearer_token(request) -> str                   # 401 + WWW-Authenticate: Bearer if absent
async def current_user(request, service, settings) -> AdminUser
    # bearer path: no CSRF (no ambient authority)
    # cookie path + unsafe method: origin_allowed() AND csrf_token_valid() else 403
```

**`services/session.py`**
```python
DEFAULT_IDLE_TTL_SECONDS      = 1800
DEFAULT_ABSOLUTE_TTL_SECONDS  = 1_209_600
_KEY     = "session:{digest}"
_BY_USER = "sessions:user:{user_id}"

@dataclass(frozen=True, slots=True)
class SessionRecord:
    user_id: str; epoch: int; fingerprint: str
    issued_at: datetime; absolute_expiry: datetime

class SessionStore:
    def __init__(client, *, idle_ttl_seconds, absolute_ttl_seconds)
    async def create(*, digest, user_id, epoch, fingerprint) -> SessionRecord   # pipelined HSET+EXPIRE+SADD+EXPIRE
    async def get(digest) -> SessionRecord | None
    async def touch(digest, record) -> bool     # False (and revokes) once past absolute_expiry
    async def revoke(digest, user_id) -> None
    async def revoke_all(user_id) -> int        # reads the SET; never SCANs the keyspace
    async def count_for(user_id) -> int
```

**`services/pending.py`**
```python
PURPOSE_LOGIN="login"; PURPOSE_ENROL="enrol"
PENDING_TTL_SECONDS=300; ENROLMENT_TTL_SECONDS=600; MAX_ATTEMPTS=5; _SPENT_TTL_SECONDS=900

@dataclass(frozen=True, slots=True)
class PendingRecord:
    user_id: str; purpose: str; totp_secret: str|None; attempts: int; fingerprint: str|None = None

class PendingReplayedError(Exception): ...     # distinct from "unknown", which is usually expiry

class PendingAuthStore:
    async def create(*, digest, user_id, purpose, ttl_seconds, totp_secret=None, fingerprint=None)
    async def get(digest) -> PendingRecord | None        # raises PendingReplayedError on a spent marker
    async def set_secret(digest, secret)                 # only writes if the key still EXISTS
    async def record_attempt(digest) -> int              # spends the handle at MAX_ATTEMPTS
    async def spend(digest)                              # DEL + SETEX pending:spent:… 900 "1"
    async def ping() -> bool                             # for /readyz; reports rather than raises
```

**`services/auth.py`**
```python
TOTP_ISSUER = "anudeep.pro"; RECOVERY_CODE_COUNT = 10

@dataclass(frozen=True, slots=True) class IssuedSession:        access_token, csrf_token, issued_at
@dataclass(frozen=True, slots=True) class IntrospectionResult:  active, user, epoch, issued_at, expires_at
@dataclass(frozen=True, slots=True) class LoginOutcome:         status, user, session, pending_handle

def _generate_recovery_codes(count=10) -> list[str]     # "xxxx-xxxx-xxxx" grouped hex

class AuthService:
    def __init__(*, session, settings, users, sessions, recovery, attempts, pending)

    # registration
    async def register(*, email, password, ip, user_agent, totp_opt_in=True)
             -> tuple[AdminUser, str|None, IssuedSession|None]
    def _require_registration_open(existing_accounts) -> None

    # login
    async def login(*, email, password, totp_code, ip, user_agent) -> LoginOutcome
    async def complete_totp(*, handle, totp_code, ip, user_agent) -> IssuedSession
    async def login_with_recovery_code(*, email, password, code, ip, user_agent) -> IssuedSession

    # TOTP enrolment
    async def pending_purpose(handle) -> str | None       # never raises; "no step" is not an error
    async def ensure_enrolment_secret(handle) -> tuple[AdminUser, str, str, list[str]]   # idempotent
    async def begin_totp_enrolment(handle)  -> tuple[AdminUser, str, str, list[str]]
    async def confirm_totp_enrolment(handle, *, code, ip=None, user_agent=None) -> AdminUser

    # ending a session
    async def logout(*, token) -> None
    async def logout_everywhere(user_id) -> None          # bump_epoch + revoke_all

    # introspection & sessions
    async def introspect(token, *, ip, user_agent) -> IntrospectionResult
    async def issue_session(user, *, ip, user_agent, parent_jti=None) -> IssuedSession
    async def resolve_session(token, *, ip, user_agent) -> AdminUser | None
    async def revoke_session(token) -> None

    # internals
    async def _issue_pending(user, *, purpose, ttl, ip, user_agent) -> str
    async def _load_pending(handle, *, purpose, ip=None, user_agent=None)
             -> tuple[str, PendingRecord, AdminUser]      # burns the handle on fingerprint mismatch
    async def _require_current_user(user_id, epoch) -> AdminUser
    def       _totp_valid(user, code) -> bool             # pyotp valid_window=1
    async def _enforce_rate_limit(*, email, ip) -> None
    async def _record(*, email, ip, success, user_agent) -> None   # commits immediately, on purpose
```

**`repositories/user_repo.py`**
```python
class AdminUserRepository:
    by_email(email) · by_id(uuid) · count() · create(*, email, password_hash, totp_opt_in=True)
    set_password_hash(user, hash) · bump_epoch(user_id) -> int   # one UPDATE … RETURNING
    mark_login(user)

class RecoveryCodeRepository:
    replace_all(user_id, hashes) · unused_for(user_id) · mark_used(code)

class LoginAttemptRepository:
    record(*, email, ip, success, user_agent, fingerprint=None)
    _failures_since_last_success(*, column, value, window_seconds) -> int
    recent_failures(*, email, ip, window_seconds) -> tuple[int, int]   # (by_ip, by_account)
```

**`security/`**
```python
tokens.py       SESSION_TOKEN_PREFIX="anut_"  PENDING_HANDLE_PREFIX="anup_"  BUILD_TOKEN_PREFIX="anub_"
                SESSION_COOKIE_BASE="anudeep_session"  HOST_PREFIX="__Host-"
                cookie_name(base, *, secure)          # __Host- only where it is honoured
                mint_session_token() -> (token, sha256hex)
                mint_pending_handle() -> (handle, sha256hex)
                hash_token(t) · hash_pending_handle(h)

passwords.py    _hasher = PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4)
                MIN_PASSWORD_LENGTH = 12
                hash_secret(s) -> PHC string
                verify_secret(hashed, s) -> (ok, rehashed_or_None)   # never raises on a wrong password
                dummy_verify()                                       # timing equalisation

fingerprint.py  IPV4_PREFIX=24  IPV6_PREFIX=64  _SEPARATOR=b"\x00"
                network_of(ip) · compute(ip, ua) · matches(stored, ip, ua)

cookies.py      PENDING_TOTP_BASE = "anudeep_totp_pending"
                names(settings) -> {"session","csrf","pending"}
                set_cookie(...)          # httponly, secure=is_prod, samesite="lax", path="/", NO Domain
                apply_session(response, issued, settings)   # session (httpOnly) + csrf (readable)
                clear_session(response, settings)

csrf.py         CSRF_COOKIE_BASE="anudeep_csrf"  CSRF_HEADER="X-CSRF-Token"
                SAFE_METHODS={"GET","HEAD","OPTIONS"}
                new_csrf_token() · csrf_token_valid(request, *, cookie_name) · origin_allowed(request, allowed)

headers.py      API_CSP · DOCS_CSP · LOGIN_CSP · STATIC_HEADERS · HSTS
                class SecurityHeadersMiddleware(BaseHTTPMiddleware)
                class OriginLockMiddleware(BaseHTTPMiddleware)      # EXEMPT_PATHS = {/healthz, /readyz}
```

**`errors.py`** — RFC 9457 Problem Details, one shape for the whole API.
```
AppError(500 internal_error)
 ├─ NotFoundError      404 not_found
 ├─ ConflictError      409 conflict
 │    └─ PublishBlocked      publish_blocked
 ├─ ValidationFailed   422 validation_failed
 ├─ AuthError          401 unauthenticated
 ├─ ForbiddenError     403 forbidden
 ├─ RateLimited        429 rate_limited
 ├─ DeployFailed       502 deploy_failed
 └─ PayloadTooLarge    413 payload_too_large

register_exception_handlers(app) installs handlers for:
    AppError · RequestValidationError · StarletteHTTPException · Exception
```

#### c.c.3 Response headers, always present

```
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(), interest-cohort=()
Content-Security-Policy: default-src 'none'; frame-ancestors 'none'; base-uri 'none'; form-action 'none'
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-site
X-Request-ID: <echoed or generated uuid4 hex>
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload      (prod only)
```
CSP is replaced per path for `/docs*` and the four `LOGIN_PATHS`, each with a per-request nonce.

---

### c.d) Dummy request / response pairs

#### 1. Register (opting into a second factor)
```http
POST /v1/register HTTP/1.1
Host: iam.anudeep.pro
Content-Type: application/json
X-Pro-Client-Ip: 203.0.113.7
X-Pro-User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)

{"email": "me@anudeep.pro", "password": "correct-horse-battery-staple", "totp_opt_in": true}
```
```http
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: __Host-anudeep_totp_pending=anup_9v3Q…; Max-Age=600; Path=/; HttpOnly; Secure; SameSite=Lax

{
  "status": "totp_enrolment_required",
  "email": "me@anudeep.pro",
  "pending_handle": "anup_9v3QYt2mKpX8sLdR7fH0jNbVcW1aZeUgTyIoP4qM6xE",
  "token": null,
  "token_issued_at": null,
  "csrf_token": null,
  "expires_in": null
}
```

#### 2. Fetch enrolment material
```http
GET /v1/register/totp HTTP/1.1
Host: iam.anudeep.pro
X-Pro-Pending-Handle: anup_9v3QYt2mKpX8sLdR7fH0jNbVcW1aZeUgTyIoP4qM6xE
```
```http
HTTP/1.1 200 OK

{
  "secret": "JBSWY3DPEHPK3PXPJBSWY3DPEHPK3PXP",
  "otpauth_uri": "otpauth://totp/anudeep.pro:me%40anudeep.pro?secret=JBSWY3DPEHPK3PXPJBSWY3DPEHPK3PXP&issuer=anudeep.pro",
  "recovery_codes": [
    "a1b2-c3d4-e5f6", "0f9e-8d7c-6b5a", "1122-3344-5566", "aabb-ccdd-eeff", "9876-5432-10fe",
    "dead-beef-cafe", "1357-9bdf-0246", "8ace-0246-8ace", "fedc-ba98-7654", "0011-2233-4455"
  ]
}
```
*Idempotent — reloading returns the same `secret` and an empty `recovery_codes` list, so a reload cannot silently invalidate codes already written down.*

#### 3. Confirm enrolment (also signs you in)
```http
POST /v1/register/totp HTTP/1.1
Content-Type: application/json
X-Pro-Client-Ip: 203.0.113.7
X-Pro-User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)

{"totp_code": "492013", "pending_handle": "anup_9v3QYt2mKpX8sLdR7fH0jNbVcW1aZeUgTyIoP4qM6xE"}
```
```http
HTTP/1.1 200 OK
Set-Cookie: __Host-anudeep_session=anut_kR7…; Max-Age=1800; Path=/; HttpOnly; Secure; SameSite=Lax
Set-Cookie: __Host-anudeep_csrf=Zx9Kq…; Max-Age=1800; Path=/; Secure; SameSite=Lax
Set-Cookie: __Host-anudeep_totp_pending=""; Path=/; Max-Age=0

{
  "token": "anut_kR7pXm2vQ9wLzNc4TbYh8JdF6sGaEuOi1x3ZrVnKt0M",
  "token_type": "Bearer",
  "email": "me@anudeep.pro",
  "token_issued_at": "2026-09-04T17:45:12.481930Z",
  "csrf_token": "Zx9KqTr4mVb8nLpW2sJdH6yGfEuAoI1c3ZxRvNkT0Mq",
  "expires_in": 1800
}
```

#### 4. Login, leg one
```http
POST /v1/login HTTP/1.1
Content-Type: application/json
X-Pro-Client-Ip: 203.0.113.7
X-Pro-User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)

{"email": "me@anudeep.pro", "password": "correct-horse-battery-staple"}
```
```http
HTTP/1.1 200 OK

{"status": "totp_required", "email": "me@anudeep.pro",
 "pending_handle": "anup_Lq8Xn3vZ…", "token": null, "token_issued_at": null,
 "csrf_token": null, "expires_in": null}
```

#### 5. Login, leg two
```http
POST /v1/login/totp HTTP/1.1
Content-Type: application/json
X-Pro-Client-Ip: 203.0.113.7
X-Pro-User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)

{"pending_handle": "anup_Lq8Xn3vZ…", "totp_code": "884120"}
```
```http
HTTP/1.1 200 OK
Set-Cookie: __Host-anudeep_session=anut_bT5…; Max-Age=1800; Path=/; HttpOnly; Secure; SameSite=Lax
Set-Cookie: __Host-anudeep_csrf=Wp2…; Max-Age=1800; Path=/; Secure; SameSite=Lax

{"token": "anut_bT5yUq9mHx2LcVnR7dJfK0sGaEwOi4z6ZpXvNbMt3Q",
 "token_type": "Bearer", "email": "me@anudeep.pro",
 "token_issued_at": "2026-09-04T18:02:44.117003Z",
 "csrf_token": "Wp2LqXr7mTb4nKpW8sJdH1yGfEuAoI5c9ZxRvNkT6Mq",
 "expires_in": 1800}
```

#### 6. Introspection — active
```http
POST /v1/validate HTTP/1.1
Host: iam.anudeep.pro
Authorization: Bearer anut_bT5yUq9mHx2LcVnR7dJfK0sGaEwOi4z6ZpXvNbMt3Q
X-Pro-Client-Ip: 203.0.113.7
X-Pro-User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)
X-Pro-Token-Issued-At: 2026-09-04T18:02:44.117003Z
```
```http
HTTP/1.1 200 OK

{
  "active": true,
  "sub": "0f6c3d2e-8a41-4b77-9c15-2ed4b8a10f93",
  "email": "me@anudeep.pro",
  "epoch": 0,
  "issued_at": "2026-09-04T18:02:44.117003Z",
  "expires_at": "2026-09-18T18:02:44.117003Z",
  "totp_enrolled": true
}
```

#### 7. Introspection — inactive (revoked, idled out, forged, stale pin, or replayed elsewhere)
```http
HTTP/1.1 200 OK

{"active": false, "sub": null, "email": null, "epoch": null,
 "issued_at": null, "expires_at": null, "totp_enrolled": null}
```

#### 8. Introspection — no bearer at all
```http
HTTP/1.1 401 Unauthorized
Content-Type: application/problem+json
WWW-Authenticate: Bearer

{"type": "https://anudeep.pro/problems/unauthenticated", "title": "Unauthenticated",
 "status": 401, "detail": "A bearer token is required.", "code": "unauthenticated",
 "request_id": "3f2a9c1b7e4d48a6b0c5e9f1d2a3b4c5"}
```

#### 9. Rate limited
```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
Retry-After: 900

{"type": "https://anudeep.pro/problems/rate_limited", "title": "Too Many Requests",
 "status": 429, "detail": "Too many failed attempts. Try again later.",
 "code": "rate_limited", "request_id": "9b1c…"}
```

#### 10. Validation failure
```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{"type": "https://anudeep.pro/problems/validation_failed", "title": "Validation Failed",
 "status": 422, "detail": "The request body failed validation.", "code": "validation_failed",
 "request_id": "aa77…",
 "errors": [{"code": "schema_invalid", "field": "totp_code",
             "message": "String should match pattern '^\\d{6}$'", "line": null, "hint": null}]}
```

#### 11. Registration conflict — deliberately non-enumerating
```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{"type": "https://anudeep.pro/problems/conflict", "title": "Conflict", "status": 409,
 "detail": "That address cannot be registered.", "code": "conflict", "request_id": "c0de…"}
```

#### 12. Authenticated account calls
```http
GET /api/v1/auth/me HTTP/1.1
Authorization: Bearer anut_bT5yUq9…
```
```http
HTTP/1.1 200 OK

{"id": "0f6c3d2e-8a41-4b77-9c15-2ed4b8a10f93", "email": "me@anudeep.pro",
 "totp_enrolled": true, "last_login_at": "2026-09-04T18:02:44.117003Z"}
```

```http
POST /api/v1/auth/logout-all HTTP/1.1
Cookie: __Host-anudeep_session=anut_bT5…; __Host-anudeep_csrf=Wp2…
Origin: https://admin.anudeep.pro
X-CSRF-Token: Wp2LqXr7mTb4nKpW8sJdH1yGfEuAoI5c9ZxRvNkT6Mq
```
```http
HTTP/1.1 204 No Content
Set-Cookie: __Host-anudeep_session=""; Path=/; Max-Age=0
Set-Cookie: __Host-anudeep_csrf=""; Path=/; Max-Age=0
```

#### 13. Recovery-code sign-in (one leg — the code *is* the second factor)
```http
POST /api/v1/auth/login/recovery HTTP/1.1
Content-Type: application/json

{"email": "me@anudeep.pro", "password": "correct-horse-battery-staple",
 "recovery_code": "a1b2-c3d4-e5f6"}
```
```http
HTTP/1.1 200 OK

{"token": "anut_Qz3…", "token_type": "Bearer", "email": "me@anudeep.pro",
 "token_issued_at": "2026-09-04T19:10:02.004411Z", "csrf_token": "Mn4…", "expires_in": 1800}
```
*Note: this route's `TokenResponse` is built without `token_issued_at` in `routers/auth.py:54` — it relies on the field's presence in the model. Worth a look if a caller pins `X-Pro-Token-Issued-At` after a recovery login.*

#### 14. Health probes
```http
GET /healthz  →  200  {"status": "ok"}

GET /readyz   →  200  {"status": "ok", "database": "ok", "redis": "ok"}
              →  503  {"status": "degraded", "database": "ok", "redis": "unreachable"}
```

#### 15. Origin lock rejection (production, request bypassing Cloudflare)
```http
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{"type": "https://anudeep.pro/problems/forbidden", "title": "Forbidden", "status": 403,
 "detail": "Direct origin access is not permitted.", "code": "forbidden"}
```

---

### c.e) Configuration reference

All variables carry the `IAM_` prefix and are read from the environment or `.env`.

| Variable | Default | Notes |
|---|---|---|
| `IAM_ENVIRONMENT` | `dev` | `prod` enables HSTS, `Secure`/`__Host-` cookies, JSON logs; disables `/docs` and `/openapi.json` |
| `IAM_LOG_LEVEL` | `INFO` | |
| `IAM_DATABASE_URL` | **required** | `postgres://` and `postgresql://` are coerced to `postgresql+asyncpg://` |
| `IAM_REDIS_URL` | `redis://localhost:6379/0` | local compose maps `6380:6379` |
| `IAM_DB_POOL_SIZE` / `IAM_DB_MAX_OVERFLOW` / `IAM_DB_ECHO` | `5` / `5` / `false` | |
| `IAM_SESSION_IDLE_TTL_SECONDS` | `1800` | ≥ 60. Sliding window |
| `IAM_SESSION_ABSOLUTE_TTL_SECONDS` | `1209600` | ≥ 300, **and ≥ idle** or boot fails |
| `IAM_SESSION_SECRET` | **required** | `openssl rand -base64 48` |
| `IAM_ADMIN_ORIGINS` | `[]` | JSON array or comma-separated. Drives CORS, the CSRF origin check, and `safe_next` |
| `IAM_TRUSTED_HOSTS` | `["*"]` | `TrustedHostMiddleware` |
| `IAM_ORIGIN_AUTH_HEADER` | unset | When set, `OriginLockMiddleware` requires header `cf-origin-auth` to match |
| `IAM_LOGIN_MAX_ATTEMPTS` | `5` | per account |
| `IAM_LOGIN_MAX_ATTEMPTS_PER_IP` | `20` | per IP, looser on purpose |
| `IAM_LOGIN_WINDOW_SECONDS` | `900` | also the `Retry-After` value |
| `IAM_LOGIN_REDIRECT_TO` | `/docs` | must be a local path or an origin in `admin_origins`. `/docs` is dev-only |
| `IAM_REGISTRATION_MODE` | `single` | `single` \| `open` \| `closed` |

> `.env.example` also carries commented-out `IAM_BUILD_TOKENS`, `IAM_CF_*` and `IAM_R2_*` entries. These are inherited from the content API and are **not** read by `Settings` — `extra="ignore"` swallows them silently. They are stale and safe to remove.

---

### c.f) Operations

```bash
make dev          # venv → docker compose (pg :5433, redis :6380) → .env → alembic upgrade head → uvicorn :8001
make db           # Postgres + Redis, waits for both healthchecks
make env          # writes .env with a freshly generated IAM_SESSION_SECRET
make migrate      # alembic upgrade head
make downgrade    # alembic downgrade base
make psql         # psql -U iam -d iam inside the container
make check        # ruff check + pytest   (what CI runs)
make fmt          # ruff format + ruff check --fix
make clean        # docker compose down -v  (destroys the volume)

# auth helpers (scripts/auth_flow.py, scripts/totp_qr.py)
make auth-enrol   # enrol the second factor, store the TOTP secret locally
make auth-login   # sign in, print a CSRF token
make auth-smoke   # assert the whole auth contract against a running server
make auth-curl    # write a cookie jar for curl
make auth-reset   # clear login failures after locking yourself out
make auth-qr      # render the TOTP secret as a QR code

# CLI
python -m anudeep.iam.cli create-user me@anudeep.pro           # bootstrap; prompts for the password
python -m anudeep.iam.cli clear-lockout [--email me@…]         # delete recorded failures
```

**Deploy (Railway).** `railway.toml` sets `preDeployCommand = "alembic upgrade head"` — migrations run *before* the new container takes traffic, never in the app lifespan "where restarts race each other and rollbacks get awkward." Healthcheck `/healthz`, 30s timeout, restart on failure, max 3 retries.

**Container.** Two-stage: `uv sync --frozen --no-dev` against `uv.lock`, then a `python:3.13-slim` runtime as UID 10001. `uvicorn --workers 2 --proxy-headers --forwarded-allow-ips *` on :8000.

**Testing.** `pytest` with `asyncio_mode = "auto"`; integration tests under `tests/integration/` (`test_auth.py`, `test_register.py`, `test_login_page.py`, `test_validate.py`) driven by `testcontainers[postgres]`. `mypy` runs in `strict` mode with the pydantic plugin. `ruff` selects `E,F,I,N,UP,B,A,C4,SIM,RUF,ASYNC,S` at line length 100.

---

### c.g) Invariants worth preserving

1. **Routers commit; services do not** — except `AuthService._record()`, which must outlive the rollback of a failed login or the rate limiter counts to zero forever.
2. **No credential is ever stored in a form it was presented in.** Passwords and recovery codes → argon2id. Tokens and handles → sha256. Fingerprints → sha256. `login_attempt` is the only place a readable IP/User-Agent is kept, and only for a *successful* attempt.
3. **A negative answer never says why.** `resolve_session` and `/v1/validate` collapse every failure mode into one response.
4. **`__Host-` only where it works.** The prefix requires `Secure`, and `Secure` cookies are dropped over plain HTTP — so a prefixed name in development means the browser silently stores nothing. `tokens.cookie_name(base, secure=)` is the single place that decides.
5. **CSRF applies to the cookie path only.** A bearer token carries no ambient authority; demanding a CSRF header there would only make scripts fake browser state.
6. **`X-Pro-Client-Ip` / `X-Pro-User-Agent` describe the END USER**, never the calling service. Getting this wrong makes every session look alike and every introspection return inactive.
7. **The candidate TOTP secret lives in Redis until confirmed**, never in a cookie and never in `admin_user`.
8. **`token_epoch` is the global kill switch.** Bumping it invalidates every issued token in one `UPDATE`; `logout_everywhere` bumps it *and* deletes the keys so the effect is immediate rather than on-next-request.
9. **Every browser-observable operation on the login page needs its own CSP directive** — a missing one fails only in a real browser, never in a test that speaks HTTP.
10. **Nothing is hardcoded.** Every knob is an `IAM_`-prefixed environment variable, and contradictory TTLs are refused at boot rather than discovered in production.
