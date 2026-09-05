# api.anudeep.pro — Delegate Identity to IAM, Expose GraphQL, Build Markdown Ingest

**Date:** 2026-09-04
**Project:** `api.anudeep.pro` (admin-api)
**Related:** `iam.anudeep.pro`, `admin.anudeep.pro`, `resume.anudeep.pro`
**Source requirement:** `/Users/anudeepchandrapaul/Projects/github.com/AnudeepChPaul/requirements.md`
**Supersedes (in part):** `markdown_plans/resume.anudeep.pro/3d-resume-site-react-admin-fastapi-backend-2026-09-03.md`

---

## a) What am I trying to do

`requirements.md` describes four moving parts — a central identity service, an admin portal, a
résumé site, and the backend behind the admin. This repo is that backend. Three facts are in
conflict with what it currently contains.

**1. Identity has moved out, but the code has not.**
`iam.anudeep.pro` already exists and is built: registration gated by `IAM_REGISTRATION_MODE`,
per-user TOTP secrets, opaque 256-bit fingerprint-bound sessions in Redis, `POST /api/v1/validate`
for introspection, and a one-time handoff. `api.anudeep.pro` was the repo IAM was forked *from*,
and it still carries the entire duplicate:

| Still in `api.anudeep.pro` | Owned by `iam.anudeep.pro` |
|---|---|
| `admin_user.password_hash`, `.totp_secret` | yes |
| `refresh_token`, `recovery_code`, `login_attempt` tables | yes |
| `routers/login.py` — 569 lines, server-rendered 3-step login page | yes |
| `routers/auth.py`, `services/auth.py` — 773 lines | yes |
| `services/pending.py`, `security/passwords.py`, `security/csrf.py`, `qr.py`, `templates.py` | yes |
| `scripts/auth_flow.py`, `scripts/totp_qr.py` | yes |
| `tests/integration/test_auth.py` + `test_login_page.py` — 988 lines | yes |

Roughly **2,900 lines** testing and implementing a second, independent way to become an admin.
Two services minting sessions for the same person is precisely what "central identity service"
removes. The requirement is explicit: *"other services will call identity to validate them."*

**2. The admin backend must speak GraphQL.**
`requirements.md`: *"the backend for this must support graphql and for file upload it can work
with REST API(s)."* The 2026-09-03 design doc reached the opposite conclusion (line 32: Astro
"removes the need for the GraphQL layer that was under consideration"). That decision is
reversed by the requirement, and the file-upload carve-out is retained because markdown files
and assets arrive as multipart — a thing GraphQL does badly.

**3. The content half — the reason this service exists — is unbuilt.**
Steps 1–4 of the original sequence landed: settings, RFC 9457 errors, security headers, health
probes, the complete SQLAlchemy data model, migration `0001_initial`, the front-matter contracts
(`schemas/frontmatter.py`) and the markdown renderer (`services/markdown.py`). Not written:
ingest, content mutations, relation resolution, publish, bundle, deploy, assets.

### Target end-state

```
                    ┌──────────────────────────────────────────────┐
                    │  iam.anudeep.pro          :8001               │
   sign in ────────▶│  users · TOTP · sessions (opaque, in Redis)   │
                    │  POST /api/v1/validate  ◀──── token+fingerprint
                    └────────────────────▲─────────────────────────┘
                                         │ service token (token:validate)
                                         │
 admin.anudeep.pro                       │
 (React SPA)  ──── POST /graphql ────────┴──▶ api.anudeep.pro       :8000
              ──── POST /api/v1/ingest (multipart) ──▶  markdown → validate →
                                                        render → ContentVersion
                                                             │
 resume.anudeep.pro  ◀── Astro build ── GET /api/v1/site-bundle (build token)
```

`api.anudeep.pro` ends up authenticating **nobody**. It asks IAM, serves the admin over GraphQL,
and takes markdown over multipart REST.

### Scope of this plan

In: identity delegation, GraphQL admin surface, markdown ingest pipeline.
Out (a later pass, deliberately): publish/bundle/rollback, `GET /site-bundle`, assets → R2, the
Cloudflare deploy hook.

---

## b) What are my options

Three independent decisions. Each is presented with its alternatives.

### b.1 — How does `api` authenticate an admin?

#### Option A — Handoff, then `api` mints its own session

IAM redirects a completed sign-in to `api/session?code=…`; `api` exchanges the one-time code at
`POST /api/v1/handoff/exchange`, then sets its **own** host-only `HttpOnly` cookie. IAM leaves
the request path entirely.

```
browser ─▶ iam/login                    password + TOTP
        ◀─ 302 api/session?code=anup_…  60s, single use, addressed to `api`
        ─▶ api/session
              api ─▶ iam POST /api/v1/handoff/exchange   X-Pro-Code: anup_…
                  ◀─ {sub, email, totp_enrolled, epoch}
        ◀─ Set-Cookie: __Host-anudeep_session          (api's own, opaque, in Redis)
        ─▶ api/graphql                                  IAM not involved
```

| Pros | Cons |
|---|---|
| Session token never reaches JavaScript — `HttpOnly` survives an XSS in the markdown preview, the app's riskiest surface | `api` runs a second session store: mint, fingerprint, idle TTL, absolute cap, revoke — a near-copy of `iam/services/session.py` |
| IAM is off the hot path; one outbound call per sign-in, not per request | A revoke at IAM does not reach `api` until it re-checks; needs an epoch poll or a webhook |
| Survives an IAM outage for already-signed-in admins | CSRF comes back, because a cookie is ambient authority |
| This is the flow IAM's README already documents | Two session lifetimes to reason about and keep aligned |

#### Option B — Stateless: validate the IAM token on every request ✅ **SELECTED**

`api` holds no session. The admin SPA presents IAM's token as `X-Pro-Token`; `api` introspects it
at IAM, with a short Redis cache.

```
admin SPA ─▶ api/graphql
               X-Pro-Token:     anut_…              (the end user's IAM token)
               ↓
             api ─▶ iam POST /api/v1/validate
                      Authorization:    Bearer <api's service token>
                      X-Pro-Token:      anut_…
                      X-Pro-Client-Ip:  <END USER's ip>       ← not ours
                      X-Pro-User-Agent: <END USER's agent>    ← not ours
                  ◀─ {"active": true, sub, email, epoch, expires_at}
               ↓ cached ≤30s, keyed by token **and** fingerprint
             resolve Identity
```

| Pros | Cons |
|---|---|
| One source of truth for "is this session live" — a logout-everywhere at IAM is honoured within the cache TTL, everywhere | The token must live in the SPA's memory; an XSS in the admin can lift it |
| No session store, no cookie, no CSRF, no handoff receiver in `api` | IAM is on the hot path — if IAM is down, the admin is down (fail closed) |
| Fingerprint binding is enforced by IAM itself, unmodified, on every request | A cache round-trip on every cold request adds latency to each admin call |
| Deletes ~2,900 lines rather than replacing them with ~800 | Cache key must include the fingerprint or the binding is silently defeated |
| Matches the requirement's own words: *"other services will call identity to validate them"* | |

**Selected: Option B.** It is what `requirements.md` literally describes, and the XSS exposure is
bounded — the token is IAM's own, fingerprint-bound to one network prefix and one user agent, so
a lifted token is not portable to the attacker's machine. Option A's `HttpOnly` advantage buys
less than it costs once IAM already refuses replays from elsewhere.

#### Option C — Verify a JWT locally, no network call

Rejected outright: IAM's token is **opaque** by design (`anut_…`, 256 random bits, meaningless
without the session store). There is nothing to verify locally. IAM's README is explicit that
this was chosen over a JWT so that expiry can slide and revocation can be immediate.

---

### b.2 — What surface does `api` expose to the admin?

#### Option A — GraphQL is the whole admin API ✅ **SELECTED**

```
POST /graphql                    ← the entire admin surface (Strawberry)
  Query:    items · item(slug) · versions · ingestBatch · unresolvedRelations
            bundles · deployments
  Mutation: saveDraft · archiveItem · restoreItem
            (publish · rollback · retryDeploy — stage 4)

REST, only where GraphQL is the wrong tool:
  POST /api/v1/ingest?dry_run=   multipart: files[] | archive.zip
  POST /api/v1/assets            multipart
  GET  /api/v1/ingest/export     zip of current drafts
  GET  /api/v1/site-bundle       build token, ETag          (stage 4)
  GET  /healthz  /readyz
```

| Pros | Cons |
|---|---|
| One contract, one test suite, nothing to keep in step | Strawberry + a dataloader layer is new machinery in this repo |
| The admin fetches exactly the fields a screen needs — the résumé graph (experience → project → skill) is the shape GraphQL is actually good at | N+1 is a real risk on that same graph; needs dataloaders from day one |
| Schema doubles as the admin's typed client via codegen | GraphQL's error channel and this repo's RFC 9457 Problem Details must be reconciled |
| Satisfies the requirement directly | |

#### Option B — Full REST admin endpoints, with GraphQL as a second facade

Build `/api/v1/content`, `/publish`, `/bundles`, `/deployments` from the 2026-09-03 doc, then add
GraphQL over the same services.

| Pros | Cons |
|---|---|
| REST endpoints are trivially curl-able and cache-able | Two contracts over one service layer — the drift this repo avoids everywhere else |
| A non-SPA consumer could use REST | Two test suites for one behaviour |
| | No second consumer exists; the résumé site reads the bundle, not the admin API |

**Selected: Option A.** Nothing but the admin SPA calls this surface, so a second contract has no
consumer to justify it.

---

### b.3 — In what order?

#### Option A — Identity → GraphQL → ingest, then pause ✅ **SELECTED**

| Pros | Cons |
|---|---|
| Identity lands and is **verified against a live IAM** before anything is built on top of it | Nothing user-visible until stage 2 |
| Each stage ends `make check`-green and committed | |
| The bundle payload — the contract Astro snapshot-tests — gets reviewed before it sets | |

#### Option B — All five stages in one pass

| Pros | Cons |
|---|---|
| One continuous context, no re-orientation | The bundle payload shape sets without review |
| | A mistake in the identity layer propagates through four stages before anyone looks |

**Selected: Option A**, with the added instruction that **token + fingerprint validation against
IAM is proven first**, before stages 2 and 3 begin.

---

## c) The selected design

### c.1 System design — component view

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ admin.anudeep.pro   (React SPA, stage-2 consumer)                             │
│   holds the IAM token in memory · sends it as X-Pro-Token on every call       │
└───────┬────────────────────────────────────────┬──────────────────────────────┘
        │ POST /graphql                          │ POST /api/v1/ingest (multipart)
        │ X-Pro-Token: anut_…                    │ X-Pro-Token: anut_…
╔═══════▼════════════════════════════════════════▼══════════════════════════════╗
║ api.anudeep.pro                                                    :8000      ║
║                                                                               ║
║  middleware   OriginLock → TrustedHost → RequestContext → CORS → SecHeaders   ║
║                                                                               ║
║  deps.current_identity ──────────────────────────────┐                        ║
║        presented_token(request)   X-Pro-Token         │                        ║
║        client_ip(request)         CF-Connecting-IP    │                        ║
║        request.headers['user-agent']                  │                        ║
║                                                       ▼                        ║
║                                            services/iam.py  IamClient          ║
║                                              ├─ Redis cache  (30s / 5s neg)   ║
║                                              └─ httpx.AsyncClient ────────────╫──▶ iam :8001
║                                                       │                        ║    POST /api/v1/validate
║                                            Identity(sub,email,epoch,expires)  ║    ◀── {"active":…}
║                                                       │                        ║
║                                     AdminUserRepository.upsert_mirror()        ║
║                                          (only on cache miss)                  ║
║                                                       │                        ║
║  ┌────────────────────────────────────────────────────▼─────────────────────┐ ║
║  │ graphql/schema.py          routers/ingest.py                             │ ║
║  │   Query · Mutation           multipart → files[] | zip                   │ ║
║  └───────────────┬──────────────────────┬───────────────────────────────────┘ ║
║                  │                      │                                     ║
║        ┌─────────▼──────────────────────▼──────────┐                          ║
║        │ services/content.py   ContentService      │  ◀── the ONE path        ║
║        │   parse → validate → render → hash → save │      "markdown → version"║
║        └────┬───────────────┬──────────────┬───────┘                          ║
║             │               │              │                                  ║
║   schemas/frontmatter  services/markdown  services/relations                   ║
║   (extra="forbid")     (markdown-it+nh3)  RelationResolver                     ║
║             │               │              │                                  ║
║  ┌──────────▼───────────────▼──────────────▼──────────────────────────────┐   ║
║  │ Postgres :5432   content_item · content_version · content_relation      │   ║
║  │                  ingest_batch · ingest_file · asset · audit_log         │   ║
║  │                  bundle_version · site_state · deployment               │   ║
║  │                  admin_user  ← MIRROR ONLY (id = IAM's sub)             │   ║
║  └────────────────────────────────────────────────────────────────────────┘   ║
║  Redis :6379        iam:tok:{token_digest}:{fp_digest}  → cached Identity      ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

**Why `admin_user` survives at all.** Four foreign keys point at it —
`content_version.created_by`, `bundle_version.created_by`, `ingest_batch.created_by`,
`audit_log.actor_id`. Dropping the table means dropping four FKs and losing referential integrity
on the audit trail. Instead it shrinks to a mirror keyed by IAM's `sub`, holding no credential.

### c.2 The `/validate` contract — build from the code, not the README

`iam.anudeep.pro/README.md` documents a JSON body:

```sh
curl -X POST .../api/v1/validate -d '{"token":"…"}'        # ← STALE. Does not work.
```

The code (`iam/routers/validate.py`, after commit `5ec8160` "Move service-to-service inputs into
X-Pro headers") takes **headers and no body**. `iam/schemas/validate.py` is authoritative:

| Header | Meaning |
|---|---|
| `Authorization: Bearer …` | `api`'s own service credential, scope `token:validate` |
| `X-Pro-Token` | the **end user's** IAM session token — required |
| `X-Pro-Client-Ip` | the **end user's** address — *not the calling service's* |
| `X-Pro-User-Agent` | the **end user's** agent |
| `X-Pro-Token-Issued-At` | optional; pins one session. Unparseable values ignored, not refused |

Always `200`. `{"active": false}` covers expired, forged, revoked, unknown **and fingerprint
mismatch**, deliberately indistinguishable.

The fingerprint is computed inside IAM (`iam/security/fingerprint.py`) as
`sha256(ip_network ‖ NUL ‖ user_agent)`, with IPv4 narrowed to `/24` and IPv6 to `/64`. Our
entire responsibility is to forward the end user's values faithfully. Forwarding our own makes
every session look identical and every answer come back `active: false` — a failure that reads
like "bad token", which is why it gets an explicit test.

### c.3 Flow diagram — an authenticated GraphQL request

```
 SPA          api/deps           IamClient        Redis         iam:8001      Postgres
  │              │                   │              │               │            │
  ├─ POST /graphql                   │              │               │            │
  │  X-Pro-Token: anut_…             │              │               │            │
  │              │                   │              │               │            │
  │              ├─ presented_token()│              │               │            │
  │              │   X-Pro-Token, else Authorization: Bearer        │            │
  │              ├─ client_ip()      │              │               │            │
  │              │   CF-Connecting-IP ▸ else request.client.host    │            │
  │              ├─ user_agent       │              │               │            │
  │              │                   │              │               │            │
  │              ├─ validate(tok, ip, ua) ─────────▶│               │            │
  │              │                   ├─ GET iam:tok:{sha256(tok)}:{sha256(fp)} ──▶│
  │              │                   │◀─ HIT ───────┤               │            │
  │              │◀─ Identity ───────┤              │               │            │
  │              │                   │   ═══ no IAM call, no DB write ═══        │
  │              │                   │              │               │            │
  │              │                   │  ── MISS ──▶ │               │            │
  │              │                   ├─ POST /api/v1/validate ─────▶│            │
  │              │                   │   Authorization: Bearer <svc>│            │
  │              │                   │   X-Pro-Token / -Client-Ip / -User-Agent  │
  │              │                   │              │   ┌───────────┤            │
  │              │                   │              │   │ session lookup +       │
  │              │                   │              │   │ fingerprint compare    │
  │              │                   │              │   └───────────┤            │
  │              │                   │◀─ 200 {"active":true,sub,email,epoch,exp} │
  │              │                   ├─ SETEX min(30s, exp-now) ───▶│            │
  │              │                   ├─ upsert_mirror(sub,email,epoch) ─────────▶│
  │              │◀─ Identity ───────┤              │               │            │
  │              │                   │              │               │            │
  │       ┌──────▼───────┐           │              │               │            │
  │       │ IsAdmin ✓    │           │              │               │            │
  │       │ resolver     ├──────────────────────────────────────────────────────▶│
  │◀──────┤ data         │           │              │               │            │
                                                                                  
 Refusal paths, all → 401 AuthError, reason never disclosed:
   no X-Pro-Token            ·  {"active": false}  ·  IAM unreachable / timeout
 IAM unreachable additionally marks /readyz degraded. FAIL CLOSED — an identity
 service being down must never mean everyone is an admin.
```

### c.4 Flow diagram — markdown ingest (stage 3)

```
 SPA                routers/ingest       IngestService     ContentService      DB
  │                       │                    │                 │             │
  ├─ POST /ingest?dry_run=1 ─────────────────▶ │                 │             │
  │  multipart: files[] | archive.zip          │                 │             │
  │                       ├─ current_identity ─┤                 │             │
  │                       │                    ├─ INSERT ingest_batch(dry_run) ▶│
  │                       │                    │                 │             │
  │                       │              ┌─────┴──── unzip, guarded ─────┐     │
  │                       │              │  entries ≤ max_archive_entries│     │
  │                       │              │  each ≤ max_upload_bytes      │     │
  │                       │              │  no "..", no absolute paths   │     │
  │                       │              │  no symlinks                  │     │
  │                       │              │  decompression ratio capped   │     │
  │                       │              └─────┬─────────────────────────┘     │
  │                       │                    │                               │
  │                       │            per file (atomic per file, not batch):  │
  │                       │                    ├─ upsert_draft() ─▶│           │
  │                       │                    │   frontmatter.parse           │
  │                       │                    │   <Type>FrontMatter validate  │
  │                       │                    │       extra="forbid"          │
  │                       │                    │   MarkdownRenderer.render()   │
  │                       │                    │       markdown-it → nh3       │
  │                       │                    │   sha256(raw, CRLF-normalised)│
  │                       │                    │        │                      │
  │                       │                    │   hash unchanged? ──▶ skipped_unchanged
  │                       │                    │        │ no                   │
  │                       │                    │   dry_run? ──▶ report only, no write
  │                       │                    │        │ no                   │
  │                       │                    │   INSERT content_version (rev = max+1) ▶│
  │                       │                    │   UPDATE content_item.draft_version_id ▶│
  │                       │                    │   RelationResolver: slugs → item ids   ▶│
  │                       │                    │        unresolved ⇒ warning, recorded   │
  │                       │                    ├─ INSERT ingest_file(status, errors[], warnings[]) ▶│
  │                       │                    │                               │
  │                       │                    ├─ UPDATE batch: ok | partial | failed ▶│
  │◀─ IngestReport ───────┤                    │                               │
  │  {batch_id, files:[{filename, status, type, slug, errors[], warnings[]}], unresolved[]}
  │
  ├─ operator reads the report, confirms
  ├─ POST /ingest            (dry_run absent) ── same path, writes for real
```

### c.5 Classes and subs

#### `src/anudeep/api/services/iam.py` — NEW

```python
@dataclass(frozen=True, slots=True)
class Identity:
    """Who IAM says the caller is. Cached; never persisted beyond the mirror row."""
    sub: uuid.UUID
    email: str
    epoch: int
    expires_at: datetime | None

class IamUnavailable(ServiceError):
    """IAM did not answer. Distinct from 'IAM said no' — the first is a 503 condition
    for /readyz, the second is a 401 for the caller. Both refuse the request."""

class IamClient:
    """The only thing in this service that knows how to authenticate a human."""

    def __init__(self, http: httpx.AsyncClient, redis: Redis, settings: Settings) -> None: ...

    async def validate(self, token: str, *, ip: str, user_agent: str | None) -> Identity | None:
        """None ⇒ not active. Raises IamUnavailable if IAM could not be reached."""

    @staticmethod
    def _fingerprint_digest(ip: str, user_agent: str | None) -> str:
        """Mirrors iam/security/fingerprint.compute — IPv4 → /24, IPv6 → /64, NUL-joined.

        Not for authentication (IAM does that); this is a CACHE KEY component. Keyed on the
        token alone, a token replayed from another network would hit the cache entry the
        legitimate holder created and sail straight past the binding IAM exists to enforce.
        Narrowing to the same prefix IAM uses means a phone moving inside its /24 still hits
        cache instead of stampeding IAM.
        """

    def _key(self, token: str, ip: str, user_agent: str | None) -> str:
        """iam:tok:{sha256(token)}:{fingerprint_digest}"""

    async def _cached(self, key: str) -> Identity | None | _Miss: ...
    async def _store(self, key: str, identity: Identity | None) -> None:
        """Positive: TTL = min(settings.iam_cache_seconds, expires_at - now) — a revoke is
        visible within 30s. Negative: TTL 5s — short, but not caching negatives at all turns
        any invalid token into an unthrottled amplifier pointed at IAM."""

    async def probe(self) -> bool:
        """For /readyz. Cheap reachability check, no token involved."""
```

#### `src/anudeep/api/deps.py` — REWRITTEN

```python
SettingsDep  = Annotated[Settings, Depends(get_settings)]
SessionDep   = Annotated[AsyncSession, Depends(get_session)]
IamClientDep = Annotated[IamClient, Depends(get_iam_client)]

def get_iam_client(settings: SettingsDep) -> IamClient: ...

def presented_token(request: Request) -> str | None:
    """X-Pro-Token first — the family convention, and it keeps the token out of URLs and
    access logs. Authorization: Bearer accepted as a fallback for curl and CI."""

def client_ip(request: Request) -> str:
    """UNCHANGED. Already prefers CF-Connecting-IP, which Cloudflare sets itself and a
    client cannot spoof once the origin only accepts proxied traffic. This is precisely
    the value that must be forwarded to IAM."""

async def current_identity(
    request: Request, iam: IamClientDep, session: SessionDep
) -> Identity:
    """Resolve the caller, and keep the mirror row alive.

    upsert_mirror runs only on a cache miss, so this is not a database write per request.
    """

CurrentIdentity = Annotated[Identity, Depends(current_identity)]

def require_build_token(request, settings) -> BuildTokenConfig:   # UNCHANGED
def require_preview_token(request, settings) -> BuildTokenConfig:  # UNCHANGED
# DELETED: current_user, CurrentUser, get_auth_service, AuthServiceDep, bearer_token
# DELETED: the CSRF branch — there is no cookie left to defend. A token in a header is not
#          ambient authority; a hostile page cannot make a browser attach it.
```

#### `src/anudeep/api/models/user.py` — REDUCED TO A MIRROR

```python
class AdminUser(Base):
    """A mirror of an IAM identity, existing only so four FKs have something to point at.
    Holds no credential: no password hash, no TOTP secret, nothing that can sign anyone in."""
    __tablename__ = "admin_user"

    id: Mapped[uuid.UUID]          # = IAM's `sub`. No server_default — IAM assigns it.
    email: Mapped[str]             # CITEXT
    epoch: Mapped[int]             # renamed from token_epoch; last epoch IAM reported
    last_seen_at: Mapped[timestamp_opt]
    created_at: Mapped[created_at]

# DELETED: RefreshToken, RecoveryCode, LoginAttempt
```

#### `src/anudeep/api/repositories/user_repo.py` — REDUCED

```python
class AdminUserRepository:
    async def by_id(self, user_id: uuid.UUID) -> AdminUser | None: ...
    async def upsert_mirror(self, *, sub: uuid.UUID, email: str, epoch: int) -> AdminUser:
        """INSERT … ON CONFLICT (id) DO UPDATE SET email, epoch, last_seen_at."""

# DELETED: RefreshTokenRepository, RecoveryCodeRepository, LoginAttemptRepository
```

#### `migrations/versions/0003_delegate_identity.py` — NEW

```python
def upgrade():
    op.drop_table("refresh_token")          # order matters: FKs to admin_user
    op.drop_table("recovery_code")
    op.drop_table("login_attempt")
    op.drop_column("admin_user", "password_hash")
    op.drop_column("admin_user", "totp_secret")
    op.drop_column("admin_user", "last_login_at")
    op.alter_column("admin_user", "token_epoch", new_column_name="epoch")
    op.add_column("admin_user", sa.Column("last_seen_at", sa.DateTime(timezone=True)))
    op.alter_column("admin_user", "id", server_default=None)   # IAM assigns the id now

def downgrade():
    """Recreates the tables and columns as nullable/defaulted. The credentials themselves
    are gone for good — which is the point of the migration, not an oversight."""
```

#### `src/anudeep/api/graphql/` — NEW (stage 2)

```python
# context.py
@dataclass
class GraphQLContext:
    session: AsyncSession
    identity: Identity
    content: ContentService
    loaders: DataLoaders          # relations, versions — the experience→project→skill
                                  # graph produces an N+1 on the very first admin screen

# permission.py
class IsAdmin(strawberry.BasePermission):
    """A FIELD permission, not middleware. Keeps the refusal attached to the field that
    needed it, and lets a public field coexist later without a second mount point."""

# types.py
@strawberry.type class ContentItemType:     # slug, type, visible, draft, published, archivedAt
@strawberry.type class ContentVersionType:  # revision, title, bodyHtml, frontMatter, relations
@strawberry.type class RelationType:        # kind, targetType, targetSlug, resolved, ordinal
@strawberry.type class IngestBatchType:     # status, counts, files[]
@strawberry.type class IngestErrorType:     # projects schemas/ingest.IngestError verbatim
@strawberry.type class BundleVersionType:   # stage 4
@strawberry.type class DeploymentType:      # stage 4

# payloads.py
@strawberry.type class SaveDraftPayload:
    item: ContentItemType | None
    errors: list[IngestErrorType]
    """Content failures are DATA, not GraphQL errors. One renderer in the admin then serves
    inline edits and bulk ingest alike — which is the reason schemas/ingest.py's error
    vocabulary exists. Genuine faults (unauthorised, not found) stay GraphQL errors,
    mapped through errors.py."""

# schema.py
@strawberry.type
class Query:
    items(type, include_archived) · item(type, slug) · versions(item_id)
    ingest_batch(id) · unresolved_relations · bundles · deployments

@strawberry.type
class Mutation:
    save_draft(input: SaveDraftInput) -> SaveDraftPayload
    archive_item(input) · restore_item(input)
    # stage 4: publish · rollback · retry_deploy
```

Mounted at `POST /graphql`; GraphiQL enabled only when `not settings.is_prod` — the schema is a
complete inventory of the admin surface, the same reason `/docs` is already unmounted in prod.

#### `src/anudeep/api/services/content.py` — NEW (stage 3), the single write path

```python
class ContentService:
    """The ONE path from markdown text to a ContentVersion. saveDraft and file ingest both
    come through here — two paths would drift, and the front-matter rules are the product."""

    async def upsert_draft(
        self, *, actor: Identity, raw_markdown: str, source_filename: str | None = None,
        ingest_file_id: uuid.UUID | None = None, dry_run: bool = False,
    ) -> DraftResult:
        """parse front matter → validate against the type's model (extra='forbid')
        → MarkdownRenderer.render() → content_hash(raw, CRLF-normalised)
        → unchanged ⇒ SKIPPED_UNCHANGED, no new revision
        → INSERT content_version(revision = max+1) → repoint content_item.draft_version_id
        → RelationResolver rewrites content_relation from the authored slug lists"""

    async def archive(self, *, actor: Identity, type_: ContentType, slug: str) -> None: ...
    async def restore(self, *, actor: Identity, type_: ContentType, slug: str) -> None: ...
```

#### `src/anudeep/api/services/relations.py` — NEW (stage 3)

```python
class RelationResolver:
    async def resolve(self, version: ContentVersion, front_matter: BaseFrontMatter) -> list[UnresolvedRelation]:
        """Slugs → item ids. target_item_id IS NULL means the slug does not resolve yet:
        a warning at ingest, a blocker at publish, silently dropped from a bundle."""
```

#### `src/anudeep/api/services/ingest.py` + `routers/ingest.py` — NEW (stage 3)

```python
class ArchiveGuard:
    """The only real attack surface in the pipeline, so it is explicit rather than implied."""
    def entries(self, data: bytes) -> Iterator[ArchiveEntry]:
        # entry count ≤ settings.max_archive_entries
        # each uncompressed size ≤ settings.max_upload_bytes
        # reject "..", absolute paths, and symlinks
        # reject on decompression ratio BEFORE reading, not after

class IngestService:
    async def ingest_upload(
        self, *, actor: Identity, uploads: list[UploadFile], dry_run: bool
    ) -> IngestReport:
        """Per-FILE atomicity, per the existing ingest_file model: one bad file reports
        itself and the rest still land. batch.status = ok | partial | failed."""

    async def report(self, batch_id: uuid.UUID) -> IngestReport: ...

# routers/ingest.py
POST /api/v1/ingest?dry_run=      multipart: files[] | archive.zip   → IngestReport
GET  /api/v1/ingest/{batch_id}                                       → IngestReport
```

#### `src/anudeep/api/settings.py` — CHANGED

```python
# REMOVED: session_secret, session_ttl_seconds, refresh_ttl_seconds,
#          login_max_attempts, login_max_attempts_per_ip, login_window_seconds,
#          login_redirect_to
# ADDED:
iam_base_url: AnyHttpUrl                 # ANUDEEP_IAM_BASE_URL=http://127.0.0.1:8001
iam_service_token: SecretStr             # ANUDEEP_IAM_SERVICE_TOKEN — scope token:validate
iam_cache_seconds: int = 30
iam_negative_cache_seconds: int = 5
iam_timeout_seconds: float = 3.0
```

Mirrored into `.env.example` and `.env.production.example`. Minted at IAM with
`python -m anudeep.iam.cli mint-service-token --name api.anudeep.pro`.

#### `src/anudeep/api/main.py` — CHANGED

```python
# routers: health, graphql (stage 2), ingest (stage 3), docs (dev only)
#          auth and login are GONE
# CORS:    allow_headers = ["Content-Type", "X-Pro-Token"]      (X-CSRF-Token gone)
#          allow_credentials = False   ← no cookie to send; leaving it true widens the
#                                        policy for nothing
# lifespan: also close the shared httpx.AsyncClient, beside the engine and Redis
```

#### Deleted in full

`routers/auth.py` · `routers/login.py` · `templates.py` · `qr.py` · `services/auth.py` ·
`services/pending.py` · `security/passwords.py` · `security/csrf.py` · `scripts/auth_flow.py` ·
`scripts/totp_qr.py` · `tests/integration/test_auth.py` · `tests/integration/test_login_page.py`

Reduced: `security/tokens.py` keeps `mint_build_token`, `verify_build_token`,
`BUILD_TOKEN_PREFIX` — the Astro build's credential is **ours**, not IAM's, and IAM's README is
explicit that it issues no build tokens. `cli.py` keeps `mint-build-token`, loses `create-user`
and `clear-lockout`. The `Makefile` loses its six `auth-*` targets.

### c.6 Test plan

**Stage 1 is a gate.** IAM is stubbed at the transport boundary (`httpx.MockTransport`), so the
assertions are about the request we *send* as much as the answer we accept —
`tests/integration/test_identity.py`:

| Case | Expected |
|---|---|
| valid token | resolved; mirror row upserted |
| `{"active": false}` | 401; no reason disclosed |
| **outbound headers** | `X-Pro-Client-Ip` / `X-Pro-User-Agent` carry the **end user's** values, not the API's |
| **same token, different fingerprint** | a *second* call to IAM — not a cache hit. The regression that would silently defeat the binding |
| repeat inside TTL, identical inputs | no IAM call, no mirror write |
| IAM unreachable | 401, and `/readyz` degraded — never an open door |
| no `X-Pro-Token` | 401 |

Plus `scripts/iam_smoke.py` against a **real** IAM on `:8001` — sign in, call `api/graphql` with
the token, then call again with a forged `X-Pro-Client-Ip` and confirm refusal. **Stage 1 results
are reported before stage 2 begins.**

Stage 2: `IsAdmin` refuses an unauthenticated query; GraphiQL absent when `is_prod`; a
dataloader test asserting one query per relation kind, not one per row.

Stage 3: `tests/fixtures/markdown/` — a worked example per type plus every rejection; dry-run
writes nothing; unchanged file ⇒ `skipped_unchanged` and no new revision; unresolved slug ⇒
warning recorded on the version; zip traversal / symlink / bomb / over-count all refused; and the
XSS case the 2026-09-03 doc calls for — a body containing `<script>`, `<img onerror>`,
`javascript:` and `<iframe>` survives as inert text in the stored `body_html`.

### c.7 Verification

```sh
make check                                   # ruff + pytest
.venv/bin/mypy src                           # strict; the deps/settings rewiring earns it

# Stage 1, against a real IAM
cd ../iam.anudeep.pro && make dev &           # :8001, its own pg :5433 / redis :6380
python -m anudeep.iam.cli mint-service-token --name api.anudeep.pro
cd ../api.anudeep.pro && make dev             # :8000
curl -s localhost:8000/readyz                 # {"status":"ok","database":"ok","redis":"ok","iam":"ok"}
PYTHONPATH=src .venv/bin/python scripts/iam_smoke.py

# Stage 2
open http://127.0.0.1:8000/graphql            # GraphiQL, dev only
curl -s localhost:8000/graphql -H "X-Pro-Token: $TOK" -H 'Content-Type: application/json' \
  -d '{"query":"{ items(type: TECHNOLOGY) { slug title visible } }"}'

# Stage 3
curl -s -X POST "localhost:8000/api/v1/ingest?dry_run=1" -H "X-Pro-Token: $TOK" \
  -F 'files[]=@tests/fixtures/markdown/technology/python.md'
```

Each stage ends `make check`-green with its own commit — detailed body, no Claude attribution in
the message, per the git rules.

### c.8 Risks

- **The IAM README is stale on `/validate`.** It shows a JSON body; the code takes `X-Pro-*`
  headers. Building from the prose produces a client that authenticates nothing. Build from
  `iam/routers/validate.py` and `iam/schemas/validate.py`. *(Worth a follow-up fix in that repo.)*
- **Forwarding the wrong ip/UA fails silently.** Every response is a well-formed
  `{"active": false}`, which reads as "the token is bad" rather than "we sent our own
  fingerprint." Hence the explicit outbound-header assertion.
- **Cache key must include the fingerprint.** Omitting it defeats the replay protection while
  every test still passes. Called out in `_fingerprint_digest`'s docstring so a future
  simplification does not quietly remove it.
- **`.env` regeneration.** `make env` only writes `.env` when absent; the existing one carries
  the removed `ANUDEEP_SESSION_SECRET` and none of the IAM keys. Both the `env` target and the
  local `.env` need updating, or `make dev` starts a server that cannot authenticate anyone.
- **Fail-closed coupling.** IAM down ⇒ the admin is down. Accepted: that is the cost of the
  stateless option, and the alternative — serving admin data during an identity outage — is worse.
```
