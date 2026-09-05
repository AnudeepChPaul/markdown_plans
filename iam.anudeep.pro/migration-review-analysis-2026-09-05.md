# Review of the Node Migration Plan — Nine Points Assessed

**Date:** 2026-09-05
**Reviewed against:** `iam.anudeep.pro_python` @ `c815c41` (branch `python`)
**Status:** Analysis only. No code changed.

---

## a) What am I trying to do

Nine review points were raised against the migration plan. Each is a claim about the system;
each was checked against the source rather than accepted or dismissed from memory.

That mattered. **Three of the nine came back different from how they were stated:**

- One rests on a premise that does not hold — the endpoint it describes does not exist.
- One has the wrong diagnosis but points at a **live vulnerability** in the running service.
- One is already implemented.

The remaining six are sound and accepted. The purpose of this document is to record what is
true, decide the contract questions the review opened, and leave the two real defects described
precisely enough to fix in one sitting.

---

## b) The nine points

### 1. "Resolve the `/v1/validate` vs `/api/v1/validate` inconsistency"

**Verdict: the premise does not hold. A different, real inconsistency exists underneath it.**

There is no `/api/v1/validate`. Introspection is served at exactly one path:

```python
# routers/validate.py:39
router = APIRouter(prefix="/v1", tags=["introspection"])
@router.post("/validate", response_model=ValidateResponse)
```

`grep -rn "api/v1/validate"` over the repository returns nothing. And "the
requirements-compliant behaviour" has no readable definition: `requirements.md` is cited in two
docstrings (`routers/v1.py:1`, `routers/auth.py:3`) but **is not in the repository**, so there
is no document to conform to.

The real inconsistency is that one service serves two prefixes:

| Prefix | Routes | Why |
|---|---|---|
| `/v1/*` | register, register/totp, login, login/totp, logout, validate | "The identity API named by requirements.md" |
| `/api/v1/auth/*` | me, logout-all, password, login/recovery | "operations that surround it and that the requirements are silent about" |

Deliberate, and explained in the docstrings — but it means `/v1/logout` and
`/api/v1/auth/logout-all` sit in different namespaces while doing adjacent jobs.

**Resolved by decision.** See (c.a).

---

### 2. "Add an endpoint matrix"

**Verdict: accepted.** No such table exists in any document. Written out in full at (c.b).

---

### 3. "Make Redis pending-token consumption atomic — replace conceptual DEL + SETEX"

**Verdict: the instinct is right, the diagnosis is wrong, and there is a real vulnerability.**

`spend()` is **already atomic**. `pipeline(transaction=True)` in redis-py issues `MULTI`/`EXEC`:

```python
# services/pending.py
async def spend(self, digest: str) -> None:
    """Single use. The marker outlives the record so a replay is still visible."""
    PENDING_HANDLE.add(event="spent")
    async with self.client.pipeline(transaction=True) as pipe:
        pipe.delete(_KEY.format(digest=digest))
        pipe.setex(_SPENT_KEY.format(digest=digest), _SPENT_TTL_SECONDS, "1")
        await pipe.execute()
```

It is not "conceptual DEL + SETEX"; the two commands already execute as one unit, and a Lua
script would add nothing.

**The defect is check-then-act around it.** `complete_totp` reads, verifies, then spends —
and never learns whether it was the caller that actually won:

```
_load_pending(handle)      HGETALL           non-destructive, so the 5-attempt budget works
_totp_valid(user, code)                      pure computation
pending.spend(digest)      MULTI[DEL,SETEX]  atomic — but the reply is discarded
issue_session(user)                          unconditional
```

```
   request A                          request B          (same handle, same valid code)
   ─────────                          ─────────
   HGETALL  → record                  HGETALL  → record        ← both read the live handle
   verify   → ok                      verify   → ok            ← same 30-second window
   MULTI[DEL→1, SETEX]                MULTI[DEL→0, SETEX]      ← B loses, and is not told
   issue_session ✓                    issue_session ✓          ← TWO sessions, one handle
```

The handle is single-use only against *sequential* callers. A handle intercepted in flight and
replayed concurrently with the legitimate request yields the attacker a session — precisely the
attack the fingerprint binding on `_load_pending` was added to make hard, defeated by timing
rather than by location.

**The fix needs no Lua.** `DEL` inside `MULTI/EXEC` already returns `1` to exactly one caller
and `0` to every other. `spend()` need only return it, and `complete_totp` refuse on `0`:

```python
results = await pipe.execute()
return bool(results[0])          # DEL's reply: 1 = this caller claimed it

# complete_totp
if not await self.pending.spend(digest):
    raise AuthError("Start again from the sign-in form.")
```

**The same shape exists in Postgres for recovery codes.** `login_with_recovery_code` iterates
`recovery.unused_for(user.id)` — `SELECT … WHERE used_at IS NULL` — verifies, then calls
`mark_used()`. Two concurrent requests with the same code both see it unused. The fix is to let
the database pick the winner:

```sql
UPDATE iam.recovery_code SET used_at = now()
 WHERE id = $1 AND used_at IS NULL
 RETURNING id;              -- no row returned ⇒ someone else spent it
```

Both are **live in the Python service today** and would have been ported faithfully as bugs.

---

### 4. "Clarify cookie parity"

**Verdict: accepted.** The observation is exactly right — `TokenResponse` and `StartResponse`
describe bodies, and nothing in the schemas says a `Set-Cookie` accompanies them. Every
issuance path calls `cookies.apply_session()`, which sets **two** cookies, and the cookie
contract is currently discoverable only by reading `security/cookies.py`. Documented at (c.c).

---

### 5. "Replace 'same OTLP output' with 'same semantic telemetry'"

**Verdict: accepted — the wording was mine and it was wrong.**

The migration document claims the OTLP output is unchanged. It cannot be:

- `service.name` is `iam.anudeep.pro-node`, deliberately, so both can report to one collector.
- SDK-generated spans differ. Python's `SQLAlchemyInstrumentor` emits `SELECT` / `INSERT`;
  `@opentelemetry/instrumentation-pg` emits `pg.query`. HTTP span names and attribute sets
  differ between the FastAPI and Fastify instrumentations.
- Resource attributes differ (`telemetry.sdk.language` is `python` versus `nodejs`).

What *is* reproducible, and what the port should be held to: the **counter names, their
attribute keys and their permitted values**, the hand-placed span names (`iam.login`,
`iam.session.resolve`, …), the histogram, the attribute rule, and log correlation by `trace_id`.
"Same semantic telemetry" is the correct claim; the document will be corrected.

---

### 6. "Strengthen parity testing — the smoke driver alone is insufficient"

**Verdict: accepted.** `auth_flow.py smoke` asserts 15 properties and is black-box, which makes
it a good acceptance gate and a poor differ: it cannot detect that two implementations agree on
status codes while disagreeing on a cookie flag or a JSON key.

Greenfield data makes deterministic fixtures cheap, because both databases can be seeded
identically. Design at (c.d).

---

### 7. "Pin versions; consider Node 24"

**Verdict: accepted.**

```
v24.20.0  (Latest LTS: Krypton)     ← installed during stage 1b
v22.23.2  (Maintenance LTS)         ← what the scaffold used until then
```

*Correction:* an earlier draft of this document said 24.20.0 was already installed. It was not
— the check that produced that claim grepped version strings out of `nvm ls`, which prints
unresolved aliases (`lts/* -> lts/krypton (-> N/A)`) alongside installed versions, so a version
that existed only as an alias target read as present. `nvm install 24.20.0` was needed.

Node 22 entered maintenance when 24 became Active LTS. For a service being written now there is
no argument for starting on the older line, and 24.20.0 is already on this machine, so the
change costs one `engines` field and one Docker base image.

On pinning: `pnpm-lock.yaml` already pins the full tree, so installs are reproducible today.
Pinning exactly in `package.json` as well removes the remaining case — `pnpm update` silently
taking a minor of `@node-rs/argon2` or a Fastify plugin. For a security service that is worth
the friction.

---

### 8. "Separate local and production Docker designs"

**Verdict: already implemented; documentation to strengthen.**

The `Dockerfile` is multi-stage with exactly this split:

| Target | Contents | Use |
|---|---|---|
| `builder` | pnpm install, `tsc` | — |
| `runtime` | Node + built app, non-root | **production** — Railway supplies managed Postgres and Redis |
| `allinone` *(default)* | `runtime` + Postgres + Redis + supervisor | development, demos, smoke |

The reasoning is in the file: the all-in-one cannot be deployed because its database lives on
container disk and is lost on every restart. What is missing is that this is not stated in the
README, so someone could reasonably `docker build .` and deploy the result.

---

### 9. "Add concurrency/security gates"

**Verdict: most already exist. Two real gaps, and one of them is point 3's.**

| Gate | Status |
|---|---|
| Registration race | **Covered** — `test_register.py:98`, `asyncio.gather` on two registrations |
| Failed-login persistence | **Covered** — the `_record(commit=False)` boundary is asserted |
| Rate-limit behaviour | **Covered** — 15 assertions |
| Recovery-code reuse | **Covered** — spent even on success |
| Logout-everywhere | **Covered** |
| Fingerprint mismatch | **Covered** — 11 assertions |
| **Pending-handle race** | **Missing** — no test, and the bug in point 3 is why |
| **Mixed-case duplicate email** | **Missing** — every test address is lower-case |

The mixed-case gap is worth naming plainly: `CITEXT` is doing security work — it is what stops
`Me@anudeep.pro` and `me@anudeep.pro` becoming two accounts — and **nothing asserts it**. The
only occurrences of `citext` in the suite are `CREATE EXTENSION` in `conftest.py:82` and a
lower-case seed at `:130`. A port to a database layer without `citext` would pass the entire
suite.

Rate-limit races were also raised. The counter is derived per request from
`recent_failures()` rather than incremented, so concurrent failures cannot lose an update —
worst case two requests both observe the pre-trip count and both proceed, which permits one
extra attempt, not an unbounded number. Worth a test; not a defect.

---

## c) The accepted design

### c.a) One canonical contract

`/api/v1/auth/*` is removed. Everything is served under `/v1`, renamed for symmetry:

| Today | Becomes |
|---|---|
| `GET /api/v1/auth/me` | `GET /v1/me` |
| `POST /api/v1/auth/logout-all` | `POST /v1/logout/all` |
| `POST /api/v1/auth/password` | `POST /v1/password/change` |
| `POST /api/v1/auth/login/recovery` | `POST /v1/login/recovery` |

The HTML screens move under `/ui/v1/*`. Health probes stay unversioned — they belong to the
platform, and an uptime monitor should not need reconfiguring when the API version changes.

```
   /v1/…      the JSON contract          machine callers
   /ui/v1/…   the browser screens        people
   /healthz   /readyz                    the platform
```

**This is a breaking change**, and deliberately so: it is made once, against the Node
implementation, rather than twice.

#### Resolved: two surfaces, each with one content type

`/v1/*` returns **JSON only**. `/ui/v1/*` returns **HTML only** (or a `303` to another HTML
page). Neither surface renders the other's content type, and the UI handlers reach the same
`AuthService` methods the `/v1` handlers reach — one implementation, no HTTP self-call.

```
   ┌─ /v1/…      JSON in  → JSON out          machine callers
   │                                          never HTML, never a 303 for a form
   │
   ├─ /ui/v1/…   form in  → HTML or 303       people, with or without JavaScript
   │
   └─ both call ─▶ AuthService ──▶ Postgres · Redis
                   one contract, one transaction, one place a bug can live
```

This is the current Python architecture with the paths renamed, which makes it the lowest-risk
shape available: the behaviour being ported is the behaviour already under test.

**Where the page's JavaScript fits.** The form's `action` is its own `/ui/v1` path, so a
browser with scripting disabled posts there and gets HTML or a redirect. The progressive
enhancement then has a choice, and the earlier instruction — *the pages should call `/v1` APIs
only* — settles it: the script intercepts the submit and calls `/v1`, reading JSON.

```
   no JS   <form action="/ui/v1/login" method="post">  → 303 → /ui/v1/login/authenticator
   JS      fetch('/v1/login', {json})                  → 200 StartResponse → location.assign(…)
```

Both paths exercise the same service; only the rendering differs.

**A simplification falls out of this.** `BrowserStartResponse` and `BrowserTokenResponse` —
the `StartResponse`/`TokenResponse` subclasses that add `redirect`, added to the Python earlier
so the HTML surface could speak the `/v1` contract — have **no counterpart in the Node port**.
The UI surface no longer returns JSON at all, so there is no browser-shaped JSON body to keep
in step. Two schemas and their drift tests disappear.

The one cost: with `/v1` returning no `redirect` field, the script maps `status` to the next
screen itself — three entries (`totp_required`, `totp_enrolment_required`, `ok`). That is
routing knowledge in two places, which is the thing `redirect` existed to avoid. It is accepted
here because the alternative is putting a browser concern back into the JSON API, and three
entries in ~700 bytes of script is a smaller liability than a field that only one caller reads.

### c.b) Endpoint matrix

Paths shown post-rename. `CallerIdentity` reads `X-Pro-Client-Ip`/`X-Pro-User-Agent`, honoured
only from a caller holding a valid service credential.

| Method | Path | Auth | Request | Success | Failure | Cookies |
|---|---|---|---|---|---|---|
| POST | `/v1/register` | none (mode-gated) | `RegisterRequest{email,password}` | `200 StartResponse` | `403` closed · `409` conflict · `422` short | sets `pending` |
| GET | `/v1/register/totp` | enrol handle | — (`X-Pro-Pending-Handle` or cookie) | `200 EnrolResponse` | `401` | — |
| POST | `/v1/register/totp` | enrol handle | `TotpRequest{totp_code,pending_handle?}` | `200 TokenResponse` | `401` | sets `session`+`csrf`, clears `pending` |
| POST | `/v1/login` | none | `LoginRequest{email,password}` | `200 StartResponse` | `401` · `429`+`Retry-After` | sets `pending` (enrol leg only) |
| POST | `/v1/login/totp` | pending handle | `LoginTotpRequest{pending_handle,totp_code}` | `200 TokenResponse` | `401` | sets `session`+`csrf`, clears `pending` |
| POST | `/v1/login/recovery` | none | `RecoveryLoginRequest{email,password,recovery_code}` | `200 TokenResponse` | `401` · `429` | sets `session`+`csrf` |
| POST | `/v1/logout` | bearer or cookie | — | `204` | — (silent) | clears `session`+`csrf` |
| POST | `/v1/logout/all` | **CurrentUser** | — | `204` | `401` · `403` CSRF/origin | clears `session`+`csrf` |
| POST | `/v1/password/change` | **CurrentUser** | `PasswordChangeRequest{current_password,new_password}` | `204` | `401` · `403` · `422` | clears `session`+`csrf` |
| GET | `/v1/me` | **CurrentUser** | — | `200 MeResponse` | `401` | — |
| POST | `/v1/validate` | **Bearer + service credential** | — (all `X-Pro-*`) | `200 ValidateResponse` | `401` no bearer/credential · `429` per-service | — |
| GET | `/healthz` | none | — | `200` | — | — |
| GET | `/readyz` | none | — | `200` / `503` degraded | — | — |
| GET | `/ui/v1/login` · `/login/authenticator` · `/login/setup` · `/login/recovery` · `/register` | none (form token minted) | `?error=` `?email=` `?next=` | `200` HTML | `303` if the step is unreachable | sets form token |
| POST | the same five paths | form token + `Origin` | form-encoded | `303` to the next screen | `4xx` **re-rendered HTML** with the message | form token, `pending`, `session`+`csrf` |

The `/ui/v1` rows never return JSON and the `/v1` rows never return HTML. A failed form post
re-renders its own screen with the message inline, as it does today — no `?error=` round trip
is needed, because the UI handler owns the rendering.

Every failure body is RFC 9457 Problem Details:
`{type,title,status,detail,code,request_id}` plus `errors[]` of
`{code,field,message,line,hint}` on `422`. Every response carries `X-Request-ID`.

### c.c) Cookie contract

Three cookies, `__Host-` prefixed in production only — the prefix requires `Secure`, and a
`Secure` cookie over plain HTTP is silently dropped, so a prefixed name in development means the
browser stores nothing.

| Cookie | Set by | httpOnly | Max-Age | Purpose |
|---|---|---|---|---|
| `anudeep_session` | every issuance path | **yes** | idle TTL | the session token |
| `anudeep_csrf` | every issuance path | **no** | idle TTL | read by the SPA, echoed in `X-CSRF-Token` |
| `anudeep_totp_pending` | register, login (enrol leg) | yes | 600 / 300 | the second-leg handle |
| `anudeep_login_form` | every rendered page | yes | 900 | double-submit for unauthenticated posts |

All: `SameSite=Lax`, `Path=/`, **no `Domain`** — host-only, so no sibling subdomain can set or
overwrite them.

**The rule the schemas do not state:** every response that issues a session sets `session` *and*
`csrf`, **and** returns the same values in the body. Both deliveries, every time — a browser
uses the cookies and ignores the body; a script does the reverse.

### c.d) Parity fixtures

Deterministic seeds, identical in both services, so the diff is byte-level rather than shape-level:

```
fixtures/parity.json
  user:     id (fixed UUID), email, password, argon2 hash (fixed salt)
  totp:     secret (fixed base32) + codes at pinned timestamps
  recovery: ten codes and their hashes
  session:  token, sha256 digest, fingerprint inputs
```

argon2 hashes are portable — `@node-rs/argon2` emits the same
`$argon2id$v=19$m=65536,t=3,p=4$…` PHC string as `argon2-cffi`, verified during scaffolding —
so one fixture seeds both.

The differ drives an identical sequence against `:8001` and `:8000` and compares status, JSON
keys and values (excluding tokens and timestamps), `Set-Cookie` names and flags, and Problem
Details bodies. `auth_flow.py smoke` remains the acceptance gate; this is the microscope.

### c.e) Version policy

| | Pinned to |
|---|---|
| Node | **24.20.0** (Latest LTS "Krypton"), `engines` + `.nvmrc` + Docker base |
| Fastify · Zod · Drizzle · ioredis · @node-rs/argon2 · otpauth · OTel | exact, no `^` |

### c.f) Concurrency gates to add

| Gate | State |
|---|---|
| Concurrent `/v1/login/totp` with one handle → exactly one session | **new** — the point 3 bug |
| Concurrent recovery-code use → exactly one session | **new** — same shape, in Postgres |
| `Me@anudeep.pro` collides with `me@anudeep.pro` on register and on login | **new** |
| Concurrent registration under `single` → one account | port existing |
| Rate-limit under concurrent failures | **new**, low severity |
| Failed-login persistence · recovery reuse · logout-everywhere · fingerprint mismatch | port existing |

---

## Summary

| # | Point | Verdict |
|---|---|---|
| 1 | validate path inconsistency | premise wrong; real split resolved by moving everything to `/v1` |
| 2 | endpoint matrix | accepted — (c.b) |
| 3 | atomic pending consumption | **live vulnerability**, different cause than stated; no Lua needed |
| 4 | cookie parity | accepted — (c.c) |
| 5 | semantic telemetry | accepted; my wording was wrong |
| 6 | parity fixtures | accepted — (c.d) |
| 7 | pin versions, Node 24 | accepted; 24.20.0 already installed |
| 8 | Docker split | already implemented; document it |
| 9 | concurrency gates | 6 of 8 exist; 2 real gaps, both named |

**Decisions taken during review:** everything moves under `/v1` with the four renames; the
screens move to `/ui/v1/*`, keeping both GET and POST. `/v1` returns JSON only and `/ui/v1`
returns HTML only; both call one shared `AuthService`, so there is no HTTP self-call and no way
for the surfaces to drift. The UI works without JavaScript because its forms post to their own
`/ui/v1` path; where scripting is available it calls `/v1` instead. `BrowserStartResponse` and
`BrowserTokenResponse` are dropped — the UI surface returns no JSON, so they have nothing to
keep in step with. Nothing outstanding.
