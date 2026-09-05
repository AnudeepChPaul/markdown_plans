# iam.anudeep.pro — conformance to requirements.md

**Date:** 2026-09-04
**Repo:** `iam.anudeep.pro` (package `anudeep.iam`) @ `8d4e1f4`
**Checked against:** `requirements.md` §1
**Status:** Plan — not yet implemented

---

## a) What am I trying to do

Bring the running identity service into conformance with `requirements.md` §1, which was
written after the service was built and therefore overrules it where they disagree.

### a.1 Conformance audit

**Already correct — the model, which is the expensive part:**

| Requirement | Where |
|---|---|
| Python, FastAPI | `pyproject.toml`, `src/anudeep/iam/main.py` |
| A distinct TOTP secret per user | `services/auth.py` — `pyotp.random_base32()` per enrolment |
| Login issues a temporary token, in Redis | `services/pending.py` — `pending:<sha256>` |
| Login reports whether two-step is enabled | `totp_required` / `totp_enrolment_required` / `ok` |
| Second leg returns a 256-bit key for later calls | `security/tokens.py` — `anut_…`, opaque |
| Key identifies the user, fingerprint proves authenticity | `services/session.py` — exactly this split |
| Other services call IAM to validate | `POST /api/v1/validate`, service-authenticated |
| `IAM_REGISTRATION_MODE` gates registration | `settings.py` — mechanism present |

**Five divergences:**

| # | Required | Present | Severity |
|---|---|---|---|
| 1 | `/v1/register`, `/v1/register/totp`, `/v1/login`, `/v1/login/totp`, `/v1/logout` | `/api/v1/auth/*`, and **register has no API at all** — `/register` is a browser page only | Surface |
| 2 | Two-step is **opt-in** at registration | Mandatory for every account | **Behavioural** |
| 3 | Mode `single` | Mode `first-user` (same semantics) | Naming |
| 4 | `X-Pro-*` headers identify the user on both login legs | Honoured only on `/validate` and `/handoff/exchange` | **Behavioural** |
| 5 | `/v1/login` creates the fingerprint; `/v1/login/totp` matches it | Fingerprint created only when the session is issued, after the code — the two legs are not tied together | **Behavioural, security** |

### a.2 Two of these are mine to own

**#2 was my decision, not an oversight.** I argued for mandatory second factors when the login
was built. The requirements say opt-in and overrule that. Stating the cost once so it is a
decision rather than drift: an account that opts out is held by a password alone, so a leaked
or reused password takes it, and the fingerprint becomes the only remaining obstacle.

**#5 is a real weakness the requirements would have prevented.** As built, a pending handle
captured between the two legs can be completed from any machine — the attacker needs the handle
and a valid code, nothing else. Binding the handle to a fingerprint at leg one closes it.

---

## b) Options

### Option A — Additive `/v1`, keep `/api/v1/auth` as an alias

New routers at `/v1/*`; the old paths stay and delegate to the same service methods.

```
  client ──▶ /v1/login          ─┐
                                 ├──▶ AuthService.login()
  client ──▶ /api/v1/auth/login ─┘
```

| Pros | Cons |
|---|---|
| Nothing that exists today breaks | Two public surfaces for one operation |
| Reversible | OpenAPI lists each endpoint twice; ambiguous to a consumer |
| | Drift: a fix applied to one path and forgotten on the other |
| | Nothing consumes the old paths yet, so the compatibility it buys is worth nothing |

### Option B — Rename the existing router to `/v1`

Change `prefix="/api/v1/auth"` to `"/v1"`, add the two missing endpoints, amend the handlers.

```
  client ──▶ /v1/login ──▶ routers/auth.py (moved) ──▶ AuthService.login()
```

| Pros | Cons |
|---|---|
| One surface, smallest diff | The handlers keep a shape built for a different contract — `mode: cookie\|bearer` in the body, `LoginResponse` carrying four optional fields for four different callers |
| Existing tests mostly survive a path rewrite | The new request shape (`X-Pro-*`, `totp_opt_in`) is bolted onto that, making an already-branchy handler branchier |

### Option C — A purpose-built `/v1` API router; retire `/api/v1/auth` ✅

Write `routers/v1.py` against the contract in `requirements.md`, calling the same
`AuthService`. Delete `routers/auth.py`. The HTML pages keep their own routes and keep calling
the service directly, as they already do.

```
  browser ──▶ /login, /register           routers/login.py   (HTML, form + fetch)
                                                  │
  api     ──▶ /v1/login, /v1/register     routers/v1.py     (JSON, X-Pro-* headers)
                                                  │
                                                  ▼
                                          services/auth.py   AuthService
                                             │          │
                                   PendingAuthStore  SessionStore   (Redis)
                                             │          │
                                   AdminUserRepository …            (Postgres, schema iam)

  service ──▶ /api/v1/validate, /api/v1/handoff/exchange   (unchanged)
```

| Pros | Cons |
|---|---|
| One surface per audience: pages for browsers, `/v1` for APIs, `/api/v1` for services | Largest diff of the three |
| Request and response shapes match the requirements rather than being adapted to them | `routers/auth.py`'s tests are rewritten, not renamed |
| Drops `mode: cookie\|bearer` from the API — the pages need cookies, an API client needs a token, and the branch existed only because one router served both | |
| `AuthService`, `PendingAuthStore`, `SessionStore`, `fingerprint` are untouched — the model already conforms | |

**Chosen: C.** The service layer is right and stays; only the surface is wrong, so the surface
is what gets rebuilt. Option A pays a permanent maintenance cost for compatibility nothing is
asking for, and B keeps a handler shaped for a contract that no longer applies.

---

## c) Design of the selected option

### c.1 System design

```
                          ┌───────────────────────── iam.anudeep.pro ─────────────────────────┐
                          │                                                                   │
  browser  ──HTML────────▶│  routers/login.py      /login  /register                          │
                          │                        /login/authenticator  /login/setup         │
                          │        │                                                          │
  API      ──JSON────────▶│  routers/v1.py         /v1/register       /v1/register/totp       │
  (Bruno,  X-Pro-*        │        │               /v1/login          /v1/login/totp          │
   admin)                 │        │               /v1/logout                                 │
                          │        ▼                                                          │
                          │  services/auth.py  ── AuthService ──────────────────────┐         │
                          │        │                    │                  │        │         │
                          │        ▼                    ▼                  ▼        ▼         │
                          │  PendingAuthStore     SessionStore    user_repo   fingerprint     │
                          │   (Redis)              (Redis)         (Postgres)  (pure)         │
                          │                                                                   │
  service  ──X-Pro-*─────▶│  routers/validate.py   /api/v1/validate                           │
  (api.…)                 │  routers/handoff.py    /api/v1/handoff/exchange                   │
                          └───────────────────────────────────────────────────────────────────┘

  Redis                                     Postgres (schema iam)
    pending:<sha256(handle)>                  admin_user      id, email, password_hash,
      user_id, purpose, attempts,                             totp_secret?, token_epoch
      fingerprint   ← NEW (gap 5)             recovery_code   user_id, code_hash, used_at
      totp_secret?, audience?                 login_attempt   email, ip, user_agent,
    session:<sha256(token)>                                   fingerprint, success
      user_id, epoch, fingerprint,
      issued_at, absolute_expiry
```

### c.2 Flow diagram

```
  REGISTER, opting in                        REGISTER, opting out
  ───────────────────                        ────────────────────
  POST /v1/register                          POST /v1/register
    {email, password, totp_opt_in: true}       {email, password, totp_opt_in: false}
    X-Pro-Client-Ip, X-Pro-User-Agent          X-Pro-*
        │                                          │
        ├─ mode gate: open | single | closed       ├─ mode gate
        ├─ create admin_user                       ├─ create admin_user
        ├─ fp = fingerprint.compute(ip, ua)        ├─ fp = compute(ip, ua)
        ├─ pending{purpose: enrol, fp}             └─▶ session, token issued now
        ▼                                              200 {token, expires_in}
    200 {status: "totp_required",
         pending_handle}                        "otherwise, this step never comes"
        │
  POST /v1/register/totp
    {totp_code}  X-Pro-*
        │
        ├─ load pending, MATCH fp ──mismatch──▶ burn handle, 401
        ├─ verify code against the per-user secret
        ├─ store secret on admin_user
        └─▶ session
    200 {token, expires_in}


  LOGIN                                      LOGIN, no second factor
  ─────                                      ───────────────────────
  POST /v1/login  {email, password}          POST /v1/login
    X-Pro-*                                      │
        │                                        ├─ rate limit, verify password
        ├─ rate limit, verify password           └─▶ session
        ├─ totp_secret is set?                       200 {token, expires_in}
        ├─ fp = compute(ip, ua)
        ├─ pending{purpose: login, fp}
        ▼
    200 {status: "totp_required", pending_handle}
        │
  POST /v1/login/totp  {totp_code}  X-Pro-*
        │
        ├─ load pending, MATCH fp ──mismatch──▶ burn handle, 401
        ├─ verify code (window ±1 step)
        ├─ spend handle
        └─▶ SessionStore.create(token, fp)
    200 {token, expires_in: 1800}
        │
        ▼
  every later call, from any app:
    POST /api/v1/validate   X-Pro-Token, X-Pro-Client-Ip, X-Pro-User-Agent
        └─▶ {active, sub, email, epoch, issued_at, expires_at}
```

### c.3 Subs and classes

**New — `src/anudeep/iam/routers/v1.py`**

```python
router = APIRouter(prefix="/v1", tags=["identity"])

@router.post("/register",       response_model=StartResponse)   # 201 | 403 closed | 409 taken
@router.post("/register/totp",  response_model=TokenResponse)   # 200 | 401 bad code or fp
@router.post("/login",          response_model=StartResponse)   # 200 | 401 | 429
@router.post("/login/totp",     response_model=TokenResponse)   # 200 | 401 bad code or fp
@router.post("/logout",         status_code=204)
```

**New — `src/anudeep/iam/schemas/v1.py`**

```python
class RegisterRequest(BaseModel):
    email: EmailStr
    password: SecretStr                 # >= MIN_PASSWORD_LENGTH
    totp_opt_in: bool = True            # default on: omission takes the safer path

class LoginRequest(BaseModel):
    email: EmailStr
    password: SecretStr

class CodeRequest(BaseModel):
    totp_code: str = Field(pattern=r"^\d{6}$")
    pending_handle: str | None = None   # header-less clients may pass it here

class StartResponse(BaseModel):
    status: Literal["totp_required", "ok"]
    email: EmailStr
    pending_handle: str | None = None   # present only when status == totp_required
    token: str | None = None            # present only when status == ok

class TokenResponse(BaseModel):
    token: str                          # anut_… 256-bit opaque
    token_type: Literal["Bearer"] = "Bearer"
    expires_in: int                     # idle window, slides on every use
```

**Changed — `src/anudeep/iam/deps.py`**

```python
def caller_identity(request: Request) -> tuple[str, str | None]:
    """The end user's address and agent.

    X-Pro-Client-Ip / X-Pro-User-Agent when supplied, else the connection's own. A gateway
    that forwards neither fingerprints itself, every session looks alike, and the binding
    checks nothing — the same mistake /validate already has a test named after.
    """

CallerIdentity = Annotated[tuple[str, str | None], Depends(caller_identity)]
```

**Changed — `src/anudeep/iam/services/pending.py`**

```python
@dataclass(frozen=True, slots=True)
class PendingRecord:
    user_id: str
    purpose: str
    totp_secret: str | None
    attempts: int
    audience: str | None = None
    fingerprint: str | None = None            # NEW — closes gap 5

class PendingAuthStore:
    async def create(..., fingerprint: str | None = None) -> None: ...
```

**Changed — `src/anudeep/iam/services/auth.py`**

```python
class AuthService:
    async def register(self, *, email, password, totp_opt_in: bool,
                       ip, user_agent) -> tuple[AdminUser, str | None, IssuedSession | None]:
        """Opt-in returns a pending handle; opt-out returns a session outright."""

    async def login(self, *, email, password, ip, user_agent) -> LoginOutcome:
        """Unchanged apart from stamping the fingerprint onto the pending handle."""

    async def complete_totp(self, *, handle, totp_code, ip, user_agent) -> IssuedSession:
        """Now matches the handle's fingerprint before the code, and burns it on mismatch."""

    async def confirm_totp_enrolment(self, handle, *, code, ip, user_agent) -> AdminUser:
        """Same fingerprint check on the enrolment leg."""

    def _require_registration_open(self, existing_accounts: int) -> None:
        """open | single | closed  — 'first-user' renamed to 'single'."""
```

**Deleted:** `src/anudeep/iam/routers/auth.py`, `src/anudeep/iam/schemas/auth.py` (the parts
serving it), and the `mode: cookie|bearer` branch with them. `tests/integration/test_auth.py`
is rewritten against `/v1`.

**Untouched:** `security/fingerprint.py`, `security/tokens.py`, `services/session.py`,
`repositories/user_repo.py`, `routers/validate.py`, `routers/handoff.py`, `routers/login.py`.

---

## d) Verification

| Check | How |
|---|---|
| Paths exist | `GET /openapi.json` lists all five `/v1/*` endpoints and no `/api/v1/auth/*` |
| Register without a browser | `POST /v1/register` with JSON only — no cookie jar, no form token |
| Opt-out | `totp_opt_in: false` → `/v1/register` returns a token; `/v1/login` returns `ok` with a token and never asks for a code |
| Opt-in | `totp_opt_in: true` → no token until `/v1/register/totp` succeeds |
| `single` mode | Second registration refused; existing `TestTheGate` passes against the renamed value |
| **Fingerprint at leg one** | A handle from `/v1/login` presented to `/v1/login/totp` with a different `X-Pro-Client-Ip` is refused **and burned** — mirrors `test_a_mismatch_burns_the_session` |
| Headers honoured | Login with `X-Pro-Client-Ip: 203.0.113.9`, then `/api/v1/validate` with that same header succeeds and with the connection's address fails |
| Pages unaffected | The 39 tests in `test_login_page.py` and 25 in `test_register.py` pass unchanged |
| Nothing regressed | Full suite, currently 125 |

---

## e) Deliberately not changing

- **`/api/v1/validate` and `/api/v1/handoff/exchange`.** The requirements describe the user
  surface; these are service-to-service and their audience is other apps.
- **The HTML pages.** §1 lists an API; the pages are an additional surface. `/login` as a page
  beside `/v1/login` as an endpoint is clearer than overloading one path on content type.
- **Recovery codes, rate limiting, lockout, `logout-all`.** Not required, not contradicted.
  Removing working protection to match a document that is silent on it would be a misreading.

---

## f) Implementation status — delivered 2026-09-04, commit `213dc5b`

All five gaps closed. 132 tests pass (125 before, 7 added).

| Gap | Landed as |
|---|---|
| 1. Endpoint paths | `routers/v1.py` + `schemas/v1.py`; cookie helpers extracted to `security/cookies.py` so the page and API surfaces cannot drift apart |
| 2. Two-step optional | `totp_opt_in` column (migration `0003_totp_opt_in`), defaulting true; `register` returns a session directly on opt-out |
| 3. `first-user` → `single` | `settings.py`, `services/auth.py`, both `.env` templates, README |
| 4. `X-Pro-` on the login legs | `deps.caller_identity`, used by all four sign-in endpoints |
| 5. Fingerprint at leg one | `PendingRecord.fingerprint`, set in `_issue_pending`, matched in `_load_pending`, burning the handle on mismatch |

### Deviations from the plan as written

- **`routers/auth.py` was trimmed, not retired.** Retiring it would have deleted `/me`,
  `/logout-all` and `/login/recovery`, which section (e) says to keep. What was actually
  wrong was the sign-in half; that is what moved.
- **`/v1/register/totp` is `GET` for the QR and `POST` to confirm.** The requirements name
  one path for both. Two verbs on one path keeps that, and a body-less `GET` had nowhere to
  put the handle, so `X-Pro-Pending-Handle` was added to the existing header set.
- **`schemas/auth.py` lost seven now-dead models** (`AuthMode`, `TokenPair`, `LoginResponse`,
  `EnrollTotpResponse`, `ConfirmTotpRequest`, `LogoutRequest`, and the old `LoginRequest`).
- **`scripts/auth_flow.py`'s refresh-rotation section was dropped.** It had been dead since
  migration `0002` moved sessions to Redis — there is no fixed-expiry token left to rotate.

### Verification actually performed

Beyond the suite: each of the three new guards was mutated off in turn (`if False`), and each
failed a test that named it. Vacuous coverage was caught this way once — the burn test passed
with the guard disabled, because a successful first attempt spends the handle and the replay
is refused either way; it now asserts the first attempt was itself a 401.

The `/v1` flow was also exercised against a live instance, both paths: opt-out registering and
signing in on one leg, opt-in through QR and confirmation, and a second leg from a different
`X-Pro-Client-Ip` refused and burned.
