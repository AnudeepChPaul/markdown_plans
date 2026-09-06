# iam.anudeep.pro — implementation deviations and findings

Companion to `python-to-nodejs-migration-2026-09-05.md`. That document is the design; this one
records where the built service departs from it, and what the port turned up that the design
did not anticipate. Stages 3–10 are implemented; tests are deferred by explicit instruction.

## a) What I was trying to do

Port the Python identity service to Node/TypeScript feature-for-feature, then keep a truthful
record of every place the result is *not* what the plan said — because a plan that silently
stops matching the code is worse than no plan.

## b) Deviations from the plan

| Plan said | Built | Why |
|---|---|---|
| `qr.py` ported as a hand-rolled matrix walk | `qrcode`'s own `toString({type:'svg'})` | The dependency was already present and emits SVG directly. Hand-rolling the run-length path would be new, untested code for an identical result. |
| `/ui` content-negotiates like the Python (`X-Requested-With: fetch` → JSON) | HTML only, always `303` | Per the instruction that "JSON APIs return JSON, UI ones deal with HTML only". Removing the JSON path removes a second response shape that had to stay in agreement with the first. |
| `BrowserStartResponse` / `BrowserTokenResponse` | Not ported | They existed only to serve that JSON path. |
| Metrics via OTLP | Same, but needed `@opentelemetry/sdk-metrics` added explicitly | `NodeSDK` takes a `metricReader`; without one the counters record and never export. Pinned to `2.11.0` to match `sdk-node`'s own resolution. |
| — | `@fastify/formbody` added | HTML forms arrive urlencoded, not JSON. |

## c) Defects found and fixed during the port

These are behaviours the Python has today.

1. **Pending-handle race.** `spend()` discarded `DEL`'s reply, so two concurrent requests with
   one handle and one valid code both got sessions. Now returns the reply; exactly one caller
   wins. Verified concurrently.
2. **Recovery-code race.** Read-then-write on `used_at`. Now
   `UPDATE … WHERE used_at IS NULL … RETURNING`, evaluated under the row lock.
3. **`HINCRBY` on an expired handle** created a key with no TTL — an unbounded Redis leak
   drivable by submitting codes for expired handles. Now a script that checks existence first.
4. **Validate rate limiter did `INCR` then a separate `EXPIRE`.** A crash between them locks a
   service out permanently. Now one script that always sets an expiry.
5. **`Origin: null` was allowed.** The sandboxed-iframe case — precisely the forged-POST
   scenario — was treated as *absent* and fell through to the Referer check. Now refused.
6. **`ZodError` escaped as a 500.** A malformed email reported the server as broken rather than
   the request. Now `422` with the offending field named.
7. **`pino-pretty` crashed the production image.** It is a devDependency, removed by
   `pnpm prune --prod`, and pino resolves transports lazily — so a container that had not set
   `IAM_ENVIRONMENT=prod` died on its first log line. The transport is now conditional on the
   package actually resolving, and the runtime image sets `IAM_ENVIRONMENT=prod`.

## d) Open design question — NOT changed, parity kept

**`/v1/validate` revokes on fingerprint mismatch.**

`resolveSession` treats a mismatch as theft and revokes. `/v1/validate` resolves the token
using the `X-Pro-Client-Ip` / `X-Pro-User-Agent` the caller forwarded. So a service that does
not forward them does not merely get `{"active": false}` — it **destroys the user's session**,
for every user it asks about.

```
service forgets the headers
        │
        ▼
introspect(token, ip='') ──▶ fingerprint mismatch ──▶ sessions.revoke()
        │                                                    │
        ▼                                                    ▼
   {"active": false}                        the user is signed out everywhere,
                                            and a correctly-configured caller
                                            now also gets inactive
```

Confirmed by test: after one such call the token is dead even for a caller that forwards
correctly.

The Python behaves identically, so this is inherited rather than introduced, and I have kept
parity rather than changing it unilaterally. Worth deciding: introspection could refuse without
revoking, leaving revocation to the paths where the *user's own* request carries the mismatch.

## e) Verification performed

No unit tests yet (deferred). Everything below was exercised against real Postgres, real Redis
and a real Fastify instance:

- service layer end to end: registration, both login legs, enrolment, recovery, password
  change, introspection, and both races under `Promise.all`
- HTTP surface: 47 assertions across every `/v1` route, CSRF on the cookie path, bearer
  bypassing CSRF, problem+json shape, and the old `/api/v1/auth/*` paths returning 404
- UI surface: 45 assertions driven as a no-JS browser with a cookie jar — form-token CSRF,
  open-redirect refusal (`//evil.example`, absolute, `javascript:`), XSS through the echoed
  email, CSP nonce matching, and codes shown exactly once
- telemetry: the attribute guard refusing `email`/`ip`/`token`/`user_agent`/`*_code`/`*_handle`,
  span nesting, and clean flush on shutdown
- production Docker image: builds, serves, and issues `__Host-pro_form=pro_lf_…; Secure`,
  which confirms the environment marker and cookie prefix in a real deployed container
