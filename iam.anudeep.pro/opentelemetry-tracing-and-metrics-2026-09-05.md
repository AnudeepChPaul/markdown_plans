# OpenTelemetry: Traced Requests and Per-Operation Counters

**Date:** 2026-09-05
**Repo:** `github.com/AnudeepChPaul/iam.anudeep.pro`
**Branch:** `harden-auth-mandatory-2fa-recovery`
**Status:** Implemented and verified end to end against a live stack

---

## a) What am I trying to do

The service logs well and measures nothing.

`logging.py` gives structlog with JSON rendering and a `request_id` bound into contextvars by
`RequestContextMiddleware`, and twenty named events are already emitted — `account_registered`,
`recovery_before_enrolment`, `pending_handle_burned`, `session_fingerprint_mismatch`. Each is
readable on its own. Collectively they answer nothing:

- *How many sign-ins failed in the last hour, and were they one account or many?*
- *Show me everything that happened during the request that returned this id.*
- *Is the fingerprint mismatch rate rising — is someone replaying a stolen token?*
- *Why does signing in feel slow?*

Answering the first needs **metrics**. The second needs **traces**. The third needs both. The
fourth needs a duration histogram on the argon2 verify, which nothing currently times.

The goal is therefore all three OpenTelemetry signals, correlated, with two hard constraints
particular to this service:

1. **Nothing sensitive may be recorded.** This is an identity service. A span attribute is an
   exfiltration surface; a metric label is that *plus* an unbounded time series.
2. **Disabled must mean free.** Dev and CI must be unaffected — no exporter, no provider, no
   behavioural difference.

---

## b) What are my options

### b.1 — How the three signals are delivered

#### Option 1 — Full OTel SDK for all three signals, everything over OTLP

Traces, metrics *and* logs exported via the OTel SDK, replacing structlog's output path with
the OTel `LoggingHandler`.

- **Pros:** one pipeline, one backend, one wire protocol; no log shipper needed; logs arrive
  already tagged with trace context.
- **Cons:** replaces a structlog setup that works well and renders exactly as wanted; the logs
  signal is the least mature of the three in Python; a backend outage becomes a log outage.

#### Option 2 — OTel API only, no SDK; fake trace fields in logs

Emit trace-shaped fields from structlog without real spans.

- **Pros:** no dependencies.
- **Cons:** not tracing. No spans, no parent/child, no distributed continuation, no metric
  instruments. It would answer none of the four questions above.

#### Option 3 — Traces and metrics over OTLP; logs stay on stdout, correlated  ✅ **selected**

The SDK carries traces and metrics. structlog keeps rendering logs to stdout as JSON, with one
processor injecting `trace_id` and `span_id` into every line.

- **Pros:** the existing log pipeline is untouched; correlation costs one processor rather than
  a second exporter; logs survive a telemetry backend outage because they never went through
  it; this is the mainstream, boring pattern.
- **Cons:** logs are not in the OTLP pipeline, so a log shipper (or Loki/Promtail) is needed if
  they are to be queried alongside traces rather than read from stdout.

**Selected: Option 3.** The deciding argument is that the log pipeline was never the problem —
correlation was. Option 3 fixes exactly the missing thing and changes nothing that already
worked.

```
   ┌──────────────────────────────────────────────────┐
   │ iam.anudeep.pro                                  │
   │                                                  │
   │  structlog ──── stdout JSON  ── trace_id ──┐     │
   │                                            │     │
   │  OTel SDK ───── traces ────┐               │     │
   │            └── metrics ────┤               │     │
   └────────────────────────────┼───────────────┼─────┘
                                │ OTLP/HTTP     │ correlate by trace_id
                        ┌───────▼──────┐        │
                        │  Collector   │        │
                        └───┬──────┬───┘        │
                     traces │      │ metrics    │
                   ┌────────▼─┐  ┌─▼──────────┐ │
                   │  Tempo   │  │ Prometheus │ │
                   └────┬─────┘  └─────┬──────┘ │
                        └──────┬───────┘        │
                          ┌────▼─────┐          │
                          │ Grafana  │◀─────────┘
                          └──────────┘
```

### b.2 — Which backend

| | Jaeger + Prometheus | **Tempo + Prometheus + Grafana** | SigNoz |
|---|---|---|---|
| Signals | traces only (+Prom) | traces + metrics | all three |
| Containers | 2 | 3 (+collector) | 1 stack, heavy (ClickHouse) |
| Panes of glass | 2 | **1** | 1 |
| Metric → trace pivot | weak | **exemplars** | native |
| Portability of skills | moderate | **highest** | lower |

**Selected: Tempo + Prometheus + Grafana.** Jaeger does traces only, so half the request —
the counts — would need Prometheus anyway; once Prometheus is in the picture the comparison is
2 containers and 2 UIs versus 3 containers and 1. Exemplars decide it: clicking a spike in
`iam_login_attempts_total{outcome="invalid_credentials"}` and landing on a real trace is the
workflow the request implies, and without it that pivot is a manual timestamp hunt.

**A collector sits in front.** It costs a container and buys the property that the application
speaks OTLP and nothing else — swapping Tempo for anything is collector configuration, never
Python.

### b.3 — How operations get instrumented

- **Auto-instrumentation only** — free, but produces spans for HTTP/DB/Redis with no domain
  meaning. A trace would show five `SELECT`s and never say "this was a login".
- **A generic decorator recording ok/error** — uniform, but collapses the domain: `login`
  refusing a password and `login` reporting `totp_required` are both "not an error", and the
  interesting distinction disappears.
- **Auto-instrumentation, plus hand-placed spans and domain counters** ✅ — the libraries give
  the I/O for free; `@traced` names the unit of work; counters carry *domain* outcomes rather
  than success/failure.

---

## c) The selected design

### c.a) Architecture

```
 request ──▶ OriginLock ─▶ TrustedHost ─▶ RequestContext ─▶ CORS ─▶ SecurityHeaders ─▶ route
                                              │
                                              ├─ bind_request_id(contextvar)
                                              └─ structlog contextvars: request_id, method, path

 AuthService.login                      @traced("iam.login")
   ├─ _enforce_rate_limit               @traced("iam.rate_limit.check")
   ├─ verify_secret ......................... PASSWORD_HASH_DURATION.record()
   ├─ SELECT admin_user ..................... span from SQLAlchemyInstrumentor
   ├─ _issue_pending                    @traced("iam.pending.issue")
   │    └─ HSET pending:… ................... span from RedisInstrumentor
   ├─ LOGIN_ATTEMPTS.add(outcome=…)
   └─ log.info(...) ......................... carries trace_id, span_id, request_id
```

Every `iam.*` span carries `app.request_id`, read from a contextvar rather than stamped onto
"the current span" in middleware. That choice matters: middleware stamping only works when the
FastAPI instrumentor has opened a span, so the link would vanish exactly when tracing is only
partly configured — the case where it is most needed.

**Module boundary.** `observability.py` is the only module importing the OTel SDK. Everything
else imports named instruments and a decorator from it.

### c.b) Flow diagrams

#### Following one request, both directions

```
      A caller holds X-Request-ID: abc123
                    │
                    ▼
      TraceQL:  { span.app.request_id = "abc123" }
                    │
                    ▼
      ┌─────────── trace ────────────┐
      │ POST /v1/login               │  ← FastAPIInstrumentor
      │  ├── iam.login               │  ← @traced      app.request_id=abc123
      │  │    ├── iam.rate_limit…    │
      │  │    ├── SELECT admin_user  │  ← SQLAlchemy
      │  │    ├── SELECT login_att…  │
      │  │    ├── iam.pending.issue  │
      │  │    │    └── HSET EXPIRE   │  ← Redis
      │  │    └── INSERT login_att…  │
      │  └── http send               │
      └──────────────────────────────┘
                    │ trace_id
                    ▼
      logs:  {"event":"pending_issued", "request_id":"abc123",
              "trace_id":"fe27…", "span_id":"d60e…"}

      …and in reverse: a log line's trace_id opens the trace,
         whose app.request_id is the id the caller can quote.
```

#### The pivot the backend choice was made for

```
   Grafana: iam_login_attempts_total{outcome="invalid_credentials"}

        │                    ╱▔▔▔▔╲          ← spike
        │        ___________╱      ╲______
        └──────────────────●─────────────────  ● = exemplar (carries trace_id)
                           │
                     click ▼
              the trace of one failing request
```

#### Enable / disable

```
 IAM_OTEL_ENABLED ?
   │
   ├── false (dev, CI) ──▶ no provider installed
   │                       trace.get_tracer()  → NoOpTracer
   │                       metrics.get_meter() → NoOpMeter
   │                       @traced and .add()  → a function call, nothing more
   │
   └── true ─────────────▶ TracerProvider + MeterProvider (OTLP)
                           SQLAlchemy / Redis / FastAPI instrumented
                           OTEL_* env vars read natively by the SDK
```

### c.c) Subs and classes

**`src/anudeep/iam/observability.py`** — new, the only SDK importer.

```python
SERVICE_NAME    = "iam.anudeep.pro"
tracer, meter                                   # rebindable module globals

_request_id: ContextVar[str | None]
bind_request_id(request_id) -> None

FORBIDDEN_ATTRIBUTE_HINTS = {"email","password","token","handle",
                             "code","secret","ip","user_agent"}
_check_attributes(attrs)  -> None               # raises ValueError; the choke point

class _Instrument:                              # a counter that validates before recording
    __init__(name, description, unit="1")
    add(amount=1, /, **attributes)

traced(name)              -> decorator          # span, records exception, stamps request id
annotate(**attributes)    -> None               # validated span attributes
record_hash_duration(started)
current_trace_context()   -> dict[str, str]     # {"trace_id","span_id"} or {}

configure_telemetry(settings) -> bool           # no-op and False when disabled
_instrument_libraries()                         # SQLAlchemy (sync_engine) + Redis
instrument_app(app)                             # FastAPI, healthz/readyz excluded
```

**Instruments**

| Instrument | Attributes |
|---|---|
| `iam.login.attempts` | `outcome`: `totp_required` \| `totp_enrolment_required` \| `invalid_credentials` \| `rate_limited` |
| `iam.login.completed` | `method`: `totp` \| `recovery` |
| `iam.registration.attempts` | `outcome`: `created` \| `closed` \| `conflict` \| `weak_password` |
| `iam.enrolment` | `stage`: `started` \| `confirmed` \| `wrong_code` |
| `iam.recovery.attempts` | `outcome`: `ok` \| `invalid` \| `before_enrolment` |
| `iam.session.issued` | — |
| `iam.session.revoked` | `scope`: `one` \| `all` |
| `iam.session.resolved` | `result`: `ok` \| `unknown` \| `fingerprint_mismatch` \| `expired` \| `epoch_stale` |
| `iam.pending_handle` | `event`, `purpose` |
| `iam.introspection` | `result`: `active` \| `inactive` \| `stale` |
| `iam.rate_limit.tripped` | `scope`: `account` \| `ip` |
| `iam.password.changed` | — |
| `iam.password.hash.duration` | histogram, milliseconds |

**Spans** — `@traced` on `register`, `login`, `complete_totp`, `login_with_recovery_code`,
`begin_totp_enrolment`, `confirm_totp_enrolment`, `change_password`, `logout_everywhere`,
`introspect`, `issue_session`, `resolve_session`, `_issue_pending`, `_enforce_rate_limit`.

**Other modules**

| File | Change |
|---|---|
| `logging.py` | `add_trace_context` processor; middleware calls `bind_request_id` |
| `main.py` | `configure_telemetry` then `instrument_app` in `create_app` |
| `services/auth.py` | decorators + counter increments beside each existing `log.*` |
| `services/session.py`, `pending.py` | `SESSION_REVOKED`, `PENDING_HANDLE` |
| `routers/validate.py` | `INTROSPECTION` by result |
| `settings.py` | `otel_enabled`, `otel_console_export` |
| `docker-compose.yml`, `observability/*` | collector, Tempo, Prometheus, Grafana under a profile |

### c.d) The attribute rule

Enforced in code, not by convention:

```python
def _check_attributes(attrs):
    for key in attrs:
        if any(hint in key.lower() for hint in FORBIDDEN_ATTRIBUTE_HINTS):
            raise ValueError(...)
```

A **name** check, not a value check — values are not reliably recognisable, but nobody writes a
session token under a key not called something like `token`. It **raises** rather than dropping,
because a silent drop means believing in a dimension that is not there, and it fires in the test
suite, which is where it is meant to fire.

Deliberately narrower than the logs, which do record email and address. A log line is read by a
person chasing one incident and then rotates away; a metric label is stored indefinitely and
multiplied by every other label, so one label per email address is both a leak and an unbounded
time series.

---

## c.e) Verification performed

Not simulated — run against the real stack.

| Check | Result |
|---|---|
| Full suite, telemetry off | **175 pass**, ruff clean |
| Telemetry enabled live | `telemetry_enabled` logged; app healthy |
| Counters in Prometheus | `iam_login_attempts_total{outcome="invalid_credentials"} = 2`, `{outcome="totp_enrolment_required"} = 1`, `iam_registration_attempts_total{outcome="created"} = 1` — matching the traffic driven exactly |
| Histogram | mean argon2 verify **47.3 ms** |
| Traces in Tempo | 4 traces; a login leg is **16 spans** — HTTP → `iam.login` → `iam.rate_limit.check` → `iam.pending.issue` → 5×`SELECT` → `INSERT` → `HSET EXPIRE` |
| Search by request id | `{ span.app.request_id = "trace-me-please" }` → 1 match |
| Log correlation | one line carrying `request_id`, `trace_id` and `span_id` together |
| Label hygiene | no credential-shaped label name reached Prometheus |

**Two defects found and fixed during verification.**

1. Stamping `app.request_id` onto "the current span" in middleware recorded nothing, because
   with the FastAPI instrumentor disabled there is no span to stamp. Replaced with a contextvar
   read by `traced`, so every service span carries it whether or not HTTP is instrumented.
2. Tempo's port 3200 was never published, so traces could be queried only through Grafana.
   Published it — "did anything arrive at all" should be answerable with `curl`.

---

## c.f) Known limits

- **Metrics take up to 60 s to appear** — the SDK's default export interval. Not a fault, but
  it is why a freshly driven counter looks missing.
- **Traces take ~20 s to become searchable** in Tempo after ingest. A trace fetched by id is
  available sooner than one found by TraceQL.
- **Logs are not queryable beside traces** without adding Loki or a shipper. That is the
  accepted cost of Option 3; the trace id is present, so adding Loki later needs no code change.
- **Sampling is always-on.** Correct at this volume; a busier service would set
  `OTEL_TRACES_SAMPLER` rather than change code.
