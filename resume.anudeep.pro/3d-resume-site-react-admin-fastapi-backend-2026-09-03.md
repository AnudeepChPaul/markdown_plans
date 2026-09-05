# resume.anudeep.pro — 3D Résumé Site, React Admin & FastAPI Content Backend

**Date:** 2026-09-03
**Repos (new):** `resume.anudeep.pro`, `admin.anudeep.pro`, `api.anudeep.pro`
**Repo (existing, to be retired):** `ext.dashboard.dnudeep.dev`
**Status:** Plan — not yet implemented

---

## Context

There is an existing admin back-office for `dnudeep.dev` — `ext.dashboard.dnudeep.dev`, a Sencha
Ext JS 7.3.0.55 app — but **there is no public résumé site anywhere**, and the admin is only
partly functional:

| Concern | Today |
|---|---|
| Public site | Does not exist in any repo |
| Backend | Flask RPC at `http://127.0.0.1:5000/resume/api/<service>/{get_x\|update\|delete}`, envelopes `{"technologies":[…]}` / `{"projects":[…]}`, **no auth** (`config.js`) |
| Skills | `app/model/Skill.js` — `name, rating, visible, techId, url`. Works, REST-backed |
| Projects | `app/model/Project.js` — `projectName, tagLine, description, role, skills[], visible`. Works, REST-backed |
| Experiences | `app/model/Experience.js` — **`localstorage` proxy only**, never persisted server-side |
| Bio / intro | Form in `classic/src/view/configurations/Configurations.js`; `ConfigController.saveIntroduction` calls `getForm().submit()` on a form with **no `url` and no `api`** — intro copy has never been saved |
| Markdown | **None.** Froala is vendored in `packages/` but unreferenced (`app.json requires: ["font-awesome","ux","charts"]`) |
| Publish step | **None.** `autoSync: true` writes every row edit straight to the live API |
| Latent bug | Project→Skill refs use `techId` (correct); Experience→Project refs use `id` while `Project.idProperty` is `projectId` |
| Vendored weight | `ext/` 138 MB + `packages/` ~26 MB against ~1,814 LOC of first-party code |

The goal is a **fast, animated, 3D-backed public résumé** whose content is authored as markdown in
an admin portal, pre-compiled into a static build. Because content is baked in at build time,
**visitors never touch the API** — which removes API latency, uptime and cold starts from the
critical path entirely, and removes the need for the GraphQL layer that was under consideration.

---

## a) What am I trying to do

Build three cooperating systems:

1. **`resume.anudeep.pro`** — a statically-built résumé with a cursor-reactive 3D background and
   section transitions, holding a sub-second LCP on mid-range mobile *despite* the WebGL scene.
   Sections: Professional Experience, Projects, Technologies (with ratings), Bio.
2. **`admin.anudeep.pro`** — a single-user React portal where markdown files for each content type
   are uploaded/edited, validated, previewed, and **published**.
3. **`api.anudeep.pro`** — FastAPI + Postgres holding raw markdown, rendered HTML, draft/published
   versions and history; exposes one build-time bundle endpoint and fires the deploy hook.

**Non-goals:** multi-user/roles, i18n, comments, analytics beyond Cloudflare's built-in, blog.

### Data flow

```
  ┌──────────────────────┐  markdown upload / edit
  │ admin.anudeep.pro    │───────────────────────────┐
  │ Vite + React + TSQ   │  POST /content/{type}     │
  └──────────────────────┘                           ▼
             ▲                            ┌────────────────────────┐
             │ drafts, validation errors  │  resume-api (FastAPI)  │
             └────────────────────────────│  Postgres              │
                                          │  frontmatter → HTML    │
                          "Publish" ─────▶│  snapshot + version    │
                                          └───────────┬────────────┘
                                                      │ POST deploy hook
                                                      ▼
                                          ┌────────────────────────┐
                                 build ──▶│ Cloudflare Pages build │
                          GET /site-bundle│   astro build          │
                          (ETag, token)   └───────────┬────────────┘
                                                      │ static assets
                                                      ▼
                                          ┌────────────────────────┐
                                          │ resume.anudeep.pro     │
   visitor ────────────────────────────▶  │ HTML + CSS + islands   │
   (never reaches the API)                └────────────────────────┘
```

---

## b) Options considered

### b.1 Résumé site rendering

| Option | Pros | Cons |
|---|---|---|
| **Astro 5 + React islands** ✅ | Text sections ship **0 KB JS**; LCP paints before any JS runs. Content Layer loader = the exact build-time-fetch pipeline needed, zod-typed and cached. `<Image>` optimizes at build with sharp. View Transitions built in. Markdown pipeline (remark/rehype/Shiki) included. Astro 5 builds MD collections ~5× faster, 25–50% less memory | Second syntax (`.astro`); React components can't be shared with the admin without extracting a package |
| Next.js 16 `output: 'export'` | One framework everywhere; components shareable with the admin; ISR available if the site ever goes dynamic | Boots React + hydrates the whole tree (~90 KB gz) for a document that has one interactive component. **`output: 'export'` disables the image optimizer** — `unoptimized: true` (no resize/WebP, no per-image escape hatch) or a custom CDN loader |
| Vite + React SPA + prerender | Simplest mental model, one bundle | Client-rendered shell; worst first paint — the opposite of the requirement |

**Chosen: Astro.** The deciding factor is not the 3D bundle (`three` + `@react-three/fiber` +
`drei` ≈ 160–180 KB gz dominates under *any* framework and is deferred under *any* framework) — it
is what the rest of the page costs while that scene loads. On Astro the résumé is complete,
readable and interactive-enough as pure HTML if the scene never loads at all.

### b.2 Admin portal

| Option | Pros | Cons |
|---|---|---|
| **New Vite + React + TS** ✅ | Drops a 138 MB commercially-licensed vendored framework; markdown editor ecosystem (CodeMirror 6 / Milkdown) is React-native; publish/diff/preview UX is the whole product and needs iteration | Rebuilds screens that partly exist |
| Extend the Ext JS dashboard | Reuses grids, forms, the 10-point rating picker | Sencha licence dependency; the `modern` profile does not build (`modern/src/view/main/List.js` requires a non-existent `Dashboard.store.Personnel`); bio + Experience persistence are broken and need writing anyway |
| React admin, retire Ext gradually | Keeps Kanban/inbox alive | Two admins, two auth stories, for screens that aren't part of the résumé |

**Chosen: new React admin.** The Ext app is retired once parity is reached; its Kanban/notifications/
inbox screens are demo scaffolding (`TaskPanel.js` loads hardcoded random rows) and are not ported.

### b.3 Publish pipeline

| Option | Pros | Cons |
|---|---|---|
| **Postgres + deploy webhook** ✅ | Draft/published split, preview builds from drafts, version history and true rollback, one transactional source of truth | Needs a DB and a bundle endpoint |
| Git-as-CMS | No DB; free diffs and history; editable outside the admin | Admin coupled to GitHub availability; slow writes; no transactional draft state |
| Runtime fetch, no rebuild | Instant updates | Puts API latency and uptime back on the visitor path — defeats the premise |

### b.4 Backend hosting

| Option | Pros | Cons |
|---|---|---|
| **Railway Hobby ~$5/mo** ✅ | Always warm (no cold start when opening the admin), managed Postgres on the same project, Singapore region on Metal, near-zero ops | $5/mo; 4 regions only |
| Hetzner CX22 (~€4/mo) | Cheapest always-on; full control; Postgres on the same box | You own updates, TLS, backups, monitoring. No India region (Falkenstein / Ashburn / Singapore) |
| Free tier (Koyeb / Render) + Neon | ₹0 | 5–15 s cold start on every admin session; Koyeb's free instance sleeps after 1 h idle and is Frankfurt/DC only |
| Oracle Always Free ARM (Mumbai) | Free forever, 4 vCPU / 24 GB, genuinely close to India | Painful signup, ARM capacity hunting, full self-managed ops |

**Chosen: Railway Hobby**, with Hetzner as the documented fallback if the bill or the region ever
matters. The résumé site itself is on **Cloudflare Pages free** — 500 builds/month is far beyond a
résumé's publish rate, and deploy hooks are supported.

---

## c) Selected design

### c.1 Repository layout

```
resume.anudeep.pro/          Astro 5, Cloudflare Pages
  src/
    loaders/resumeApi.ts     Content Layer loader → GET /api/v1/site-bundle
    content.config.ts        defineCollection + zod schemas
    components/
      three/Scene.tsx        React island, client:only="react"
      three/Field.tsx        instanced/shader background
      three/usePointer.ts    damped cursor → uniform
      sections/*.astro       Experience, Projects, Tech, Bio — 0 KB JS
      motion/Reveal.tsx      scroll-reveal island (client:visible)
    pages/index.astro
    styles/tokens.css
  astro.config.mjs

admin.anudeep.pro/           Vite + React + TS + TanStack Query
  src/
    api/client.ts            fetch wrapper, cookie auth, error envelope
    features/{experiences,projects,technologies,bio}/
    features/publish/        diff view, publish button, deploy status poll
    components/MarkdownEditor.tsx   CodeMirror 6 + live preview
    components/DropZone.tsx         single file or multi-file/zip drop

api.anudeep.pro/                  FastAPI + Postgres + Alembic
  app/  (see c.4)
  alembic/
  Dockerfile
```

### c.2 Résumé site — how "3D background + blazing fast" is actually achieved

The whole perf argument rests on the scene being **strictly optional to the document**.

```
index.astro  ──build──▶  static HTML  (experience, projects, tech, bio inlined)
                              │
                              ├── critical CSS inlined, fonts preloaded, subset woff2
                              ├── LCP = hero heading text     ← paints with ZERO JS
                              │
                              └── <Scene client:only="react" />   ~170 KB gz, loaded LAST
                                       │
                                       ├─ gate: prefers-reduced-motion  → skip entirely
                                       ├─ gate: navigator.deviceMemory < 4
                                       │        || hardwareConcurrency <= 4  → static poster
                                       ├─ gate: no WebGL2 context        → static poster
                                       └─ IntersectionObserver + document.hidden → pause loop
```

Concrete rules:

- **`client:only="react"`** on the canvas. WebGL cannot server-render under any framework, so
  there is no SSR pass to waste; this is the same as `dynamic(…, {ssr:false})` in Next, one
  directive instead of a wrapper.
- **Geometry, not models.** An instanced particle field or a full-screen shader plane driven by
  the pointer — not a GLTF. A GLTF adds a loader, a parse cost, a decoder (draco/meshopt) and a
  network round-trip for something that is decoration. If a model is later wanted, it is
  draco-compressed, `useGLTF.preload`-ed only after the `load` event, and still gated as above.
- **Cursor tracking is damped, not raw.** `usePointer.ts` writes normalized pointer to a ref;
  `useFrame` lerps a uniform (`MathUtils.damp`) toward it. Never `setState` per pointer move —
  that would re-render React 120×/s.
- **Adaptive quality.** `<Canvas dpr={[1, 1.5]} gl={{ antialias: false, powerPreference: 'high-performance' }} performance={{ min: 0.5 }}>`
  wrapped in drei's `<PerformanceMonitor>` with hysteresis bounds — `onDecline` drops dpr and
  particle count, `onIncline` restores. Verified with `r3f-perf` in dev only.
- **Section animations are CSS-first.** Scroll-driven animations and Astro View Transitions handle
  reveals and section morphs; a React motion island is used only where CSS genuinely can't.
- **Budget, enforced in CI:** LCP < 1.2 s and TBT < 150 ms on a Moto G4 profile *with* the scene
  active; the no-JS render must score ≥ 98 on Lighthouse. Lighthouse CI runs on every Pages build
  and fails the build on regression.

### c.3 Content: markdown front-matter contracts

Relations are expressed as **human-writable slugs**, not UUIDs — this is what fixes the existing
`techId` vs `id` vs `projectId` mismatch, since a slug is the same token everywhere.

```yaml
# technologies/typescript.md
slug: typescript          # required, unique, [a-z0-9-]
name: TypeScript          # required
rating: 9                 # required, 1..10  (carried over from Ext's rating picker)
category: language        # language | framework | infra | tooling | database
url: https://…            # optional
visible: true             # default true
order: 10                 # optional manual sort within category
---
Optional markdown blurb.
```

```yaml
# experiences/acme-2021.md
slug: acme-2021
company: Acme Corp
designation: Senior Engineer
start: 2021-04-01         # real dates replace the old free-text `duration`
end: null                 # null ⇒ "Present"
url: https://acme.example
projects: [payments-api, ledger-ui]   # slugs → projects
visible: true
---
Markdown body: what you did, achievements.
```

```yaml
# projects/payments-api.md
slug: payments-api
name: Payments API
tagline: One-line hook
role: Tech lead
skills: [typescript, postgres]        # slugs → technologies
repo / demo / cover: optional
visible: true
featured: false
---
Markdown body: the long description (was a plain textarea in Ext).
```

```yaml
# bio/index.md   (singleton — replaces the never-persisted Introductions form)
pageTitle, headline, tagLines[], experienceTagLine, livingInTagLine, avatar, socials[]
---
Markdown body: the bio prose.
```

### c.4 Backend — core decisions

| Decision | Choice | Why |
|---|---|---|
| Item identity | **`slug`** is the human/API key; `uuid` PK stays internal | This is what kills the `techId` / `projectId` / `id` mismatch — a slug is the same token in every file and every table |
| Typed columns vs JSONB | **JSONB `front_matter` validated by Pydantic**, plus promoted columns (`slug, title, sort_order, visible, starts_on, ends_on`) | Pydantic is the real schema; Postgres enforces identity, relations and ordering. Avoids a migration per front-matter field |
| Markdown → HTML | **At ingest**, stored on the version row, with `renderer` recorded | Author sees the render error attributed to the exact file in the same request; publish stays a pointer move; Astro's build stays dumb and deterministic. Re-rendering after a plugin change is an explicit backfill job producing new versions |
| Markdown lib | **markdown-it-py** + `mdit-py-plugins` | CommonMark-strict, so output matches what remark would do; stable token stream for asset-URL rewriting. `mistune` is ~2× faster but not CommonMark and thinner on plugins — throughput is irrelevant for tens of files at ingest |
| Sanitizer | **`nh3`** (Rust/ammonia binding) | `bleach` is unmaintained as of 2026-06 with no further security releases. `html=False` on MarkdownIt too — belt and braces |
| Draft/publish | Append-only `content_version`; `content_item` holds `draft_version_id` + `published_version_id` | History for free; publish and rollback are both pointer moves |
| Bundle | Immutable `bundle_version` row: entire site payload as JSONB + precomputed ETag | Build-time fetch is one indexed row read; a build is reproducible even if content changes mid-build |
| Auth | argon2id + JWT in httpOnly cookie (admin); static bearer token (build) | GitHub OAuth needs an app, a callback, state/PKCE — and *still* ends in a local session cookie. Strictly more surface for one user |

### c.5 Postgres schema

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS citext;

CREATE TYPE content_type  AS ENUM ('bio','technology','project','experience');
CREATE TYPE relation_kind AS ENUM ('project_skill','experience_project','experience_skill');
CREATE TYPE deploy_status AS ENUM ('pending','triggered','failed','succeeded','unknown');
CREATE TYPE ingest_status AS ENUM ('pending','ok','failed','partial');
CREATE TYPE asset_kind    AS ENUM ('image','document');

CREATE TABLE admin_user (
    id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    email         citext NOT NULL UNIQUE,
    password_hash text   NOT NULL,               -- argon2id PHC string
    totp_secret   text,
    token_epoch   integer NOT NULL DEFAULT 0,    -- bump ⇒ all JWTs invalid
    last_login_at timestamptz,
    created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE content_item (
    id                   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    type                 content_type NOT NULL,
    slug                 text NOT NULL
                         CHECK (slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$' AND length(slug) <= 64),
    draft_version_id     uuid,        -- FK added after content_version (circular)
    published_version_id uuid,        -- NULL ⇒ never published
    archived_at          timestamptz, -- soft delete; excluded from bundles
    created_at           timestamptz NOT NULL DEFAULT now(),
    updated_at           timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT content_item_slug_uniq UNIQUE (type, slug)
);
CREATE UNIQUE INDEX content_item_bio_singleton
    ON content_item ((true)) WHERE type = 'bio' AND archived_at IS NULL;
CREATE INDEX content_item_type_idx ON content_item (type) WHERE archived_at IS NULL;

CREATE TABLE content_version (
    id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id         uuid NOT NULL REFERENCES content_item(id) ON DELETE CASCADE,
    revision        integer NOT NULL,          -- 1..n per item
    type            content_type NOT NULL,
    slug            text NOT NULL,             -- snapshot at this revision
    raw_markdown    text NOT NULL,             -- full file incl. front-matter
    body_markdown   text NOT NULL,
    body_html       text NOT NULL,             -- rendered + sanitized
    front_matter    jsonb NOT NULL,            -- validated model_dump()
    title           text,
    visible         boolean NOT NULL DEFAULT true,
    sort_order      integer NOT NULL DEFAULT 1000,
    starts_on       date,
    ends_on         date,                      -- NULL ⇒ present
    content_hash    char(64) NOT NULL,         -- sha256(raw_markdown), CRLF-normalised
    renderer        text NOT NULL,             -- 'markdown-it-py@3.0.0+nh3@0.2.18+v1'
    source_filename text,
    ingest_file_id  uuid,
    created_at      timestamptz NOT NULL DEFAULT now(),
    created_by      uuid REFERENCES admin_user(id),
    CONSTRAINT content_version_rev_uniq UNIQUE (item_id, revision),
    CONSTRAINT content_version_dates_ck
        CHECK (ends_on IS NULL OR starts_on IS NULL OR ends_on >= starts_on)
);
CREATE INDEX content_version_item_idx ON content_version (item_id, revision DESC);
CREATE INDEX content_version_hash_idx ON content_version (item_id, content_hash);

ALTER TABLE content_item
  ADD CONSTRAINT content_item_draft_fk     FOREIGN KEY (draft_version_id)
      REFERENCES content_version(id) ON DELETE SET NULL,
  ADD CONSTRAINT content_item_published_fk FOREIGN KEY (published_version_id)
      REFERENCES content_version(id) ON DELETE SET NULL;

-- Version-scoped: a draft can change its relations without touching what is live.
CREATE TABLE content_relation (
    id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    from_version_id uuid NOT NULL REFERENCES content_version(id) ON DELETE CASCADE,
    kind            relation_kind NOT NULL,
    target_type     content_type NOT NULL,
    target_slug     text NOT NULL,
    target_item_id  uuid REFERENCES content_item(id) ON DELETE SET NULL,  -- NULL ⇒ dangling
    ordinal         integer NOT NULL,          -- list order == display order
    CONSTRAINT content_relation_uniq UNIQUE (from_version_id, kind, target_slug)
);
CREATE INDEX content_relation_target_idx ON content_relation (target_type, target_slug);
CREATE INDEX content_relation_dangling_idx
    ON content_relation (from_version_id) WHERE target_item_id IS NULL;

CREATE TABLE asset (
    id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    key          text NOT NULL UNIQUE,        -- 'assets/aurora-cover.png'
    kind         asset_kind NOT NULL,
    content_type text NOT NULL,
    byte_size    bigint NOT NULL,
    checksum     char(64) NOT NULL,
    width        integer, height integer, blurhash text, alt_text text,
    storage_url  text NOT NULL,               -- content-addressed R2/CDN URL
    created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE ingest_batch (
    id     uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    status ingest_status NOT NULL DEFAULT 'pending',
    file_count integer NOT NULL DEFAULT 0,
    ok_count   integer NOT NULL DEFAULT 0,
    error_count integer NOT NULL DEFAULT 0,
    dry_run    boolean NOT NULL DEFAULT false,
    created_by uuid REFERENCES admin_user(id),
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE ingest_file (
    id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    batch_id      uuid NOT NULL REFERENCES ingest_batch(id) ON DELETE CASCADE,
    filename      text NOT NULL,              -- path inside the zip
    byte_size     integer NOT NULL,
    status        ingest_status NOT NULL,
    detected_type content_type,
    slug          text,
    version_id    uuid REFERENCES content_version(id) ON DELETE SET NULL,
    errors        jsonb NOT NULL DEFAULT '[]'::jsonb,
    warnings      jsonb NOT NULL DEFAULT '[]'::jsonb
);
CREATE INDEX ingest_file_batch_idx ON ingest_file (batch_id);

ALTER TABLE content_version ADD CONSTRAINT content_version_ingest_fk
    FOREIGN KEY (ingest_file_id) REFERENCES ingest_file(id) ON DELETE SET NULL;

CREATE TABLE bundle_version (
    id       uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    number   bigint GENERATED ALWAYS AS IDENTITY,   -- human-facing build number
    payload  jsonb NOT NULL,                        -- the complete SiteBundle document
    etag     char(64) NOT NULL,                     -- sha256 of canonical JSON
    schema_version integer NOT NULL,
    item_count integer NOT NULL,
    version_ids uuid[] NOT NULL,                    -- provenance, drives rollback
    note text,
    rolled_back_from uuid REFERENCES bundle_version(id),
    created_by uuid REFERENCES admin_user(id),
    created_at timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX bundle_version_number_idx ON bundle_version (number);

CREATE TABLE site_state (          -- exactly one row: the "what is live" pointer
    id                boolean PRIMARY KEY DEFAULT true CHECK (id),
    current_bundle_id uuid REFERENCES bundle_version(id),
    updated_at        timestamptz NOT NULL DEFAULT now()
);
INSERT INTO site_state (id) VALUES (true) ON CONFLICT DO NOTHING;

CREATE TABLE deployment (
    id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    bundle_id        uuid NOT NULL REFERENCES bundle_version(id) ON DELETE CASCADE,
    status           deploy_status NOT NULL DEFAULT 'pending',
    hook_url_label   text NOT NULL,             -- 'production' | 'preview'
    hook_http_status integer,
    hook_response    jsonb,
    cf_deployment_id text,
    attempt          integer NOT NULL DEFAULT 1,
    error            text,
    triggered_at timestamptz, settled_at timestamptz,
    created_at   timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX deployment_active_idx ON deployment (status)
    WHERE status IN ('pending','triggered');

-- refresh-token rotation with reuse detection; a replayed jti bumps token_epoch
CREATE TABLE refresh_token (
    jti         uuid PRIMARY KEY,
    user_id     uuid NOT NULL REFERENCES admin_user(id) ON DELETE CASCADE,
    parent_jti  uuid REFERENCES refresh_token(jti),
    used_at     timestamptz,          -- non-NULL + presented again ⇒ replay
    expires_at  timestamptz NOT NULL,
    user_agent  text, ip inet,
    created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX refresh_token_user_idx ON refresh_token (user_id, expires_at DESC);

CREATE TABLE recovery_code (
    id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    uuid NOT NULL REFERENCES admin_user(id) ON DELETE CASCADE,
    code_hash  text NOT NULL,         -- argon2id, single use
    used_at    timestamptz,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE login_attempt (        -- rate limiting + lockout + alerting
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email      citext, ip inet NOT NULL,
    success    boolean NOT NULL,
    user_agent text,
    created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX login_attempt_ip_idx    ON login_attempt (ip, created_at DESC);
CREATE INDEX login_attempt_email_idx ON login_attempt (email, created_at DESC);

CREATE TABLE audit_log (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    actor_id   uuid REFERENCES admin_user(id),
    action     text NOT NULL,        -- 'content.ingest' | 'content.publish' | 'bundle.rollback'
    subject    text,                 -- 'project:aurora-billing' | 'bundle:23'
    detail     jsonb NOT NULL DEFAULT '{}'::jsonb,
    ip         inet,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

Publish takes `pg_advisory_xact_lock(hashtext('resume:publish'))` — cheap insurance against a
double-clicked Publish button and against the second uvicorn worker.

### c.6 Ingestion pipeline

```
POST /api/v1/ingest   (multipart: files[] | archive.zip, ?dry_run=)
  IngestService.ingest_upload()
    1. unpack     ZipExtractor — reject >20 MB total, >200 entries, path traversal,
                  symlinks, non-allowlisted extensions. Stream to temp dir, never extractall
    2. parse      python-frontmatter.loads()          → FRONTMATTER_PARSE
    3. classify   fm["type"] or infer from directory  → TYPE_UNKNOWN
    4. validate   FRONT_MATTER_MODELS[type].model_validate()   (Pydantic v2)
    5. render     MarkdownRenderer.render(body, assets)
    6. sanitize   nh3.clean(html, allowlist, url_schemes={http,https,mailto})
    7. stage      StagedItem(type, slug, fm, body, html, relations)
    ── all files staged ──
    8. resolve    RelationResolver over (staged ∪ existing)  → dangling warnings
    9. commit     per-file atomicity, not per-batch: OK files land, failures reported
   10. write      content_version rows + move content_item.draft_version_id
```

Steps 1–8 are pure functions; step 10 is one transaction. `dry_run=true` runs 1–8 and returns the
identical report with `applied: false` — that is the admin's "validate before upload" affordance.

Renderer: `MarkdownIt("commonmark", {"html": False}).enable(["table","strikethrough","linkify"])`
plus `attrs_plugin`, `anchors_plugin`, `footnote_plugin`, `tasklists_plugin`. Token-stream passes
rewrite `^assets/` image srcs to `asset.storage_url` (unknown key ⇒ warning, not failure) and add
`rel="noopener noreferrer"` to external links. Inline fields that want emphasis (`tagline`,
`highlights[]`) go through `render_inline()` into `*_html` keys.

Error contract — one renderer serves both ingest and inline edits in the admin:

```python
class IngestError(BaseModel):
    code: IngestErrorCode   # FRONTMATTER_PARSE | SCHEMA_INVALID | SLUG_DUPLICATE_IN_BATCH
                            # | RELATION_UNRESOLVED | ASSET_MISSING | BIO_DUPLICATE | ...
    field: str | None       # dotted path: "skills.2", "starts_on"
    message: str
    line: int | None        # front-matter line when derivable
    hint: str | None        # "Known technology slugs: typescript, postgres, ..."

class IngestFileResult(BaseModel):
    filename: str
    status: Literal["ok", "failed", "skipped_unchanged"]   # unchanged ⇒ hash match, no churn
    type: ContentType | None; slug: str | None
    version_id: UUID | None; revision: int | None
    errors: list[IngestError]; warnings: list[IngestError]

class IngestReport(BaseModel):
    batch_id: UUID; applied: bool
    status: Literal["ok", "partial", "failed"]
    file_count: int; ok_count: int; error_count: int
    files: list[IngestFileResult]
    unresolved_slugs: list[UnresolvedRelation]
```

### c.7 Publish, bundle and deploy

```python
class PublishService:
    async def preflight(self) -> PublishPreflight: ...
    async def publish(self, *, actor: AdminUser, note: str | None,
                      item_ids: list[UUID] | None = None,
                      force: bool = False) -> PublishResult: ...
    async def rollback(self, *, actor: AdminUser, to_bundle_id: UUID) -> PublishResult: ...
```

`publish()`, one transaction under the advisory lock:

1. **Preflight** — 409 `PUBLISH_BLOCKED` unless `force`: dangling relations among the items being
   published, bio missing, zero visible projects, or a `pending`/`triggered` deployment younger
   than `deploy_stale_after_seconds` (900).
2. Move `published_version_id = draft_version_id` for every dirty item (or `item_ids`).
3. `BundleBuilder.build(preview=False)` materialises the whole `SiteBundle`.
4. Canonical JSON (`sort_keys=True, separators=(",",":")`) → sha256 → `etag`. **Equal to the
   current etag ⇒ short-circuit**: no bundle, no deploy, return `unchanged`.
5. Insert `bundle_version`, repoint `site_state`, insert `deployment(status='pending')`.
6. `COMMIT`, then **after commit** (`BackgroundTasks`) `DeployHookClient.trigger()` — httpx, 10 s
   timeout, 3 attempts, backoff 1s/4s/16s with jitter. A hook failure must never undo a publish.

Deploy status settles one of two ways: if `CF_API_TOKEN`/`CF_ACCOUNT_ID`/`CF_PROJECT_NAME` are set,
an APScheduler poller resolves `triggered → succeeded|failed` (20 s interval, 15 min ceiling);
otherwise the Astro build's `postbuild` script calls `POST /deployments/{id}/callback` with the
build token. Unsettled past the ceiling ⇒ `unknown`, and no longer blocking.

**Rollback is append-only:** copy bundle #N's payload/etag into a *new* `bundle_version`
(`rolled_back_from`, note `"rollback to #N"`), repoint `site_state`, and repoint each item's
`published_version_id` from that bundle's `version_ids` — items created after #N get `NULL` (they
leave the site but stay in the admin as drafts). Drafts are never touched, so in-progress edits
survive a rollback.

**`GET /api/v1/site-bundle`** — the single build-time endpoint:

- Published mode: one indexed row read via `site_state.current_bundle_id`. Sets
  `ETag`, `Cache-Control: public, max-age=60, stale-while-revalidate=600`, `X-Bundle-Number`.
  `If-None-Match` hit ⇒ `304`, no body.
- `?preview=1`: composed from **draft** versions on the fly, never persisted; ETag is the sha256 of
  sorted `(item_id, content_hash)` pairs so preview builds still get 304s. `Cache-Control: private,
  no-store`. Requires the `bundle:preview` scope. Adds `"mode": "preview"` and `"unresolved": [...]`
  so a preview build can render a warning banner.

Payload shape (abridged — the full JSON snapshot in `tests/` is the contract with the Astro loader):

```jsonc
{
  "schema_version": 1,
  "bundle": { "number": 23, "mode": "published", "etag": "9c1a…", "generated_at": "…" },
  "bio": { "name": "…", "page_title": "…", "headline": "…", "headline_html": "…",
           "taglines": ["…"], "experience_tagline": "…", "living_in_tagline": "…",
           "avatar": { "url": "…", "width": 512, "height": 512, "blurhash": "L6P…" },
           "socials": [{ "label": "GitHub", "url": "…", "icon": "github" }],
           "body_html": "<p>…</p>" },
  "technologies": [ { "slug": "typescript", "title": "TypeScript", "rating": 9,
                      "category": "language", "years": 7, "order": 10 } ],
  "projects":     [ { "slug": "aurora-billing", "tagline_html": "…", "role": "Tech Lead",
                      "skills": ["typescript","postgres"], "featured": true,
                      "cover": { "url": "…", "width": 1600, "height": 900, "blurhash": "…" },
                      "duration_label": "Apr 2023 — Present", "body_html": "…" } ],
  "experiences":  [ { "slug": "acme-corp-senior-engineer", "title": "Acme Corp",
                      "designation": "Senior Engineer", "is_current": false,
                      "duration_label": "Jun 2021 — Mar 2023 · 1 yr 10 mo", "duration_months": 22,
                      "projects": ["aurora-billing"], "skills": ["typescript","kafka"],
                      "highlights": [{ "text": "…", "html": "…" }], "body_html": "…" } ],
  "index": { "technologies_by_slug": { "typescript": 0 },
             "projects_by_slug": { "aurora-billing": 0 },
             "technologies_by_category": { "language": ["typescript"] } },
  "stats": { "years_experience": 9, "project_count": 6, "technology_count": 24 }
}
```

Three properties the site depends on: lists arrive **pre-filtered on `visible` and pre-sorted** by
`(sort_order, lower(title), slug)` so Astro does zero filtering or sorting *and* an unchanged
publish reliably reproduces its ETag; relations stay as **slug arrays** with `index.*_by_slug` for
O(1) lookup instead of duplicated embedded objects; and `duration_label` is computed server-side, so
the old free-text `duration` never comes back and the site formats no dates.

The Astro loader writes `.cache/site-bundle.json` on every successful fetch and **falls back to it
when the API is unreachable** — a Railway blip must not break a Cloudflare build.

### c.8 Auth

This is a real, public site on a real domain, so the admin is treated as an internet-exposed
control plane, not a hobby CRUD app. Origins:

| Host | Serves | Exposure |
|---|---|---|
| `resume.anudeep.pro` | Static Astro build, Cloudflare Pages | Public |
| `admin.anudeep.pro` | Admin SPA, Cloudflare Pages | **Cloudflare Access — never publicly reachable** |
| `api.anudeep.pro` | FastAPI on Railway, proxied through Cloudflare | Public origin, WAF-fronted |
| `cdn.anudeep.pro` | R2 assets | Public, **separate origin by design** |

#### c.8.1 Threat model

What an attacker actually gains, and what stops it:

| Threat | Consequence | Control |
|---|---|---|
| Credential stuffing / brute force on `/auth/login` | Full content control, site defacement | Cloudflare Access in front, argon2id, **mandatory** TOTP, per-IP + per-account rate limit, lockout, alert on failure streak |
| Stolen session cookie (XSS) | Same | `__Host-` prefixed HttpOnly cookie, strict CSP with nonces on the admin, no token ever in JS |
| CSRF from another `*.anudeep.pro` origin | Unauthorized publish/delete | Double-submit CSRF token **plus** an `Origin` header allowlist check server-side |
| Malicious upload (zip bomb, traversal, polyglot, SVG) | RCE, stored XSS, disk exhaustion | Magic-byte sniffing, hard caps, no `extractall`, **SVG rejected outright**, assets served from a separate origin |
| Stored XSS via markdown | Persistent XSS on the public résumé | `html=False` on MarkdownIt + `nh3` allowlist sanitize + preview rendered inside a sandboxed iframe |
| Leaked build token | Read the bundle (already-public content) | Read-only scope, rotatable, no write path — deliberately the least valuable secret in the system |
| Leaked CF API / R2 key | Deface site, delete assets | Scoped to one Pages project / one bucket; never in the repo |
| Compromised dependency | Everything | Lockfiles, Renovate, `pip-audit` + `npm audit` + Trivy in CI, no post-install scripts in the Astro build |
| Railway origin found directly | Bypasses the WAF | Origin locked to Cloudflare IPs + a shared origin-auth header |

#### c.8.2 Perimeter — the cheapest real win

**Put `admin.anudeep.pro` behind Cloudflare Access** (Zero Trust, free up to 50 users). A
one-user admin has no business being reachable by the internet at all: Access authenticates at
the edge with a GitHub/Google identity plus the device policy, and the SPA is never served to an
unauthenticated request. Application login sits *behind* that as the second factor of a
defence-in-depth stack, not the only gate.

Corollary for the API: the admin endpoints (`/api/v1/{content,ingest,assets,publish,bundles,
deployments,auth}`) get an Access **service-token** policy too, so unauthenticated traffic never
reaches FastAPI. `GET /api/v1/site-bundle` and `/healthz` stay outside that policy — the Cloudflare
Pages build runner is not an Access client. Proxy `api.anudeep.pro` (orange cloud) for WAF, bot
fight mode and rate limiting, and lock the Railway service to Cloudflare IP ranges plus a shared
`CF-Origin-Auth` header so nobody reaches the origin directly.

#### c.8.3 Session and credentials

- `argon2-cffi` `PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4)`, rehash-on-verify
  when parameters change. Password minimum 12 chars, checked against a breach list at set time.
- **TOTP is mandatory, not optional** (`pyotp`) — the plan's earlier "optional if `totp_secret IS
  NOT NULL`" is downgraded to: an account without `totp_secret` may only reach `/auth/enroll-totp`.
  Recovery codes are single-use argon2 hashes in a `recovery_code` table.
- Cookie: **`__Host-resume_session`** — the prefix forces `Secure`, `Path=/`, and **no `Domain`
  attribute**, so it is host-only to `api.anudeep.pro` and cannot be injected by any sibling
  subdomain. `HttpOnly; SameSite=Lax`. Because admin and API are both under `anudeep.pro`, Lax
  still sends the cookie on same-site XHR — a real benefit of subdomain layout over a split domain.
- Short-lived access claim (30 min) + a rotating refresh cookie (`__Host-resume_refresh`, 14 days,
  `Path=/api/v1/auth`). Refresh rotation with reuse detection: a replayed refresh `jti` bumps
  `token_epoch` and kills every session.
- JWT `{sub, epoch, iat, exp, jti}`, HS256, `SESSION_SECRET` ≥ 32 random bytes. `token_epoch` gives
  a one-query global logout and is the reuse-detection kill switch.
- CSRF: double-submit `X-CSRF-Token` vs the readable `__Host-resume_csrf` cookie **and** a
  server-side `Origin` allowlist check on every non-GET. Same-site subdomains mean `SameSite`
  alone is not a CSRF defence.
- Rate limit: 5 attempts / 15 min per IP *and* per account, constant-time failure path, exponential
  lockout after 10. Every login (success and failure) writes `audit_log` with IP and user agent; a
  failure streak fires a notification.

#### c.8.4 Input, upload and output

- **Uploads:** validate magic bytes with `python-magic`, never the extension. Allowlist
  `png/jpeg/webp/avif/pdf` — **SVG is rejected**, because a sanitized SVG is still a scriptable
  document and there is no upside for a résumé. Strip EXIF (Pillow). Cap 20 MB total / 200 entries
  / per-entry size / nesting depth; reject absolute paths, `..`, symlinks; stream to a temp dir,
  never `extractall`. Content-addressed keys (`assets/{sha256[:16]}{ext}`) so a filename can never
  be attacker-controlled.
- **Asset origin isolation:** assets are served from `cdn.anudeep.pro` with
  `X-Content-Type-Options: nosniff` and an explicit `Content-Type` — a stored payload therefore
  cannot execute in the admin or résumé origin.
- **Markdown:** `MarkdownIt(..., {"html": False})` so raw HTML never survives parsing, *then*
  `nh3.clean()` with a tag/attribute allowlist and `url_schemes={"http","https","mailto"}`. Two
  independent controls, deliberately.
- **Admin preview:** the server-rendered HTML is injected into a **sandboxed iframe**
  (`sandbox` with no `allow-scripts`, `srcdoc`), not `dangerouslySetInnerHTML`. If the sanitizer
  ever fails open, the blast radius is an inert iframe rather than the session cookie.
- **CSP** — admin: `default-src 'self'; script-src 'self' 'nonce-…'; style-src 'self' 'nonce-…';
  img-src 'self' https://cdn.anudeep.pro data:; connect-src 'self' https://api.anudeep.pro;
  frame-src 'self'; object-src 'none'; base-uri 'none'; form-action 'none';
  frame-ancestors 'none'`. Résumé site: same shape, `connect-src 'none'` (it talks to nothing at
  runtime), plus `worker-src 'self' blob:` if the WebGL scene ever moves to a worker. Both get
  HSTS with `includeSubDomains; preload`, `X-Content-Type-Options`, `Referrer-Policy:
  strict-origin-when-cross-origin`, `Permissions-Policy` denying camera/mic/geolocation.
- **CORS:** `allow_origins = settings.admin_origins` (`https://admin.anudeep.pro`, plus
  `http://localhost:5173` in dev only — never both in one environment), `allow_credentials=True`,
  `allow_headers=["Content-Type","X-CSRF-Token"]`,
  `expose_headers=["ETag","X-Bundle-Number","X-Request-ID"]`, `max_age=600`. Never
  `allow_origins=["*"]` with credentials. The Astro build is server-side and needs no CORS entry.
- `TrustedHostMiddleware` pinned to the real hosts; `MAX_UPLOAD_BYTES` enforced **before** reading
  the body; request-body size limit at the proxy too.

#### c.8.5 Secrets, least privilege, supply chain

- **Build token:** `RESUME_BUILD_TOKENS` env holds `{"id","name","hash","scopes"}` — sha256 of a
  `resb_<32B base64url>` secret, compared with `hmac.compare_digest`. In env, not the DB, so a
  build authenticates even mid-migration. Production Pages gets `bundle:read` only; `bundle:preview`
  is a separate token on the preview environment. Rotation is additive: add the new hash, update the
  Pages variable, remove the old.
  *Why the endpoint is not public:* the rendered content is public anyway, but an open bundle
  endpoint is an unauthenticated full-table read on a $5 box that also leaks publish cadence and
  not-yet-linked items. The router is shaped so exposing `GET /site-bundle/public` later is a
  decorator swap.
- Every credential is least-privilege: the CF API token is scoped to **Pages:Edit on one project**;
  the R2 key to **one bucket**; the Postgres role is **not** superuser and connects with
  `sslmode=require`; the deploy hook URL is treated as a secret (it triggers builds).
- Secrets live only in Railway/Cloudflare env, never in a repo. `gitleaks` runs as a pre-commit
  hook and in CI.
- Supply chain: `uv.lock` / `pnpm-lock.yaml` committed, Renovate for updates, `pip-audit` +
  `npm audit --audit-level=high` + Trivy on the container image in CI, and the Astro build runs
  with lifecycle scripts disabled. Container runs non-root on `python:3.13-slim`.
- **Backups:** nightly `pg_dump`, age-encrypted, to R2 with 30-day retention — plus a **restore
  drill** before go-live, because an untested backup is not a backup. `GET /ingest/export` is the
  second, human-readable escape hatch.
- **Audit:** `audit_log` already records actor, action, subject, IP for every ingest, publish and
  rollback. Structured JSON logs with a request id; never log cookie values, tokens or raw
  front-matter secrets.

### c.9 FastAPI app layout

```
api.anudeep.pro/src/resume_api/
  main.py settings.py db.py deps.py errors.py logging.py
  security/     passwords.py tokens.py csrf.py
  models/       base content bundle deploy asset user ingest      (SQLAlchemy 2.0, Mapped[…])
  schemas/      frontmatter.py  # FRONT_MATTER_MODELS: dict[ContentType, type[BaseFrontMatter]]
                content.py ingest.py bundle.py publish.py auth.py common.py
  repositories/ content_repo version_repo relation_repo bundle_repo deploy_repo asset_repo user_repo
  services/     markdown.py ingest.py relations.py content.py bundle.py publish.py
                deploy.py assets.py auth.py duration.py
  routers/      health auth content ingest assets publish bundle deployments
```

DI stays thin: `get_session()` yields an `AsyncSession` per request; `get_*_service()` composes
service ← repositories ← session. No container library; tests use `app.dependency_overrides`.

Errors are RFC 9457 Problem Details (`{type,title,status,detail,code,errors?,request_id}`);
`ValidationFailed` carries `list[IngestError]` so the admin has **one** error renderer.

Endpoint inventory (all `/api/v1`, session+CSRF unless noted):

| Method | Path | Notes |
|---|---|---|
| POST/GET | `auth/login`, `auth/logout`, `auth/me` | login is unauthenticated |
| GET | `content?type=&status=draft\|published\|dirty&q=` | list |
| GET/PUT | `content/{type}/{slug}/raw` | `text/markdown` — powers the editor round-trip |
| GET | `content/{type}/{slug}/versions`, `.../versions/{rev}/diff` | history, unified diff vs published |
| POST | `content/{type}/{slug}/revert`, `.../rename` | new draft from an old revision; slug rename |
| DELETE | `content/{type}/{slug}` | soft delete |
| POST/GET | `ingest?dry_run=`, `ingest/{batch_id}` | multipart; report replay |
| GET | `ingest/export` | zip of current drafts — round-trips back through ingest |
| POST/GET/DELETE | `assets`, `assets/{key}` | |
| GET/POST | `publish/preflight`, `publish` | `{note, item_ids?, force?}` |
| GET/POST | `bundles`, `bundles/{id}/rollback` | |
| GET/POST | `deployments`, `deployments/{id}/retry` | |
| POST | `deployments/{id}/callback` | **build token** — the build reports its own outcome |
| **GET** | **`site-bundle`** | **build token**, ETag, `?preview=1` |
| GET | `/healthz`, `/readyz` | unauthenticated |

**Deploy:** multi-stage `uv` Dockerfile on `python:3.13-slim`, non-root, uvicorn `--workers 2
--proxy-headers`. `railway.toml` runs `alembic upgrade head` as `preDeployCommand` — **not** in the
app lifespan, where restarts race and rollbacks get awkward. Managed Postgres in the same region;
daily `pg_dump` to R2, with `GET /ingest/export` as a second human-readable escape hatch.

### c.10 Admin portal

```
src/
  api/client.ts             fetch wrapper: credentials:'include', X-CSRF-Token, Problem→Error
  components/DropZone.tsx   folder / multi-file / zip drop → POST /ingest?dry_run=1 first
  components/IngestReport.tsx   per-file rows: filename, status, field-level errors with hints
  components/MarkdownEditor.tsx CodeMirror 6 (@codemirror/lang-markdown + lang-yaml for the
                                front-matter block) + debounced server-rendered preview
  features/{technologies,projects,experiences,bio}/   list + editor, TanStack Query
  features/publish/         preflight blockers, dirty-item diff, Publish, deploy status poll,
                            bundle history + rollback
```

CodeMirror 6 over Milkdown: the source of truth is a **markdown file with YAML front-matter**, and
CM6 edits exactly that with real YAML highlighting in the front-matter block. Milkdown/ProseMirror
is WYSIWYG over an AST — it would fight the front-matter and risk lossy round-trips. Preview comes
from the server renderer, so what the admin previews is byte-identical to what ships.

Upload flow is always dry-run first: drop → `POST /ingest?dry_run=1` → show the report → confirm →
`POST /ingest`. Publish is a separate, deliberate action with its own preflight screen.

---

## d) Implementation sequence

**Backend** (`api.anudeep.pro`) — unblocks everything else:

1. Skeleton: `pyproject` (uv, ruff, mypy strict), settings, db, `create_app`, `/healthz`,
   Dockerfile; deploy the empty app to Railway to prove the pipe.
2. Alembic revision 1 (all of §c.5). Seed the admin via `python -m resume_api.cli create-user`.
3. Auth and hardening (§c.8): argon2id, `__Host-` cookies, refresh rotation with reuse detection,
   mandatory TOTP enrolment, CSRF + `Origin` check, rate limit + lockout, CORS, security-headers
   middleware, `/auth/*`. **Cloudflare Access on `admin.anudeep.pro` and on the admin API paths is
   configured in this step, before any content endpoint exists.**
4. `MarkdownRenderer` + `schemas/frontmatter.py` + golden-file tests under
   `tests/fixtures/markdown/`. Pure, no DB — do this **before** the ingest plumbing.
5. `IngestService` + `RelationResolver` + `POST /ingest` (dry-run path first).
6. Content read/list/history/diff/revert endpoints.
7. `AssetService` + R2.
8. `BundleBuilder` + `GET /site-bundle` with ETag and preview. **Snapshot-test the payload — that
   snapshot is the contract with the Astro loader.**
9. `PublishService`, `DeployHookClient`, preflight, rollback, deployments.
10. **Migration script:** read the legacy Flask envelopes (`{"technologies":[…]}`,
    `{"projects":[…]}`) and an Ext localStorage export for experiences, emit markdown files, feed
    them through `POST /ingest?dry_run=1`. Legacy `techId`/`id` refs map to slugs
    (`slugify(name)`) here — **this is where the old ref bug dies**.

**Résumé site** (`resume.anudeep.pro`): scaffold Astro + `@astrojs/react` → Content Layer loader
against the snapshot fixture (no live API needed) → the four sections as `.astro`, styled, zero JS →
Lighthouse baseline → *then* the R3F scene as an island with all the gates from §c.2 → Lighthouse
again, compare.

**Admin** (`admin.anudeep.pro`): client + auth → DropZone + IngestReport → lists → editor →
publish/rollback screen.

**Retire** `ext.dashboard.dnudeep.dev` once parity is reached. Its Kanban/notifications/inbox
screens are demo scaffolding — `classic/src/view/tasks/TaskPanel.js` loads hardcoded random rows —
and are not ported.

---

## e) Verification

| Layer | How |
|---|---|
| Front-matter + renderer | Golden-file tests: `tests/fixtures/markdown/*.md` → expected HTML + expected `IngestReport`. Include the hostile cases: `<script>` in a body, `javascript:` href, missing asset key, bad enum, unresolved slug |
| Ingest | Integration test against a disposable Postgres (testcontainers): drop a zip of all four types, assert per-file statuses; re-drop unchanged ⇒ every file `skipped_unchanged` and **no new versions** |
| Publish/rollback | Publish → assert bundle #1 + `deployment(pending)`; publish again unchanged ⇒ `unchanged`, no new bundle; edit + publish ⇒ #2; rollback to #1 ⇒ #3 with #1's etag, drafts intact |
| Bundle contract | `GET /site-bundle` payload snapshot-tested; the **same** snapshot file is the Astro loader's dev fixture. If they diverge, both suites fail |
| ETag | Second request with `If-None-Match` ⇒ 304, empty body |
| Auth | Login → `__Host-` cookies set; no `X-CSRF-Token` ⇒ 403; forged `Origin` ⇒ 403; `token_epoch` bump ⇒ 401; **replayed refresh `jti` ⇒ 401 and every session killed**; account without `totp_secret` reaches nothing but `/auth/enroll-totp`; 6th login attempt in 15 min ⇒ 429; `site-bundle` without the build token ⇒ 401; `bundle:read`-only token with `?preview=1` ⇒ 403 |
| Upload hardening | Fixture suite must all be rejected: zip bomb, `../../etc/passwd` entry, symlink entry, 25 MB archive, 201 entries, a `.png` whose magic bytes are ELF, an `.svg`, a PDF with an embedded script. Assert nothing is written outside the temp dir |
| XSS | A body containing `<script>`, `<img onerror>`, `javascript:` and `<iframe>` survives ingest as inert text; assert on the stored `body_html`. Admin preview iframe has no `allow-scripts` |
| Headers / perimeter | `curl -I` both sites: CSP, HSTS `includeSubDomains; preload`, `nosniff`, `Referrer-Policy`, `Permissions-Policy` present. `admin.anudeep.pro` from a logged-out browser ⇒ Cloudflare Access login, **never the SPA**. The Railway origin URL directly ⇒ refused without the `CF-Origin-Auth` header |
| Backup | Restore the nightly dump into a scratch database and boot the API against it — done once before go-live, then quarterly |
| Résumé perf | Lighthouse CI on every Cloudflare Pages build, Moto G4 profile: **LCP < 1.2 s, TBT < 150 ms with the scene active**; build fails on regression. Separately assert the JS-disabled render still scores ≥ 98 |
| 3D gates | Manual matrix: `prefers-reduced-motion: reduce` ⇒ no canvas mounted at all; WebGL2 disabled ⇒ poster; background tab ⇒ `useFrame` loop paused. `r3f-perf` in dev only, never bundled |
| End-to-end | Edit a technology's rating in the admin → dry-run → ingest → preflight → Publish → deploy hook fires → Cloudflare build pulls the new bundle → the rating renders on `resume.anudeep.pro`, and `GET /deployments` shows `succeeded` |

---

## f) Open items

- **3D concept is deliberately unspecified** beyond "instanced particle field or shader plane, not
  a GLTF". Pick the visual after the site's palette exists; the perf gates in §c.2 are the
  constraint that concept must satisfy.
- **Apex `anudeep.pro`**: redirect to `resume.anudeep.pro`, or serve the résumé at the apex with
  `resume.` as an alias? Everything else in this plan is unaffected either way.
- **Cloudflare Access identity provider** for the admin — GitHub, Google, or one-time PIN to
  `me@anudeep.pro`. GitHub assumed here.
- **Old `dnudeep.dev`**: keep as a redirect to `anudeep.pro`, or retire the domain entirely.
- **R2 vs Railway volume** for assets — R2 assumed here (free egress, CDN-fronted); a volume is
  cheaper to wire but puts image bytes on the origin.
- On approval, this file is copied to
  `markdown_plans/resume.anudeep.pro/3d-resume-site-react-admin-fastapi-backend-2026-09-03.md`
  per the repo plan convention.

