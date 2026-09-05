# iam.anudeep.pro — session storage: Redis and Postgres

**Date:** 2026-09-04
**Repo:** `iam.anudeep.pro` (package `anudeep.iam`)
**Status:** Implemented — 117 tests

---

## a) What am I trying to do

Replace signed session tokens with **opaque, fingerprint-bound sessions** that expire on
inactivity, and put each piece of state where it belongs.

Three requirements drove it:

1. The session token must carry nothing readable — an opaque key, not a claim set.
2. It must stop working 30 minutes after the last use, not 30 minutes after issue.
3. A stolen token must not work from somewhere else.

The JWT that was here satisfied none of them. Its claims are base64, its expiry is baked into
the signature so it cannot slide, and it is bearer-only — whoever holds it is the user.

---

## b) Where each piece of state lives, and why

| State | Store | Why there |
|---|---|---|
| Live session | **Redis** | The idle window *is* the key's TTL. In Postgres it is a column plus an `UPDATE` on every request and a sweeper job — a write per read for data worthless in half an hour |
| Fingerprint (digest) | **Redis**, on the session | Compared on every validation; needed exactly where the session is read |
| Pending handles, handoff codes | **Redis** | Seconds to minutes; already there |
| Accounts, credentials, TOTP secret | **Postgres** | Durable, relational, must survive anything |
| Recovery codes | **Postgres** | Single-use, tied to an account by foreign key |
| Sign-in record + readable IP/agent | **Postgres** | Append-only history. The only place the components of a fingerprint are legible |

The split is deliberate: Redis holds what expires, Postgres holds what happened.

### Redis keyspace

```
session:<sha256(token)>            hash    TTL 1800s, reset on every use
  user_id           uuid of the account
  epoch             token_epoch at issue; a bump invalidates without a delete
  fingerprint       sha256(ip_network + NUL + user_agent)
  issued_at         unix seconds
  absolute_expiry   unix seconds — the ceiling activity cannot push past

sessions:user:<user_id>            set     TTL 1_209_600s
  every live session digest for one account, so logout-everywhere is O(n) on that
  account rather than a SCAN of a shared keyspace

pending:<sha256(handle)>           hash    TTL 300s login / 600s enrol / 60s handoff
  user_id, purpose, attempts, totp_secret?, audience?

pending:spent:<sha256(handle)>     string  TTL 900s
  tombstone, so a replay is distinguishable from ordinary expiry
```

**Persistence changed with this.** Redis previously ran `--appendonly no`, which was right
when it held only five-minute handles — losing them costs an in-flight sign-in. Holding
sessions changes that: a restart would sign everyone out. It now runs
`--appendonly yes --appendfsync everysec`, risking at most a second of TTL slides.

### Postgres tables (schema `iam`)

```
admin_user
  id uuid pk · email citext unique · password_hash text (argon2id)
  totp_secret text? · token_epoch int · last_login_at · created_at

recovery_code
  id uuid pk · user_id → admin_user (cascade) · code_hash text (argon2id)
  used_at? · created_at

login_attempt
  id bigint identity pk · email citext? · ip inet · success bool
  user_agent text? · fingerprint char(64)?      ← added by 0002
  created_at
  idx (ip, created_at desc) · idx (email, created_at desc)
```

`refresh_token` was **dropped** in `0002`. Rotation existed to extend a fixed-expiry token,
and its reuse detection is replaced by fingerprint binding — which catches a stolen credential
on first use elsewhere rather than on a race.

`login_attempt` now does double duty: it feeds the rate limiter *and* is the audit trail. It
holds the readable IP and user agent and the digest they produced, which is what lets a later
mismatch be traced back to the sign-in it diverged from.

---

## c) The fingerprint

```python
fingerprint = sha256(ip_network + b"\x00" + user_agent)
```

- **IPv4 → /24, IPv6 → /64.** Exact-IP binding logs people out constantly: phones move between
  cell and wifi, IPv6 privacy extensions rotate hourly. A prefix survives that and still
  refuses a token replayed from another network.
- **NUL separator.** Without it `ip="1.2.3.4" ua="0Mozilla"` and `ip="1.2.3.40" ua="Mozilla"`
  hash identically.
- **Addresses normalised first** — `2001:db8::1` and `2001:0db8:0000::0001` are one address.
- **Digest only** is stored. Whoever reads Redis learns nothing about where the account signs
  in from; the readable values live in `login_attempt`.

**It is a second condition, not the source of strength.** The token is 256 random bits on its
own. Deriving the token *from* these values would be the mistake: an email is known, an IP is
often known, and a login time within a day is about 2^17 possibilities — a 256-bit-wide value
carrying roughly seventeen bits of secret.

A mismatch **revokes the session** rather than just failing the check: a replay from elsewhere
is a theft signal, and the legitimate holder losing the session is the correct outcome.

---

## d) Lifetime

```
issue      →  TTL 1800s,  absolute_expiry = now + 14 days
every use  →  TTL reset to min(1800, remaining to absolute_expiry)
30m idle   →  key expires; token means nothing
14 days    →  ceiling reached; touch() revokes regardless of activity
```

Two clocks, because one is not enough: the idle window ends abandoned sessions, and the
ceiling stops a session living forever by being poked hourly.

---

## e) Endpoints

| | |
|---|---|
| `POST /api/v1/auth/login` | email + password → pending handle |
| `POST /api/v1/auth/login/totp` | handle + code → session token |
| `POST /api/v1/auth/logout` | ends this session |
| `POST /api/v1/auth/logout-all` | bumps epoch, drops every session for the account |
| `POST /api/v1/validate` | `X-Pro-Token`, `X-Pro-Client-Ip`, `X-Pro-User-Agent` → `{active, sub, email, epoch, …}` |
| `POST /api/v1/handoff/exchange` | `X-Pro-Code` → identity, for another origin |

**Service-to-service calls carry no body.** Every input travels as an `X-Pro-*` header,
because a header is not written to access logs, proxy logs or browser history the way a URL
is. `X-Pro-Token-Issued-At` is optional and lets a caller pin the exact session it believes
it holds — a mismatch reads as inactive, and an unreadable value is ignored rather than
treated as fatal, since an optional assertion should not be the difference between a live
session and a dead one.
| `/register`, `/login`, `/login/authenticator`, `/login/setup` | the pages |

**A calling service must forward the end user's IP and agent** to `/validate`. Forwarding its
own fingerprints the service, every session looks alike, and the binding checks nothing —
which is why there is a test asserting that forwarding neither returns `active: false`.

---

## f) Verification

| Check | How |
|---|---|
| Opaque token | `anut_…`, no dots, no readable claims — asserted |
| Idle slide | TTL read before and after a use; moves back to 1800 |
| Absolute ceiling | `touch()` revokes once `absolute_expiry` passes |
| Same /24 accepted | Session survives `127.0.0.1 → 127.0.0.250` |
| Other network refused | `198.51.100.4` → `active: false` |
| Other agent refused | `curl/8.4` → `active: false` |
| Mismatch burns it | The legitimate holder is refused afterwards too |
| Service forgets to forward | `active: false`, with a test naming the mistake |
| Audit row | `login_attempt` carries readable ip, agent and a 64-char digest on success; none on failure |
| Redis holds no plaintext | `HKEYS` shows `fingerprint`, never ip or user agent |
