# iam.anudeep.pro — the admin console API surface

Derived from three console mockups: **Registration policy**, **Active sessions**, **Users**.

## a) What am I trying to do

Work out what APIs the three screens need, and — more usefully — which of them the service
cannot currently answer. The endpoint list is the easy half. The screens quietly assume a data
model that does not exist yet and, in two places, assume the opposite of a decision the service
made deliberately. Naming those before anything is built is the point of this document.

Nothing here is implemented. This is the shopping list and the objections.

## b) Options

### Option 1 — `/v1/admin/*` inside this service

```
  browser ──▶ /ui/v1/*      sign-in screens        (exists)
           ──▶ /v1/*        identity API           (exists)
           ──▶ /v1/admin/*  console API            (new)
                    │
                    └─▶ requireUser + requireAdmin ─▶ AuthService + AdminService
```

**Pros.** One deployment, one session cookie, one service layer. The console reads exactly the
state the auth paths write, with no replication lag and no second source of truth. Reuses
`requireUser`, the CSRF gate, the fingerprint binding and the Problem Details handler as they
stand.

**Cons.** Grows the blast radius of the identity service: a bug in a console listing runs in the
process that mints credentials. Wants a hard authorisation boundary rather than a soft one.

### Option 2 — a separate admin service reading the same stores

```
  browser ──▶ admin.anudeep.pro ──▶ its own API ──▶ Postgres + Redis (shared)
                                          │
                                          └─▶ /v1/validate ──▶ iam.anudeep.pro
```

**Pros.** Console traffic cannot touch the credential-issuing process. Independently
deployable and scalable.

**Cons.** Two writers to one schema, which is how invariants rot — the registration lock, the
epoch bump and the session index all assume a single writer. Session revocation from outside
`SessionStore` would bypass the index maintenance it does. Needs its own auth, and the console
is small.

### Option 3 — extend the CLI only, no console API

**Pros.** Free; the CLI already lists users and clears lockouts.

**Cons.** Does not answer the mockups, which are a browser UI with bulk selection and live TTLs.

### Selected: **Option 1**, with a real admin role

The console is small and its whole value is reading live state. Splitting it costs a shared
schema with two writers, which is a worse problem than the blast radius. The blast radius is
addressed with an explicit role check rather than by a separate process.

## c) The design

### c.a) Authorisation — the prerequisite

Today every authenticated account is fully privileged: `requireUser` proves *who* you are and
nothing decides *what* you may do. The Users screen shows an account with status `service`,
which already implies at least two kinds of principal. So before any endpoint below:

```
  admin_user
    + role         'admin' | 'operator' | 'service'   NOT NULL DEFAULT 'admin'
    + locked_at    timestamptz NULL
```

```
  request ──▶ requireUser ──▶ requireAdmin ──▶ handler
                  │               │
                  │               └─ role = 'admin'?  no ──▶ 403 (audited)
                  └─ session valid, fingerprint matched, CSRF checked on unsafe methods
```

`requireAdmin` composes with `requireUser` rather than replacing it, so the console inherits
the cookie/bearer split and the CSRF rule already in `plugins/context.ts`.

### c.b) The endpoints

All under `/v1/admin`, all JSON, all `requireUser` + `requireAdmin`.

#### Screen 3 — Users

| Method | Path | Backs |
|---|---|---|
| `GET` | `/v1/admin/overview` | the three header tiles: accounts, TOTP enrolled, registration mode |
| `GET` | `/v1/admin/users` | the table. `?q=` (email **or hash prefix**), `?filter=all\|totp\|locked`, `?cursor=&limit=` |
| `GET` | `/v1/admin/users/:id` | User detail: hash key, TOTP enrolled date, fingerprint header count |
| `POST` | `/v1/admin/users/:id/sessions/revoke` | "Revoke all sessions" → `AuthService.logoutEverywhere` |
| `POST` | `/v1/admin/users/:id/lock` | the `Locked` filter and "Currently locked" tile imply a lock verb |
| `DELETE` | `/v1/admin/users/:id/lock` | unlock |
| `POST` | `/v1/admin/invitations` | "Invite user" |
| `GET` | `/v1/admin/invitations` | outstanding invites (implied — an invite you cannot see or revoke is a leak) |
| `DELETE` | `/v1/admin/invitations/:id` | revoke an unused invite |

Public counterpart, unauthenticated, needed for an invite to be usable:

| `POST` | `/v1/invitations/:token/accept` | redeem → account + enrolment handle |

The list row needs `sessions` (a count per user). `SessionStore.countFor` already does this, but
one `SCARD` per row is N+1 — pipeline them, or denormalise.

#### Screen 2 — Active sessions

| Method | Path | Backs |
|---|---|---|
| `GET` | `/v1/admin/sessions` | the table. `?stage=verified\|awaiting_totp\|rejected`, `?user=`, `?cursor=` |
| `DELETE` | `/v1/admin/sessions/:key` | the per-row "Revoke" |
| `POST` | `/v1/admin/sessions/revoke` | "Revoke selected" — bulk, `{ keys: [...] }` |
| `GET` | `/v1/admin/fingerprint/composition` | the `X-Pro-*` header list (static config, not state) |
| `GET` | `/v1/admin/metrics/funnel?window=24h` | login → TOTP funnel counts |

#### Screen 1 — Registration policy

| Method | Path | Backs |
|---|---|---|
| `GET` | `/v1/admin/policy` | all three panels in one read |
| `PUT` | `/v1/admin/policy/registration` | the only panel with a Save button |

The enrolment and abuse-control panels render as **read-only** in the mock — no control, no
Save. They are reported by `GET /v1/admin/policy` and remain environment-configured.

#### Audit

| `GET` | `/v1/admin/audit` | "Recent identity events". `?since=&user=&type=&cursor=` |

### c.c) Flow: saving the registration policy

The mock's own caption is the hard requirement — *"applies on next /v1/register call · no
restart"*:

```
  PUT /v1/admin/policy/registration {mode}
        │
        ├─ requireAdmin
        ├─ validate mode ∈ {open, single}
        ├─ UPDATE iam.setting SET value=… WHERE key='registration_mode'   ← durable
        ├─ audit('policy_changed', actor, from, to)
        └─ invalidate the cached policy
                    ▼
  POST /v1/register ──▶ policy.registrationMode()  ← reads the store, not process.env
```

This does not work today. `IAM_REGISTRATION_MODE` is an environment variable read through
`getSettings()`, which is memoised for the life of the process — so a saved policy could not
take effect without a restart, which is precisely what the caption forbids.

```
  iam.setting
    key    text primary key
    value  jsonb not null
    updated_at timestamptz
    updated_by uuid → admin_user(id)
```

Environment stays the **seed**: on first boot the row is created from `IAM_REGISTRATION_MODE`.
After that the row wins. One source of truth at runtime, with the env var as the bootstrap.

### c.d) Classes

```ts
// src/services/admin-service.ts — the console's service layer.
class AdminService {
  overview(): Promise<Overview>                              // 3 tiles
  listUsers(q: UserQuery): Promise<Page<UserRow>>
  getUser(id: string): Promise<UserDetail>
  lock(id: string, by: string): Promise<void>
  unlock(id: string, by: string): Promise<void>
  invite(email: string, by: string): Promise<Invitation>
  listSessions(q: SessionQuery): Promise<Page<SessionRow>>
  revokeSessions(keys: string[], by: string): Promise<number>  // returns how many actually died
  funnel(windowSeconds: number): Promise<Funnel>
}

// src/services/policy-service.ts — runtime-mutable settings.
class PolicyService {
  registrationMode(): Promise<'open' | 'single'>   // cached, invalidated on write
  setRegistrationMode(mode: string, by: string): Promise<void>
  snapshot(): Promise<PolicySnapshot>              // everything the policy screen reports
}

// src/repositories/audit-repo.ts
class AuditRepository {
  record(e: AuditEvent): Promise<void>
  recent(q: AuditQuery): Promise<Page<AuditEvent>>
}
```

New store support required:

```ts
// SessionStore — listing every session, without SCAN across a shared Redis.
sessionsAll(cursor): Promise<Page<SessionRow>>   // needs a global index; see conflict 4
// PendingAuthStore — the tmp: rows the sessions table shows
pendingAll(cursor): Promise<Page<PendingRow>>    // no index exists at all today
```

### c.e) What the screens assume that the service does not do

Ordered by how much work the disagreement implies. **1 and 2 are contradictions, not gaps** —
they ask for the opposite of a decision made on purpose.

1. **Sessions display readable IP and User-Agent per row.** The session store keeps *only* a
   sha256 fingerprint digest, deliberately: "whoever reads the session store learns nothing
   about where the account signs in from — the readable values live in `login_attempt`, where
   they are the audit trail rather than a lookup key." Rendering this column means either
   storing readable IP/agent in Redis beside every session (undoing that), or joining each
   session to its originating `login_attempt` row (no such link exists — nothing correlates a
   session digest to the attempt that produced it). A correlation id written to both is the
   cheap fix, and it keeps the digest-only property.

2. **"Offer TOTP opt-in at register" implies TOTP is optional.** The service makes a second
   factor mandatory, and the whole login flow is shaped around it: a correct password returns a
   handle, never a session, and an account without a secret is offered enrolment instead of
   being let in. An opt-in reintroduces the "declined a second factor" state that was
   deliberately removed.

3. **`closed` is shown as a registration mode.** It was removed on instruction earlier today —
   `IAM_REGISTRATION_MODE` is now `'single' | 'open'` and `closed` is refused at boot. The mock
   predates that. **Decision needed:** the mock or the instruction.

4. **Listing every session requires an index that does not exist.** `SessionStore` avoids
   `SCAN` on purpose ("SCAN across a shared Redis is exactly the operation to avoid") and keeps
   only a per-user set. Pending handles have no index whatsoever. A global sorted set keyed by
   expiry, maintained on create/revoke, is the shape that fits.

5. **`rejected` and `expired` rows are shown.** Both are deleted today; only a 15-minute
   `pending:spent:` marker survives a rejection, and it carries no user, IP or agent. Showing
   them needs those outcomes retained as records.

6. **No audit event log exists.** `login_attempt` records sign-in attempts only. The audit panel
   wants `temp token issued`, `fingerprint mismatch rejected`, `registration blocked
   mode=single` — a general event table.

7. **No `role`, `locked_at`, invitations, or service accounts** in the schema. Screens show all
   four.

8. **The fingerprint composes 6 headers in the mock, 2 inputs in the code.** Today it is the IP
   network (/24 or /64) plus User-Agent. The mock adds `X-Pro-Device-Id`, `X-Pro-Tz`,
   `X-Pro-Client` and `X-Pro-Email`. Adding inputs changes what every stored digest means and
   **invalidates every live session** on deploy. Also worth noting: a fingerprint is a *second
   condition*, not a secret — adding low-entropy inputs adds no strength, only more ways for a
   legitimate user to be logged out.

9. **Stated limits disagree with the implementation.** None is load-bearing for the API list,
   but the screen is a settings display and it should not lie:

| Screen says | Code does |
|---|---|
| Temp token TTL 120s | `PENDING_TTL_SECONDS = 300` |
| Session key TTL 12h sliding | 1800s idle + 14d absolute ceiling |
| Failed TOTP 3 → new login | `MAX_ATTEMPTS = 5` |
| `/v1/login` 10/min per IP | 20 per 900s window |
| Fingerprint mismatch → reject + audit | reject + audit **+ revoke the session** |

10. **The funnel panel may not need an API.** Those four numbers are already counters
    (`iam.login.attempts`, `iam.totp.verifications`, `iam.session.resolved{fingerprint_mismatch}`,
    `iam.pending.handle`) exported to Prometheus. Querying Prometheus avoids a bespoke
    aggregation endpoint and a second definition of each number.

11. **Mock inconsistency, minor:** the Users screen shows 3 accounts under `registration mode:
    single`, while the policy screen defines single as "exactly one account may ever be
    created". Consistent with the implementation (single blocks only while an account already
    exists; the CLI can still create more) but the copy overstates it.

### c.f) Suggested order

1. `role` + `locked_at` + `requireAdmin` — nothing else is safe to expose first.
2. `iam.setting` + `PolicyService` — unblocks the one screen with a Save button.
3. Read-only endpoints: overview, users list/detail, audit, policy snapshot.
4. Session listing — needs the global index (conflict 4) and the IP/agent decision (conflict 1).
5. Mutations: lock/unlock, revoke, bulk revoke.
6. Invitations, both halves.
