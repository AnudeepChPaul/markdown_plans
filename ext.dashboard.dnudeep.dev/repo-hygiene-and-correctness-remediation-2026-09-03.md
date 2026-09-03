# ext.dashboard.dnudeep.dev — Repository Hygiene & Correctness Remediation

**Date:** 2026-09-03
**Repo:** `github.com/AnudeepChPaul/ext.dashboard.dnudeep.dev`
**Branch:** `main`
**Framework:** Sencha Ext JS 7.3.0.55 (universal app, Sencha Cmd 7.x)
**Status:** Phase A + Phase C implemented (uncommitted). Phase B + D deferred by decision.

---

## a) What am I trying to do

### a.1 Baseline

The repository is a Sencha Cmd universal application (`Dashboard`) that acts as the
**admin/CMS back-office for the résumé site `dnudeep.dev`**. It manages skills, projects,
work experience and the site's introduction copy, with a notifications inbox, an email
inbox and a three-column Kanban board.

Scale is heavily skewed toward vendored code:

| Segment | Size | Nature |
|---|---:|---|
| `ext/` | 138 MB | Ext JS 7.3.0.55 framework, commercially licensed (`ext/license.txt`) |
| `packages/{calendar,d3,exporter,froala-editor,pivot,pivot-d3,pivot-locale}` | ~26 MB | Sencha premium packages, **zero references from the app** |
| `packages/dash-classic-matte/` | 168 KB | First-party classic theme, `extend: theme-triton` |
| `app/`, `classic/src/`, `modern/src/` | ~150 KB | First-party application, **~1,814 LOC** |
| `bootstrap.*`, `classic.json`, `classic.jsonp` | ~370 KB | Sencha Cmd dev-mode output, **committed in error** |

Git history is three commits (`Initial commit` → `Add files via upload` →
`[InitialCommit] - Adding the ext dashboard project`), 8,013 tracked files, no branches,
no CI, no tests, no lint configuration.

### a.2 Baseline architecture

```
index.html
  └─ Ext.beforeLoad(tags)          profile = ?classic|?modern override
     │                                     : tags.desktop ? 'classic' : 'modern'
     ├─ Ext.buildSettings.baseCSSPrefix = 'dash-'
     └─ bootstrap.js (microloader) ─▶ app.js
                                        Ext.application({
                                          extend:  'Dashboard.Application',
                                          requires:['Dashboard.*'],
                                          mainView:'Dashboard.view.main.MainView'
                                        })

Dashboard.view.main.MainView  (border)      controller:'main'  viewModel:'main'
└─ Dashboard.view.main.Resumeview (border)
   ├─ north   dash-header        Header.js       app combo + inbox/bell tools
   ├─ west    dash-sidemenu      Menu.js         segmentedbutton ──▶ {selectedMenu}
   ├─ center  layout:'card'  activeItem ◀── {selectedMenu}
   │            itemId:'dashboard'      dash-board          Dashboard.js
   │            itemId:'configurations' dash-config         Configurations.js
   │            itemId:'tasks'          dash-tasks          Tasks.js
   │            itemId:'mails'          dash-email          Emails.js
   │            itemId:'notifications'  dash-notifications  Notifications.js
   └─ south   statusbar          Ext.ux.StatusBar   itemId:'dash-stat'
```

Two architectural facts that constrain every change below:

1. **There is no router.** `app/controller/` does not exist. Navigation is a pure
   two-way ViewModel binding: `Menu.js` writes `{selectedMenu}`, the card layout in
   `Resumeview.js` reads it as `activeItem` and matches against child `itemId`s.
2. **`dash-email` and `dash-notifications` are each instantiated twice** — once as a
   standalone card under `Resumeview`, once embedded inside `dash-board`. Any
   `down('dash-email')` therefore addresses only one of two live instances.

Data layer:

```
Dashboard.model.Base  (idProperty:'id', identifier:'uuid', schema ns 'Dashboard.model')
   ├─ Notification   proxy: localstorage  id:'notifications'
   ├─ Experience     proxy: localstorage  id:'experiences'
   ├─ Skill          proxy: rest  idProperty:'techId'     rootProperty:'technologies'
   └─ Project        proxy: rest  idProperty:'projectId'  rootProperty:'projects'

MainModel.stores { notifications, projects, skills, experiences }
   all: autoLoad:true, autoSync:true   ──▶ every grid edit round-trips immediately
```

REST endpoints target an out-of-repo Flask service on `http://127.0.0.1:5000/resume/api/…`.

Cross-component messaging is a single global event: `ConfigController.saveIntroduction`
fires `Ext.GlobalEvents` `updatestatus`; `MainController.onUpdateStatus` writes it to the
status bar.

### a.3 Defects to be removed

**Blocking**

| # | Defect | Location |
|---|---|---|
| B1 | Modern profile cannot build: requires non-existent `Dashboard.store.Personnel`; its `main` controller/viewmodel live under `classic/src/` (outside the modern classpath); `app.js` `mainView` is classic-only | `modern/src/view/main/List.js:9`, `Main.js`, `app.js` |
| B2 | API host hard-coded to `127.0.0.1:5000` across 8 endpoints; no environment override | `app/model/Skill.js`, `app/model/Project.js` |

**Correctness**

| # | Defect | Location |
|---|---|---|
| C1 | Three handlers referenced but defined nowhere (`handleBuyAction`, `handleSellAction`, `onRemoveClick`); no store bound; still carries Sencha "Plants" demo columns | `dashboard/Emails.js` |
| C2 | Experiences grid trash `actioncolumn` has icon + tooltip but **no handler** | `Configurations.js` |
| C3 | `submitExperience` pushes *every* project in the store instead of the tagfield selection | `ConfigController.js` |
| C4 | `submitExperience` `add()`s unconditionally, but edit-mode fields are two-way bound to the selected record → **duplicate record on update** | `ConfigController.js` |
| C5 | `form.getValues()` serialises a `tagfield` to a joined string; the model expects an array of ids | `ConfigController.js` |
| C6 | `openNewExperienceWindowWithValues` never resets the form (its sibling does) | `ConfigController.js` |
| C7 | Ancestor-xtype reach-throughs: `up('dash-main').getViewModel()`, `up('dash-config').getController().getViewModel()` | `Notifications.js`, `Configurations.js` ×2 |
| C8 | `templatecolumn` emits `<a onclick="window.open('{url}')">` from unescaped user input | `Configurations.js` |
| C9 | `d = sourceEl.cloneNode(true)` undeclared → implicit global on every drag | `tasks/TaskPanel.js` |
| C10 | `getTargetFromEvent` passes an element **id** where a **selector** is expected → never matches | `tasks/TaskPanel.js` |
| C11 | Drop inserts `record.getData()` (detached plain object), losing identity/dirty state; manually nulls `record.store` | `tasks/TaskPanel.js` |
| C12 | `getDragRecordId` dereferences a possibly-null `sourceEl` | `tasks/TaskPanel.js` |
| C13 | Selection clearing uses `down()` on a doubly-instantiated xtype | `MainController.js` |
| C14 | Global `Ext.data.Store` override whose only live code is three `console.log`s — fires for every store incl. framework stores | `app/store/Base.js` |
| C15 | Typos: `makrSelectedUnread`, `align:'strechmax'`, duplicate `scale` key in `defaults` | 3 files |
| C16 | `markAllNotificationsRead(choice)` unused param; `filter().forEach()` allocates a throwaway array | `MainController.js` |

**Hygiene**

| # | Defect |
|---|---|
| H1 | `README.md` and `Readme.md` share **one inode** (case-insensitive APFS) and **both are tracked**. Sencha boilerplate has overwritten the real README; splits into two files on a case-sensitive checkout |
| H2 | No `.gitignore`; generated artifacts + 6 `.DS_Store`/AppleDouble files committed |
| H3 | ~26 MB unused premium packages vendored (`app.json` requires only `font-awesome`, `ux`, `charts`) |
| H4 | `dashboard/InfoSlide.js` is 107 lines of fully commented-out chart code |
| H5 | No tests, no CI, no ESLint/Prettier/EditorConfig; `modern/src` diverges from house style |

### a.4 Goal

Make the repository **clonable, runnable and correct for someone other than the author**,
without altering the product surface, and without touching the vendored framework.

---

## b) What are my options

### Option 1 — Documentation only

Write a real README and an analysis document. Change no code.

**Pros**
- Zero regression risk; nothing to re-test.
- Fastest path to onboarding a second contributor.

**Cons**
- Every defect in a.3 survives. C8 (XSS-shaped column) and C14 (global store override)
  are shipped as-is.
- B2 keeps the app unusable off the author's machine.
- The README would have to *document* broken behaviour (C1, C2) rather than fix it.

```
┌──────────────┐
│  README.md   │◀── new prose only
└──────────────┘
   app/, classic/src/, packages/  ── UNCHANGED ──▶ all 16 correctness defects remain
```

---

### Option 2 — Hygiene + correctness, product surface frozen  ◀ SELECTED

Four phases, of which **A** (hygiene/config) and **C** (correctness) execute now;
**B** (modern profile) and **D** (tooling/pruning) are explicitly deferred.

**Pros**
- Removes all 16 correctness defects and both hygiene blockers.
- Config extraction (B2) is the single highest-leverage change — it converts a
  machine-local app into a deployable one.
- Every change is local to first-party code; the vendored framework is untouched, so the
  diff is reviewable in one sitting (~12 files).
- Deferrals are recorded, not forgotten: the modern-profile breakage becomes a documented
  **Known issue** in the README rather than a silent trap.

**Cons**
- No build or runtime verification is possible on this machine (no `node`, no `npx`,
  no `sencha`, no JS engine) — see §c.6. Verification is static only.
- Fixing `Emails.js` (C1) requires *inventing* a `Dashboard.model.Email`; there is no
  smaller correct form of "bind a real store", so this slightly widens scope.
- `modern/` stays broken; non-desktop visitors still need `?classic`.

```
┌─────────────────────────────────────────────────────────────┐
│ PHASE A  hygiene + config          ── executes now          │
│   .gitignore │ untrack artifacts │ README split │ config.js │
├─────────────────────────────────────────────────────────────┤
│ PHASE C  correctness               ── executes now          │
│   Emails │ ConfigController │ TaskPanel │ reach-throughs    │
├─────────────────────────────────────────────────────────────┤
│ PHASE B  modern profile            ── DEFERRED              │
│ PHASE D  lint / prune / dead code  ── DEFERRED              │
└─────────────────────────────────────────────────────────────┘
```

---

### Option 3 — Full rewrite onto a modern stack

Discard Ext JS; rebuild the admin as React/Vue against the same Flask API.

**Pros**
- Sheds a 138 MB commercially-licensed dependency and its Sencha Cmd toolchain.
- Opens the door to a real test suite and CI, neither of which exists today.
- The app is only ~1,800 LOC — the rewrite is genuinely tractable.

**Cons**
- Throws away the `dash-classic-matte` theme and every grid/row-editor/drag-drop
  behaviour that Ext provides for free.
- Weeks of work to reach parity on an internal tool that already works for its one user.
- Does not fix the current repository, which stays broken while the rewrite lands.

```
   Flask API (unchanged)
        ▲            ▲
        │            │
   ┌────┴────┐  ┌────┴──────────┐
   │ Ext 7.3 │  │ new SPA       │  ← parallel maintenance during migration
   │ (legacy)│  │ React + Vite  │
   └─────────┘  └───────────────┘
```

---

### Decision

**Option 2**, scoped to **Phase A + Phase C**.

Rationale: Option 1 leaves a security-shaped defect (C8) and a machine-local API host
(B2) in place; Option 3 is disproportionate to a working ~1,800-LOC internal tool. Within
Option 2, Phase B was explicitly deferred (`modern/` left broken, documented) and Phase D
deferred (lint + package pruning are independent of correctness).

---

## c) Selected option — detailed design

### c.1 Phase A — target state

```
repo root
├── .gitignore                 NEW   build/, bootstrap.*, classic.json{,p},
│                                    packages/*/build/, .DS_Store, ._*, .idea/, .vscode/
├── config.js                  NEW   pre-boot settings; listed in app.json "js"
├── README.md                  REWRITTEN  real project README
├── docs/
│   └── sencha-structure.md    NEW   relocated Sencha Cmd boilerplate, retitled
└── app.json                   MODIFIED  "js": [config.js, app.js]
```

**Load order — why `config.js` is not an `Ext.define`d class.**

The first implementation used `Ext.define('Dashboard.Config', { singleton: true, … })`
and had the models call it from their own class body. That is wrong:

```
Ext.Loader fetches Skill.js
  └─ evaluates the object literal passed to Ext.define
       └─ evaluates `Dashboard.Config.restApi(...)`   ◀── ReferenceError
  └─ ONLY THEN does Ext.define read `requires` and schedule Dashboard.Config
```

A class body is evaluated *before* its own `requires` are resolved, so `requires` cannot
protect a call made inside the literal. The fix is a plain namespace assignment loaded by
the microloader ahead of every application class:

```
manifest "js" order (framework entries are prepended by the build profile):
   1. ${framework.dir}/build/ext-all-rtl-debug.js   ← Ext available
   2. config.js                                     ← Dashboard.config defined
   3. app.js  ──▶ Ext.application ──▶ Dashboard.* classes evaluated
```

**`config.js`**

```js
Ext.namespace('Dashboard').config = {
  apiBaseUrl: 'http://127.0.0.1:5000/resume/api'

  /**
   * @param {String} service     e.g. "project_service"
   * @param {String} readAction  e.g. "get_projects"
   * @return {Object} Ext.data.proxy.Rest "api" config
   */
  , restApi: function (service, readAction) {
    const base = `${this.apiBaseUrl}/${service}`
    return {
      create:  `${base}/update`
      , read:  `${base}/${readAction}`
      , update:`${base}/update`
      , destroy:`${base}/delete`
    }
  }
}
```

**Model call sites**

```js
// app/model/Skill.js
, proxy: { type:'rest', api: Dashboard.config.restApi('technology_service','get_technologies'), … }

// app/model/Project.js
, proxy: { type:'rest', api: Dashboard.config.restApi('project_service','get_projects'), … }
```

**README collision (H1).** `README.md` and `Readme.md` are one inode; both are in the
index. `git rm --cached Readme.md` would resolve the pathspec case-insensitively and take
`README.md` with it. The safe sequence:

```
cp README.md docs/sencha-structure.md          # preserve boilerplate content
sed -i '' '…retitle…' docs/sencha-structure.md
git update-index --force-remove Readme.md      # index-only, exact path, no FS lookup
<write new README.md>
```

---

### c.2 Phase C — class & method design

#### `Dashboard.model.Email` — NEW (`app/model/Email.js`)

Mirrors `Dashboard.model.Notification` exactly; required because C1 ("bind a real store")
has no smaller correct form — no email model existed.

```js
Ext.define('Dashboard.model.Email', {
  extend: 'Dashboard.model.Base'
  , fields: [ 'from', 'subject', 'received_on', { name:'read', type:'number' } ]
  , proxy:  { type:'localstorage', id:'emails' }
})
```

#### `Dashboard.view.main.MainModel` — MODIFIED

```
data.selectedEmails : null                    NEW  drives @read/@unread disabled binds
stores.emails       : { model:'Dashboard.model.Email', autoLoad:true, autoSync:true }   NEW
```

#### `Dashboard.view.dashboard.Emails` — REWRITTEN

| Before | After |
|---|---|
| no store | `bind: { store: '{emails}' }` |
| `cellmodel` + `cellediting` | `rowmodel` mode `SIMPLE` (read/unread is not cell editing) |
| Plants columns (`light`, `availDate`, weekend `disabledDaysText`) | `read` flag, `from`, `subject`, `received_on` |
| `handleBuyAction` / `handleSellAction` / `onRemoveClick` (undefined) | `markSelectedEmailsRead` / `markSelectedEmailsUnread` / `onRemoveEmail` |
| — | `selectionchange` → `this.lookupViewModel().set('selectedEmails', …)` |

#### `Dashboard.view.main.MainController` — MODIFIED

```js
// NEW
markSelectedEmailsRead()            → setEmailReadState(1)
markSelectedEmailsUnread()          → setEmailReadState(0)
setEmailReadState(read)             // {Number} read 1|0
  ├─ selected = vm.get('selectedEmails'); bail if falsy
  ├─ Ext.Array.from(selected).forEach(rec => rec.set('read', read))
  ├─ view.query('dash-email').forEach(g => g.setSelection(null))   // C13: query, not down
  └─ vm.set('selectedEmails', null)
onRemoveEmail(gridView, rIndex, cIndex, item, evt, record)
  └─ vm.getStore('emails').remove(record)

// CHANGED
markAllNotificationsRead()          // C16: was (choice); filter().forEach() → store.each()
markSelectedUnread()                // C15: was makrSelectedUnread
clearNotificationSelection()        // C13: down() → query().forEach()
init()                              // REMOVED: console.log only
```

#### `Dashboard.view.configurations.ConfigController` — MODIFIED

```js
// NEW — collapses three duplicated Ext.Msg blocks
confirmDelete(onConfirm)            // {Function} onConfirm, invoked with `this` = controller
  └─ Ext.Msg.show({ …YESNO, WARNING, scope:this,
                    fn: opt => opt === 'yes' && onConfirm.call(this) })

// NEW — collapses openNewExperienceWindow + …WithValues (C6)
showExperienceWindow(record)        // {Ext.data.Model} [record] omit ⇒ create mode
  ├─ Ext.create window, bind title '{!experiencesList.selection ? "New" : "Update"} Experience'
  ├─ down('dash-config-experienceform').reset()          // reset BEFORE binding
  ├─ lookupReference('experiencesList').setSelection(record || null)
  └─ return this.view.add(win).showAt()

openNewExperienceWindow()                                    → showExperienceWindow()
openNewExperienceWindowWithValues(tv,rI,cI,btn,evt,record)    → showExperienceWindow(record)

// NEW (C2)
onRemoveExperience(gridView, rIndex, cIndex, item, evt, record)
  └─ confirmDelete(function () { vm.getStore('experiences').remove(record) })

// CHANGED
onRemoveSkill(gridView, rIndex)     → confirmDelete(…)
onProjectItemClick(…)               → confirmDelete(…); brace style normalised
submitExperience()                  // C3 + C4 + C5
  ├─ bail unless form.form.isValid()
  ├─ if (lookupReference('experiencesList').getSelection())  → skip add
  │     // fields are two-way bound to the record; it is already mutated in place
  ├─ else add({ companyName, designation, duration, visible,
  │             projects: Ext.Array.from(form.down('[name=projects]').getValue() || []) })
  │             // read the tagfield directly: getValues() joins it to a string
  └─ form.up('window').close()

init()                              // REMOVED: console.log only
```

**Why C4 is a real defect** — the edit path binds fields to the record itself:

```
ExperienceForm field  ──bind '{experiencesList.selection.companyName}'──▶ Experience record
                          (two-way: typing mutates the record live)

old submitExperience: store.add({…})   ⇒ record mutated  +  duplicate row appended
new submitExperience: selection ? no-op : add(…)
```

#### `Dashboard.view.configurations.Configurations` — MODIFIED

| Defect | Change |
|---|---|
| C15 | `align:'strechmax'` → `'stretchmax'` |
| C8  | `tpl: '<a onclick="window.open(\'{url}\');">{url}</a>'` → `'<a href="{url}" target="_blank" rel="noopener noreferrer">{url}</a>'` — `templatecolumn` HTML-encodes `{url}` |
| C7  | `this.owner.up('dash-config').getController().getViewModel()` → `this.owner.lookupViewModel()` |
| C7  | `view.ownerGrid.up('dash-config').getController().getViewModel()` → `view.ownerGrid.lookupViewModel()`; `value.map` → `Ext.Array.from(value \|\| []).map` |
| C2  | Experiences trash item gains `handler: 'onRemoveExperience'` |

#### `Dashboard.view.tasks.TaskPanel` — MODIFIED

```js
getDragRecordId(e)                  // C12
  ├─ sourceEl = e.getTarget(this.itemSelector, 10)
  └─ return sourceEl ? Number(sourceEl.getAttribute('record-id')
                              || sourceEl.parentElement.getAttribute('record-id'))
                     : NaN

DragZone.getDragData(e)             // C9: `var sourceEl, d` — `d` was an implicit global

DropZone.getTargetFromEvent(e)      // C10
  └─ e.getTarget(scope.itemSelector, 10)     // was: e.getTarget(<element id string>)

DropZone.onNodeDrop(target, dd, e, data)     // C11
  ├─ record = data.draggedRecord; return false if !record || data.sourceStore === scope.store
  ├─ dropIndex = scope.getDragRecordId(e)
  ├─ data.sourceStore.remove(record)
  └─ scope.store.insert(isNumber(dropIndex) && !isNaN(dropIndex) ? dropIndex + 1 : 0, record)
```

Drag/drop data flow, before vs after:

```
BEFORE                                    AFTER
sourceStore.remove(rec)                   sourceStore.remove(rec)
rec.store = null            ← manual      (Ext detaches on remove)
target.insert(i, rec.getData())           target.insert(i, rec)
   └─ plain object: new id, dirty            └─ same record: identity + dirty state kept
      state lost, uuid regenerated
```

#### `Dashboard.view.dashboard.Notifications` — MODIFIED

```
handler: 'makrSelectedUnread'  →  'markSelectedUnread'                      (C15)
selectionchange: this.up('dash-main').getViewModel()  →  this.lookupViewModel()   (C7)
```

#### `Dashboard.view.sidemenu.Menu` — MODIFIED

`defaults` declared `scale` twice (`'medium'` then `'large'`); the first was dead. Removed.

#### `Dashboard.store.Base` — DELETED (`app/store/Base.js`)

```js
Ext.define('Dashboard.store.Base', {
  override: 'Ext.data.Store'          // ← global: every store in the app, framework included
  , listeners: {
      load:   function () { /* entirely commented out */ }
    , add:    (s, r) => console.log('add: ', r)
    , update: (s, r) => console.log('update: ', r)
    , remove: (s, r) => console.log('remove: ', r)
  }
})
```

Referenced nowhere (`grep -rn 'Dashboard.store.Base' app classic modern` → only its own
definition). Deleted rather than gated behind a debug flag: an unconditional global store
override that logs on every mutation is a production footgun with no offsetting value.

---

### c.3 Complete change manifest

| File | Action | Defects closed |
|---|---|---|
| `.gitignore` | new | H2 |
| `config.js` | new | B2 |
| `app.json` | modified — `"js"` array | B2 |
| `docs/sencha-structure.md` | new (relocated boilerplate) | H1 |
| `README.md` | rewritten | H1 |
| `Readme.md` | index entry removed | H1 |
| `app/model/Email.js` | new | C1 |
| `app/model/Skill.js` | modified | B2 |
| `app/model/Project.js` | modified | B2 |
| `app/store/Base.js` | **deleted** | C14 |
| `classic/src/view/dashboard/Emails.js` | rewritten | C1 |
| `classic/src/view/dashboard/Notifications.js` | modified | C7, C15 |
| `classic/src/view/main/MainModel.js` | modified | C1 |
| `classic/src/view/main/MainController.js` | modified | C1, C13, C15, C16 |
| `classic/src/view/configurations/ConfigController.js` | modified | C3, C4, C5, C6, C2 |
| `classic/src/view/configurations/Configurations.js` | modified | C2, C7, C8, C15 |
| `classic/src/view/sidemenu/Menu.js` | modified | C15 |
| `classic/src/view/tasks/TaskPanel.js` | modified | C9, C10, C11, C12 |
| *untracked (kept on disk)* | `bootstrap.js`, `bootstrap.css`, `classic.json`, `classic.jsonp`, 6 × `.DS_Store`/`._*` | H2 |

---

### c.4 Deferred — Phase B (modern profile)

Decision: **leave broken, document it.** The README now carries an explicit Known-issue
callout directing non-desktop users to `?classic`.

When picked up, two mutually exclusive designs:

**B-1 (recommended) — drop the modern profile.**

```
- rm -r modern/
- app.json: delete builds.modern
- index.html: Ext.beforeLoad ⇒ Ext.manifest = 'classic' unconditionally
```
Matches reality: the app is desktop-only (border layouts, row editors, `Ext.dd` drag zones).

**B-2 — repair it.** Promote `MainController`/`MainModel` to the shared `app/` classpath,
author the missing `Dashboard.store.Personnel`, give `modern` its own `mainView`. Larger,
and it incurs an ongoing obligation to build a real mobile UI.

### c.5 Deferred — Phase D (tooling & pruning)

- ESLint + a config matching the house style already present in `classic/src/`
  (comma-first, 2-space, semicolon-free, ES6). `modern/src/` does not conform.
- Delete `classic/src/view/dashboard/InfoSlide.js` — 107 lines, 100 % commented out (H4).
- Prune unused premium packages, keeping `dash-classic-matte` (H3, ~26 MB).
- No test harness exists; `ext/test/` is the framework's own Jazzman suite, not the app's.

---

### c.6 Verification

**Executed in this session — static only.**

| Check | Result |
|---|---|
| Bracket/paren/brace balance across all 12 changed JS files (strings + comments stripped) | 12/12 balanced |
| Every string handler reference in `classic/src` resolves to a method on `MainController` or `ConfigController` | 21/21 resolve |
| `grep -rn '127.0.0.1:5000' app classic modern` | only `config.js:11` |
| `git ls-files \| grep -cE '(^\|/)(\._\|\.DS_Store)'` | 0 |
| `grep -rn 'Dashboard.store.Base' app classic modern` | no references before deletion |

**Not executed.** No `node`, `npx`, `deno`, `bun`, or `sencha` on this machine, and no JS
engine reachable from the installed JRE. **Nothing was built, parsed by a real parser, or
run.** The following must be done manually before this is trusted:

1. `sencha app build production` — must complete with no unresolved-class errors. This is
   the check that currently catches B1 (`Dashboard.store.Personnel`), which remains open
   by decision; a `classic`-only build is the meaningful gate.
2. `sencha app watch classic` → `http://localhost:1841`.
3. Start the Flask API on `127.0.0.1:5000`. Without it, skills and projects load empty and
   `autoSync` writes fail silently.
4. **Configurations** — create/edit/delete a skill; create a project; create an experience,
   then edit it and confirm **exactly one** row exists afterwards (C4); confirm the
   experience's `projects` field holds an **array of ids**, not a joined string (C5);
   confirm the Skills URL column opens in a new tab (C8); confirm the experience trash icon
   deletes (C2).
5. **Dashboard** — select rows in Emails, mark read/unread, delete (C1); confirm selection
   clears in **both** instances of the grid — the embedded one and the standalone `mails`
   card (C13).
6. **Tasks** — drag a card between all three columns; confirm the card moves rather than
   duplicating, and that `window.d` is never created (C9, C10, C11).
7. Confirm `localStorage` keys `notifications`, `experiences`, `emails` survive a reload.
8. Confirm no `console.log` fires on store mutations (C14).

### c.7 Commit plan

Not yet staged or committed. Per the repo's git conventions: no Claude-specific commentary
in the message, and a detailed body summarising the whole change. Suggested split:

```
1. chore: add .gitignore and untrack Sencha build output
2. docs: restore project README, relocate Sencha boilerplate to docs/
3. refactor: extract API base URL into pre-boot config.js
4. fix: correct experience editor, task drag-drop, and email grid defects
```

### c.8 Open risks

| Risk | Mitigation |
|---|---|
| No parser/build ran; a syntax error could reach `main` | Run `sencha app build production` before committing |
| `TaskPanel.getTargetFromEvent` change is behavioural and untested | Manual drag test (§c.6 step 6) |
| `Dashboard.model.Email` is invented, not specified — field names are a guess | Confirm against whatever feeds the inbox, or delete the grid instead |
| `lookupViewModel()` requires Ext ≥ 6.5 on `Ext.Component` | Satisfied: framework is 7.3.0.55 |
| Load order of `config.js` assumes build-profile `js` entries are prepended to top-level `js` | Verified against `app.json` structure; confirm in the generated manifest after first build |
