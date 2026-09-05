# Analysis of the Seven Hardening Recommendations — iam.anudeep.pro

**Date:** 2026-09-04
**Repo:** `github.com/AnudeepChPaul/iam.anudeep.pro`
**Verified against:** `main` @ `8e926f9`
**Companion document:** `architecture-and-auth-flow-reference-2026-09-04.md`

---

## The re-arranged list

Ordered by **value per unit effort**, not by absolute severity. Items 1-8 total roughly an
afternoon; the reasoning is that cheap correctness work should not queue behind a security
feature whose present exposure is limited by `registration_mode=single`. If you would rather
order by absolute severity alone, item 9 moves to the top and everything above it slides down.

*(Bracketed numbers refer to the original seven-item list; "missed" marks findings that were
not on it.)*

**Do first — an afternoon, together**

1. **Correct the Redis persistence claim in the README.** `README.md:344-347` states Redis is
   "deliberately configured without persistence… nothing durable." False since `0002_sessions`,
   and directly contradicted by `docker-compose.yml:26-28`. One line; prevents a production
   misconfiguration that would sign out every user at once. *[6, docs half]*
2. **Delete the stale service-token documentation and dead test scaffolding.**
   `README.md:105-108` documents `X-IAM-Service-Id`/`X-IAM-Service-Token`, removed in `5016e17`;
   `conftest.py:19` and `test_auth.py:52-53` still send them. The scaffolding makes the suite
   look as though service auth is covered, which conceals item 9. *[missed]*
3. **Resolve `IAM_SESSION_SECRET`.** Required at `settings.py:46`, read only by
   `AuthService._secret` (`auth.py:109`), which has zero call sites. Delete it, or reserve it as
   the key for item 12. *[missed]*
4. **Fix the fingerprint-source divergence.** `current_user` (`deps.py:123`) ignores `X-Pro-*`
   while `/v1/validate` honours it, and `resolve_session` *revokes* on mismatch — so one call to
   `/api/v1/auth/me` destroys a proxy-issued session. *[missed]*
5. **Fix the remaining README defects** — contradictory ports (`:340` vs `:327`), duplicate
   "## Sessions" and "## Validating a token" sections. *[missed]*
6. **Close the single-registration race.** `auth.py:137`. Note the mid-flight commit at
   `auth.py:579` defeats a `pg_advisory_xact_lock`; use a session-level lock or move the audit
   write off the request transaction. *[3]*
7. **Add the password-change route.** `PasswordChangeRequest` (`schemas/auth.py:33`) already
   exists and is dead. Must `bump_epoch` + `revoke_all`. *[5, part]*
8. **Add `token_issued_at` to the recovery-login response.** `routers/auth.py:54`. *[missed]*

**Then — the first real security feature**

9. **Trusted-proxy enforcement for `X-Pro-*`.** `IAM_TRUSTED_PROXY_IPS` checked against the real
   peer in `caller_identity` (`deps.py:68-83`), plus a Cloudflare Transform Rule stripping the
   headers on public ingress. Blocked conceptually by `5016e17` having removed service tokens —
   there is no longer any way to identify a trusted caller. *[1]*

**Then**

10. **Token audiences on sessions.** One `anut_` token is currently honoured by every service on
    the estate. Costlier per service integrated, so do it before more arrive. *[4, part]*
11. **Account suspension + CLI `reset-password`.** One column and one check; one CLI subcommand
    beside `create-user`. *[5, part]*

**Later**

12. **Encrypt TOTP secrets at rest**, keyed by the secret freed in item 3. *[2]*
13. **Redis HA topology** beyond the item-1 documentation fix. *[6, infra half]*

**Deferred, deliberately**

14. **RBAC (roles + scopes)** until a second class of user exists. *[4, part]*
15. **Email-based password reset and verification** unless `registration_mode` moves to `open`;
    no email channel exists in the repo. *[5, part]*
16. **WebAuthn / device-bound keys** — and note it subsumes items 9, 12 and the second-factor
    half of 11, which argues for keeping item 9 minimal rather than elaborate. *[7]*

---

## a) What am I trying to do

Seven hardening recommendations were proposed, in a stated priority order. This document
assesses each one against the actual source — not against the README, which as it turns out
describes a system that no longer exists in several places.

For each recommendation I establish four things:

1. **Is it real?** Does the defect exist in the code as written, with a `file:line` citation.
2. **What is the actual exposure?** Not the category of risk, but the specific thing an
   attacker gains — because two of these are much narrower than their headline suggests, and
   one is broader.
3. **What does it cost to fix?** Two of the seven bundle work spanning an afternoon and work
   spanning a month under a single bullet.
4. **Is the stated priority right?**

I then add the defects the list missed, and propose a re-sequenced plan.

**Headline conclusion.** All seven are real; none is a false positive. But the ordering is
wrong in four places, items #4 and #5 each need splitting, and the list omits a cluster of
cheaper defects — including documentation that is not merely absent but actively wrong in a
way that will cause a production misconfiguration, and a latent bug that silently *destroys*
sessions rather than refusing them.

---

## b) What are my options

Three of the seven involve a genuine architectural choice rather than a single obvious fix.
Those get an options analysis here. The remaining four have one sensible implementation and
are covered in section (c).

### b.1 — Recommendation #1: enforcing trust on `X-Pro-*`

**The defect is confirmed.** `deps.py:81-82`:

```python
ip = request.headers.get(HEADER_CLIENT_IP.lower()) or client_ip(request)
agent = request.headers.get(HEADER_USER_AGENT.lower()) or request.headers.get("user-agent")
```

No trust check. Any client that can reach `/v1/login` sets its own apparent address. The
docstring acknowledges this deliberately — "These headers are not a trust boundary on their
own" — so it is a known trade rather than an oversight. The question is whether the trade is
still correct, and it is not, for a reason the docstring could not have anticipated.

**What the attacker actually gains** — narrower than "IP spoofing" implies:

| Control | Effect of forgery |
|---|---|
| Per-account rate limit (5 / 900s) | **Holds.** Keyed on email, not IP. Single-account brute force is still capped |
| Per-IP rate limit (20 / 900s) | **Defeated.** Rotate the header per request; the limit never trips |
| `login_attempt.ip` audit trail | **Poisoned.** The forensic record is attacker-controlled |
| Session fingerprint binding | **Weakened.** A thief who also knows the victim's /24 and UA can bind or replay |

So the real exposure is **password spraying** — unbounded attempts across *many* accounts —
plus an untrustworthy audit trail. Not single-account brute force, which the per-account limit
still contains. That distinction matters for severity: on a service where
`registration_mode=single` and exactly one account exists, spraying has one target and the
per-account limit is the binding constraint. The severity therefore scales with account count,
and is presently low but structurally wrong.

**The complication that makes this harder than it looks.** Commit `5016e17` ("Remove service
tokens and the handoff exchange") deleted the only mechanism for identifying a trusted service
caller. There is now no credential distinguishing "the admin SPA forwarding a real user's
address" from "an attacker sending whatever they like." So the obvious fix — *trust `X-Pro-*`
only from authenticated services* — cannot be implemented without reintroducing something.

```
        TODAY                                     REQUIRED
                                                                  
   any client ──X-Pro-Client-Ip: <anything>──▶    trusted caller ──X-Pro-*──▶ honoured
                                          │                                        
                            honoured unconditionally               public client ──X-Pro-*──▶ ignored,
                                                                          real peer used instead
```

#### Option 1A — Cloudflare Transform Rule strips `X-Pro-*` on public ingress

A rule on the public zone removes inbound `X-Pro-*` headers before they reach the origin.
Internal service traffic reaches IAM by a path that does not apply the rule.

- **Pros:** zero code; zero new secrets; enforced before a request consumes an origin worker;
  composes with `OriginLockMiddleware`, which already proves traffic came through the proxy.
- **Cons:** the security property lives in vendor configuration rather than in the repo, so it
  is invisible to tests and to anyone reading the code; needs a documented, reviewable
  Terraform/dashboard change; a second ingress path added later silently bypasses it.

#### Option 1B — Gate the trust behind the existing `IAM_ORIGIN_AUTH_HEADER`

`OriginLockMiddleware` (`security/headers.py:82`) already compares a shared secret in
`cf-origin-auth`. Reuse that signal: honour `X-Pro-*` only when the origin lock passed.

- **Pros:** in-code and testable; no new configuration — the secret already exists and is
  already required in production; a natural reading of "inside the perimeter."
- **Cons:** conflates two distinct claims — "came through our proxy" and "is entitled to speak
  for another user." Every proxied request would be trusted, including a browser's, which is
  most of the traffic. This weakens the property to almost nothing in production, where all
  traffic is proxied. **Not recommended on its own.**

#### Option 1C — An explicit `IAM_TRUSTED_PROXY_IPS` allowlist  ✅ **recommended, with 1A**

Compare the *real* peer (`request.client.host`, before any header override) against a
configured CIDR allowlist. Honour `X-Pro-*` only for a peer in that set; otherwise ignore the
headers and use the connection's own address.

- **Pros:** the check is in the repo, unit-testable, and states the actual rule; independent of
  the vendor; fails closed for anything unlisted; a natural home for the Railway private-network
  ranges the sibling services occupy.
- **Cons:** an operational list to maintain; behind Cloudflare the peer is a Cloudflare address,
  so the allowlist must either enumerate Cloudflare ranges (too broad — that is every proxied
  request again) or, correctly, the internal service network only.

**Recommendation: 1C in the application, 1A at the edge.** 1C makes the rule explicit and
testable for internal callers; 1A prevents the header ever arriving from the public internet in
the first place. Neither alone is sufficient: 1A is invisible to the codebase, and 1C behind a
proxy needs 1A to keep the peer meaningful. 1B is rejected — it would trust every browser.

---

### b.2 — Recommendation #4: authorization model

The recommendation bundles **roles**, **scopes** and **token audiences**. Confirmed absent —
grep for role/scope/permission/audience across `src/` returns only an SVG `role="img"`
attribute and unrelated prose. But these are three different propositions with three different
justifications, and bundling them obscures that one is worth doing now and two are not.

#### Option 4A — Full RBAC now (roles + scopes + audiences)

- **Pros:** done once, correctly; no migration of an established token format later.
- **Cons:** `registration_mode=single`; exactly one account class exists. Roles and scopes for
  one user is a permission system with nothing to permit — speculative structure that must be
  maintained and reasoned about before it does any work.

#### Option 4B — Token audiences only  ✅ **recommended**

Bind each session to the service(s) it may be presented to, and have `/v1/validate` refuse a
token presented for the wrong audience.

- **Pros:** addresses a real property that has nothing to do with user count. Today **one
  `anut_` token is honoured by every service on the estate** — a token captured by, logged by,
  or leaked from the lowest-value service is immediately valid at the admin. Audience is the
  blast-radius control, and it is *cheaper now than later*: it changes the introspection
  contract, so the cost grows with every service that integrates.
- **Cons:** callers must declare an audience; `SessionRecord` gains a field; a migration path
  for tokens already issued (mitigated — sessions are ≤14 days by construction, so a two-week
  dual-read window retires them all without any explicit migration).

#### Option 4C — Defer all of it

- **Pros:** nothing speculative gets built.
- **Cons:** accepts the shared-blast-radius property indefinitely and pays a rising integration
  cost to fix it.

**Recommendation: 4B now, 4A deferred** until a second class of user actually exists. Roles
invented before there is a distinction to encode will encode the wrong distinction.

---

### b.3 — Recommendation #5: the account-lifecycle bundle

Four features under one bullet, spanning an afternoon to a month. They must be costed
separately.

| Feature | State today | Cost | Verdict |
|---|---|---|---|
| Password **change** | `PasswordChangeRequest` exists at `schemas/auth.py:33` and is **referenced nowhere** — a dead schema | ~1 route | **Do it** |
| Account **suspension** | No `disabled`/`suspended` column anywhere. Only levers: delete the row, or `bump_epoch` (logs out, does **not** prevent signing back in) | 1 column + 1 check in `resolve_session` | **Do it** |
| Password **reset** | No email infrastructure of any kind — no SMTP dependency, no provider, nothing in `pyproject.toml` | New provider, deliverability, templates, token lifecycle, new attack surface | **See below** |
| Email **verification** | Same — no channel exists | Same | **See below** |

The first two are genuinely missing and nearly free. The second two are gated on an email
channel that does not exist in the repo at all.

#### Option 5A — Build the email channel and do self-service reset

- **Pros:** the expected flow for a normal consumer product; no operator involvement.
- **Cons:** a mail provider becomes a production dependency of the *auth* system; reset tokens
  are a new credential class with their own expiry, single-use and enumeration concerns; email
  deliverability becomes an on-call issue; substantial surface for one account.

#### Option 5B — CLI password reset  ✅ **recommended**

Add `reset-password` beside the existing `create-user` and `clear-lockout` in `cli.py`.

- **Pros:** ~15 lines against helpers that already exist (`hash_secret`, `AdminUserRepository`,
  `bump_epoch`); no new dependency, no new credential class, no deliverability concern. The
  operator and the account holder are the same person on this service, and the CLI is already
  the established bootstrap and un-lockout path. Recovery codes already cover the *second
  factor* loss case.
- **Cons:** requires shell access — unacceptable for a multi-tenant product, entirely
  appropriate for a single-admin one.

**Recommendation: 5B.** Revisit 5A only if `registration_mode` moves to `open` and real users
appear. Email verification is likewise moot while registration is `single` — there is one
account and its address is known to its owner.

---

## c) The selected plan

### c.a) Findings the recommendation list missed

These were not in the seven and several are cheaper than anything that was.

#### M1 — `IAM_SESSION_SECRET` is a required setting that nothing reads

`settings.py:46` declares it mandatory. Its only reader is:

```python
# services/auth.py:108-110
@property
def _secret(self) -> str:
    return self.settings.session_secret.get_secret_value()
```

`grep -rn "self\._secret" src` returns **nothing**. The property is dead; the setting it wraps
is dead. It is a leftover from the JWT era, retained after `0002_sessions` made tokens opaque.
Every deployment is required to generate and manage a secret that does nothing.

This connects directly to recommendation #2: it is a ready-made, already-required, already-
provisioned key for encrypting TOTP secrets. Either delete it or give it that job — but it
should not stay as it is.

#### M2 — `current_user` and `/v1/validate` disagree on the fingerprint source

Two code paths resolve a session, and they read the caller's identity differently:

```
/v1/login, /v1/login/totp, /v1/register…   →  caller_identity()   honours X-Pro-*   (deps.py:81)
/v1/validate                               →  X-Pro-* headers     honours X-Pro-*   (validate.py:66)
/api/v1/auth/me, /logout-all               →  client_ip()         IGNORES X-Pro-*   (deps.py:123)
```

A session **issued** through a proxy that forwards `X-Pro-Client-Ip: 198.51.100.7` is bound to
that network. `/v1/validate` resolves it correctly. But `/api/v1/auth/me` recomputes the
fingerprint from the real connection, mismatches — and `resolve_session` does not merely refuse:

```python
# services/auth.py:515-522
if not fingerprint.matches(record.fingerprint, ip, user_agent):
    log.warning("session_fingerprint_mismatch", ...)
    await self.sessions.revoke(digest, record.user_id)   # ← destroys the session
    return None
```

So one call to `/api/v1/auth/me` with a proxy-issued token **destroys** that session rather than
returning 401 for it. The test suite confirms the binding behaviour
(`test_auth.py:486-490`) but only ever exercises it through `/v1/validate`, so the divergent
path is untested.

This is defensible *if* the intended contract is "proxied sessions are introspected, never used
directly." That contract is nowhere stated, and the revoke-on-mismatch makes the failure
destructive rather than merely inconvenient.

#### M3-M6 — Documentation that is wrong, not merely missing

| Location | Problem |
|---|---|
| `README.md:105-108` | Documents `X-IAM-Service-Id` and `X-IAM-Service-Token` and "validation is rate-limited per registered service in Redis." **All removed in `5016e17`.** An integrator following the README will implement headers that do not exist |
| `README.md:344-347` | "Redis holds one thing: the half-authenticated state… **deliberately configured without persistence** — losing it costs an in-flight sign-in and nothing durable." **False since `0002_sessions`.** Redis holds live sessions; `docker-compose.yml:26-28` sets `appendonly yes --appendfsync everysec` with a comment saying *"Holds live sessions now, not just five-minute handles: losing it signs everyone out."* The README and the compose file state opposite things |
| `README.md:340` vs `:327` | Ports contradict: `:5432`/`:6379` versus the correct `:5433`/`:6380` |
| `README.md:162,251` / `:196,285` | "## Sessions" and "## Validating a token" each appear **twice** |

M4 is the dangerous one and is the strongest argument for promoting recommendation #6. Absent
documentation makes an operator ask; *wrong* documentation makes them act. That single sentence
is exactly what would justify provisioning production Redis without persistence — the failure
mode being every user signed out simultaneously, with no way to distinguish it from a breach.

#### M7 — Dead test scaffolding disguises the gap

`tests/conftest.py:19` sets `IAM_SERVICE_TOKENS`, and `tests/integration/test_auth.py:52-53`
sends `X-IAM-Service-Id` / `X-IAM-Service-Token` on every introspection call. Both are for the
feature deleted in `5016e17`. They are silently ignored, so the suite *reads* as though service
authentication is covered when nothing enforces it. This actively conceals the gap
recommendation #1 is about.

#### M8 — `routers/auth.py:54`

The recovery-login `TokenResponse` is constructed without `token_issued_at`, unlike every other
issuance path. Relevant to any caller pinning `X-Pro-Token-Issued-At` after a recovery sign-in.

---

### c.b) Flow diagrams

#### The spraying path that recommendation #1 closes

```
ATTACKER                         IAM                              POSTGRES
   │                              │                                   │
   │ POST /v1/login               │                                   │
   │ X-Pro-Client-Ip: 10.0.0.1    │                                   │
   │ {alice@…, "guess1"}          │                                   │
   ├─────────────────────────────▶│ _enforce_rate_limit(alice, 10.0.0.1)
   │                              ├─ recent_failures ────────────────▶│
   │                              │   by_ip=0  by_account=0           │
   │                              │   ✓ under both limits             │
   │◀────────── 401 ──────────────┤  INSERT login_attempt(ip=10.0.0.1) ← attacker-chosen
   │                              │                                   │
   │ X-Pro-Client-Ip: 10.0.0.2  ← rotate per request                   │
   │ {bob@…, "guess1"}            │                                   │
   ├─────────────────────────────▶│   by_ip=0 (new "IP")  ✓           │
   │◀────────── 401 ──────────────┤                                   │
   │                              │                                   │
   │  … unbounded across N accounts; per-IP cap of 20 never trips …   │
   │  per-ACCOUNT cap of 5 still holds → this is SPRAYING, not brute force
```

With 1C in place the peer is checked first, the forged header is discarded, and every attempt
counts against the *real* address — restoring the 20-per-window ceiling.

#### The proposed `X-Pro-*` trust boundary

```
                        ┌────────────────────────────────────────────┐
   public browser ──────▶│ Cloudflare                                 │
   (may send X-Pro-*)   │  Transform Rule: STRIP inbound X-Pro-*  (1A)│
                        └───────────────────┬────────────────────────┘
                                            │ cf-origin-auth: <secret>
                                            ▼
                        ┌────────────────────────────────────────────┐
                        │ OriginLockMiddleware  (already exists)     │
                        └───────────────────┬────────────────────────┘
                                            ▼
                        ┌────────────────────────────────────────────┐
   internal service ───▶│ caller_identity()                      (1C)│
   (Railway private net)│   peer = request.client.host               │
                        │   peer ∈ IAM_TRUSTED_PROXY_IPS ?           │
                        │     yes → honour X-Pro-Client-Ip / -Agent  │
                        │     no  → ignore them; use the real peer   │
                        └────────────────────────────────────────────┘
```

#### The registration race (#3), and why the obvious fix does not work

```
   REQUEST A                          REQUEST B
       │                                  │
       ├─ users.count() → 0               │
       │                                  ├─ users.count() → 0     ← both see zero
       ├─ _require_registration_open(0) ✓ │
       │                                  ├─ _require_registration_open(0) ✓
       ├─ users.create(alice)             │
       ├─ _record() → session.commit()  ←─┼── COMMITS MID-FLIGHT (auth.py:579)
       │                                  ├─ users.create(mallory)
       │                                  ├─ _record() → commit
       ▼                                  ▼
              TWO admin accounts under registration_mode="single"
```

The subtlety: `_record()` commits inside `register()`, deliberately, so the audit row survives
a rollback. That commit **would release a `pg_advisory_xact_lock`** taken at the top of
`register()`, leaving the guard ineffective. The fix must therefore be either a
session-level `pg_advisory_lock` explicitly released in a `finally`, or the audit write moved
to a separate connection so the registration transaction is a real unit. The second is cleaner
and also removes the standing oddity of a service method owning a commit.

Severity is genuinely low — the window is one bootstrap moment — but the fix is small and the
failure is silent and permanent.

---

### c.c) Re-sequenced plan

#### Tier 0 — hours, highest value per unit effort

| Item | Files | Notes |
|---|---|---|
| Correct the Redis durability claim | `README.md:344-347` | Contradicts `docker-compose.yml`; the most consequential single line in the repo |
| Fix contradictory ports; delete duplicate sections | `README.md:340`, `:162/251`, `:196/285` | |
| Delete stale service-token documentation | `README.md:105-108` | Describes headers removed in `5016e17` |
| Remove dead test scaffolding | `tests/conftest.py:19`, `tests/integration/test_auth.py:52-53` | Currently disguises the #1 gap |
| Resolve `IAM_SESSION_SECRET` | `settings.py:46`, `services/auth.py:108-110` | Delete, or reserve as the Tier 3 encryption key |
| Fix the fingerprint-source divergence (M2) | `deps.py:114-148` | Decide the contract, then make both paths agree; at minimum stop *revoking* on a mismatch that the architecture makes routine |
| Registration race (#3) | `services/auth.py:137,167,579` | Session-level advisory lock, or move the audit write off the request transaction |
| Password change route (#5a) | `routers/auth.py`, `schemas/auth.py:33` | Schema already exists. **Must** `bump_epoch` + `revoke_all` — the model comment at `models/user.py:26` already specifies this |
| Add `token_issued_at` (M8) | `routers/auth.py:54` | |

#### Tier 1 — the actual first priority

**Trusted-proxy enforcement (#1)** — Option 1C in code plus 1A at the edge.

- `settings.py` — add `trusted_proxy_ips: list[str]`, parsed by the existing `_parse_json_list`
  validator so it accepts a JSON array or a comma-separated string like its siblings.
- `deps.py:68-83` — `caller_identity` checks `request.client.host` against the allowlist
  *before* consulting any header; falls back to `client_ip(request)` when unlisted.
- Tests — the gap M7 hides: assert a forged `X-Pro-Client-Ip` from an untrusted peer is
  ignored, and that per-IP rate limiting counts against the real address.
- Edge — a documented Cloudflare Transform Rule stripping inbound `X-Pro-*` from the public zone.

#### Tier 2

- **Token audiences (4B)** — `SessionRecord` gains an audience; `/v1/validate` refuses a
  mismatch. Sessions expire within 14 days by construction, so a dual-read window retires
  every existing token without a migration.
- **Account suspension (5b)** — one column, one check in `resolve_session`, one CLI subcommand.
- **CLI `reset-password` (5B)** — beside `create-user` and `clear-lockout` in `cli.py`.

#### Tier 3

- **TOTP encryption at rest (#2)** — AEAD (`cryptography`, a new dependency) keyed by the
  repurposed `IAM_SESSION_SECRET`; decrypt-on-verify in `_totp_valid` and
  `confirm_totp_enrolment`. Closes the standing asymmetry: passwords and recovery codes are
  argon2-hashed, the TOTP secret is the sole plaintext credential in Postgres.
- **Redis HA topology (#6)** beyond the Tier 0 documentation fix — replica or managed failover,
  and a written recovery procedure for total session loss.

#### Deferred, deliberately

- **RBAC (4A)** until a second class of user exists.
- **Email-based reset and verification (5A)** unless `registration_mode` moves to `open`.
- **WebAuthn (#7)** — correctly last, and worth stating explicitly: it **subsumes** #1 (device-
  bound keys make IP/UA fingerprinting unnecessary), #2 (no shared secret at rest to encrypt)
  and the second-factor half of #5. That is a direct argument against over-investing in
  fingerprint hardening now — do the cheap correctness fix in Tier 1, not an elaborate one.

---

### c.d) Summary — proposed versus stated priority

| Stated | Item | Proposed | Reason for the change |
|---|---|---|---|
| 1 | Trusted proxy for `X-Pro-*` | **Tier 1** | Confirmed. Exposure is spraying + audit poisoning, not brute force. Complicated by service tokens having been removed |
| 2 | Encrypt TOTP at rest | **Tier 3** | Only helps against DB-without-env access — but cheap, because a required unused secret already exists |
| 3 | Registration race | **Tier 0** | Low severity, near-zero cost. The mid-flight commit makes the obvious fix wrong |
| 4 | Roles, scopes, audiences | **Split: audiences Tier 2, RBAC deferred** | RBAC for one account is speculative; audience is a real blast-radius control that gets costlier per integration |
| 5 | Reset, change, verify, suspend | **Split: change + suspend Tier 0/2, reset via CLI, email deferred** | Four features, one bullet. Two are ~free; two need an email channel that does not exist |
| 6 | Redis HA + docs | **Docs Tier 0, HA Tier 3** | The docs are *wrong*, not missing — README contradicts `docker-compose.yml` on persistence |
| 7 | WebAuthn later | **Deferred — agreed** | Subsumes #1, #2 and part of #5; an argument for keeping the #1 fix minimal |
| — | *8 missed findings (M1-M8)* | **Mostly Tier 0** | Dead required config, a session-destroying inconsistency, and four documentation defects |
