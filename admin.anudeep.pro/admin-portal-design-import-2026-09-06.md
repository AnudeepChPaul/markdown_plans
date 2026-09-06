# Admin Portal — Claude Design import

Source: `claude.ai/design/p/679c7b3d-d325-4a2d-8683-c6cbbc7994f0` → `Admin Portal.dc.html`
Design system: `_ds/modernist-258ba3da` (Modernist — flat, zero-radius, Archivo, single red accent)
Date: 2026-09-06

## a. What am I trying to do

Bring the `Admin Portal.dc.html` artboard into `admin.anudeep.pro` (Preact + Vite + react-router
+ TanStack Query + GraphQL codegen) without breaking the route-chunk budgets the repo enforces
in `scripts/route-budget.mjs`.

The artboard is a single-file prototype: one `DCLogic` component, `sc-if` / `sc-for` template
directives, inline `style` strings, and hard-coded `APPS` / `SECTIONS` / `DOCS` / `EXPERIENCES`
/ `SESSIONS` / `MEDIA` fixtures. None of it is wired to an API. The real app already has the
GraphQL/REST plumbing. So the import is a **shape and system transfer**, not a code transfer.

## b. Options considered

### Option 1 — Restyle in place (SELECTED)

Map Modernist tokens onto the existing token layer, rewrite the chrome to the design's two-row
model, and rebuild the resume screens to the design's three-pane workspace, keeping every
existing hook (`useDraft`, `usePublish`, `useUpload`) untouched.

Pros
- Existing data layer, conflict resolution, autosave and idempotent publishing all survive.
- Route budgets stay meaningful — no chunk moves.
- Every number on screen stays real.

Cons
- The design's Identity app has no backend, so it cannot be delivered as designed.
- Some artboard sections (Technologies group editor) have no matching mutation yet.

```
 ┌──────────────── nav (apps, 2px ink rule) ─────────────────┐
 ├──────────── subnav slot (module-owned) ───────────────────┤
 │  rail 300px │  DocumentPane      │  preview pane          │
 ├──────────── PublishStrip (resume only) ───────────────────┤
```

### Option 2 — Port the artboard verbatim as a new route

Translate `sc-for`/`sc-if` into JSX one-for-one, keeping the fixtures, and reconcile later.

Pros
- Pixel-faithful to the artboard immediately, including Identity and the diff drawer.

Cons
- Ships fabricated users, sessions and audit rows into an admin portal — actively misleading.
- Duplicates the shell; two navigations to keep in sync.
- Inline style strings defeat the token layer and the CSS budget.

### Option 3 — Design-system only

Take `styles.css` tokens, leave all layout alone.

Pros / Cons
- Cheapest, but delivers none of the interaction model the artboard exists to specify.

## c. Selected option — detail

### c.a System design

| Layer | File | Note |
|---|---|---|
| Tokens | `src/styles/tokens.css` | Modernist verbatim + aliases for legacy var names |
| Components | `src/styles/shell.css` | btn/badge/table/card/dialog + chrome + workspace |
| Chrome row 1 | `src/shell/Navigation.tsx` | apps, ⌘K, identity, log out |
| Chrome row 2 | `src/shell/Subnav.tsx` | empty slot; modules portal in |
| Resume sections | `src/features/resume/components/ResumeSubnav.tsx` | live counts from `resumeSummary` |
| Workspace | `src/features/resume/pages/ContentEditorPage.tsx` | rail + editor + preview |
| Rail | `src/features/resume/components/EntryList.tsx` | j/k navigation |
| Editor | `src/features/resume/components/DocumentPane.tsx` | draft, autosave, conflict, recovery |
| Publish strip | `src/features/resume/components/PublishStrip.tsx` | real draft counts + live bundle |
| Preview pref | `src/features/resume/previewPref.ts` | external store across the portal boundary |

### c.b Flow

```
route /app/resume/:type/:slug
   │
   ├─ ResumeSubnav ──portal──> #subnav-slot   (counts ← useResumeSummary)
   │
   ├─ useContentList(type) ──> EntryList ──(j/k, click)──> navigate(:slug)
   │
   ├─ DocumentPane key={slug}
   │     useContentItem ─┐
   │     useDraft ───────┼─> MarkdownEditor ──onChange──> autosave 2s
   │                     └─> conflict? ──> ConflictDialog ──> DiffViewer (lazy)
   │
   └─ previewing? ──> MarkdownPreviewPane (API-rendered HTML, lazy)
```

### c.c Subs and classes

- `SubnavPortal({children})` — resolves `#subnav-slot` in an effect, `createPortal`s into it.
- `useAppSwitchKeys(enabled)` — unmodified `1`–`4` navigate between apps; suppressed while
  the palette is open or the caret is in a field.
- `usePreviewShowing()` / `togglePreview()` — `useSyncExternalStore` over a module-level
  boolean, persisted to `localStorage`, both accessors guarded in try/catch.
- `useEntries(nodes)` — unwraps the masked `ContentListEntryFields` fragment once.
- `DocumentPane({type, slug, file})` — mounted with `key={slug}`.

## Deliberately not built

| Artboard section | Why |
|---|---|
| Identity → Users / Sessions / Registration | No IAM endpoints in this app. Building it means shipping fabricated accounts, session keys and audit rows. |
| Settings → API tokens | Same — no token API. |
| Technologies group editor | No group/item mutation in the schema; content types are markdown documents. |
| Diff/publish right drawer | `PublishPage` already does this with real idempotent publishing; a drawer is a relocation, not a feature. |
| Signed-out screen, toasts, confirm dialog | Auth already redirects via `/auth/failed`; no toast system to hang the design's copy on yet. |

## Follow-ups

1. Confirm the build is green in the browser, then add coverage (per standing instruction,
   tests come only after that confirmation).
2. Decide whether Identity is in scope for this repo or belongs to `iam.anudeep.pro`.
3. Modernist ships no dark ramp — `color-scheme` is pinned to light. Retune in
   `tokens.css` if a dark portal is wanted.
