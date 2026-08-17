# CMS — content-authoring surface

The CMS is the **author → save → live** loop over the content the game
already runs on: a browsable explorer over the two trees the engine reads
(Mongo-backed **templates** and the **source** filesystem), a code editor,
and a save that validates server-side, persists to the right backend, and
makes the change live without a restart.

This is the **first build** of the CMS / authoring tools
(cms-slate.md, Wave 1) — the "start with
the code editor" front half, built against the backend that already exists
(`SourceTreeApi`, `TemplateApi`, `HotReloadApi`, `AccessApi`). It is
**wizard-tier**: the surface is for people who can already write source
and edit templates. The lease model, content editors, versioning, git, the
LSP, and the law==code review gate are **deferred** — see *Deferral
boundary* below.

## Shape at a glance

- **Surface** — a route of the existing client SPA (`?surface=cms`), opened
  in its **own browser tab** sharing the one express-session. It is
  **REST-only**: the CMS tab opens **no WebSocket**.
- **Transport** — REST for CMS data (reusing express-session); every
  request pays the **session→`runRoot` attribution bridge** so the op is
  attributed to the acting player. No GraphQL.
- **Server surface** — a thin gated `CmsApi` → `CmsLogic` singleton
  forwarding to the four existing Apis; it reimplements none of them.
- **Editor** — Monaco, lazy-loaded into its own Vite chunk (only the editor
  mode pays the bundle), with stock TS/JSON/YAML support.

## The unified-tree projection

Three backends, one navigation model:

- **`content`** — `Template` docs in the `domain` collection (you edit
  `data`), via `TemplateApi`.
- **`source`** — sandboxed files via `SourceTreeApi`, **rooted at the
  mudlib** (`packages/server/src/mud`). Authors edit game content, not
  client/build scaffolding, so the source root is mud — not the monorepo —
  and colocated **`__tests__` folders are hidden** from the tree. Source
  paths are mud-relative (`/api/cms.ts`, not `/server/src/mud/api/cms.ts`),
  and `..` that climbs out of mud throws (mud is a hard boundary, enforced
  in `CmsLogic` via `sourceAbs`, on top of the `SourceTreeApi` sandbox).
- **`document`** — the path-addressed **document store** (`StoredDocument`
  in the `documents` collection) via `DocumentApi` — the generic
  owner-claimed JSON tree ([document-store.md](./document-store.md)). Each
  record carries a **`kind`** that drives the editor treatment: a `script`
  kind is a plain-text code leaf (its write funnels through
  `ScriptApi.saveScript` — the script chokepoint: gate + provenance + AST
  go-live); any other kind is its `data` pretty-printed as JSON. Scripts
  ride it today; dorm-room customization is the deferred next kind.

A **node ref** is `{ backend: 'content' | 'source' | 'document'; path }`.
`path` is the canonical identifier *within that backend* (a template path
`/obj/Avatar/foo` for content; a mud-relative path `/api/cms.ts` for
source; a store path `/home/<player>/scripts/<name>` for document).
**There is no merged namespace** — the unified-ness is that one `CmsApi`
and one explorer drive all three, discriminated by `backend`. The explorer
shows three fixed roots ("content" / "source" / "documents"); a synthetic
merged root was deliberately avoided (it would invent a namespace the
backends don't share and complicate write-dispatch).

A **leaf** carries an editable body: in content, a non-Zone template (its
`data`); in source, a file. A **folder** has children and no body: a Zone
template, or a directory. The folder/leaf split rides
`ZoneApi.isFolderClass(tpl.class)` for content and `isDir`/`isFile` for
source.

For **content**, `listTree` also **synthesizes namespace folders** for
intermediate path segments that have no template doc of their own —
`/obj` and `/lib` are browsable even though only `/obj/X` / `/lib/Y`
templates exist. Without this most engine content is unreachable from the
root (the `domain` collection has templates at `/obj/Avatar/<id>`,
`/lib/lounge/...` etc., but no literal `/obj` or `/lib` doc). A child is a
real template at depth+1, or a synthetic folder implied by deeper
descendants; entries sort folders-first, then alphabetically.

## `CmsApi` / `CmsLogic`

`CmsApi` (`mud/api/cms.ts`) is a thin, security-gated forwarding shell —
the structural twin of `source-tree.ts`. Four statics, each forwarding 1:1
to the hot-reloadable `CmsLogic` singleton at `/obj/api/cms` (reached
synchronously via `StuffApi.singletonSync`):

```
listTree(backend, path)       → CmsTreeListing   // immediate children
read(backend, path)           → CmsReadResult     // a leaf's body + language
stat(backend, path)           → CmsStatResult     // existence/kind probe
write(backend, path, body)    → CmsWriteResult    // gated + go-live
```

**There is no `actor` parameter — by design (the anti-spoof property).**
The acting principal is resolved *inside* `CmsLogic` from the execution
context (`ExecutionContextApi.getActingAuthor`): the command-frame giver on
the in-game path, or the `tagActingAuthor` stamp a backend boundary planted
on the REST path. A caller who holds a reference to a privileged Avatar
(via MQL / `findByTemplatePath` / `self`) **cannot** substitute it for the
gate's subject, because there is no argument to pass it through — exactly
the provenance/standing rule (*derive from context, never trust the call
argument*). A context with no derivable actor → `null` → every gate fails
closed. **All cross-backend dispatch lives in `CmsLogic`**, never in the
REST handlers — the write gates are module-private functions mirroring
`WriteController._gateContentWrite` / `_gateSourceWrite` verbatim. Every
public `CmsLogic` method carries `@CallSecurity(FromModule('/api/cms#CmsApi'))`:
anything that grabs the singleton and calls it other than through `CmsApi`
gets `SecurityError`.

The wire types (`Cms*`) live in `@saxonberg/types`. The one
application-level error, **`CmsError`** (with a `code: 'denied' |
'not-found' | 'invalid'`), is exported from `mud/api/cms.ts` itself — homed
with the surface that throws it, no separate `lib/cms/` module. Sandbox
escapes stay the existing `SourceTreeSandboxError`.

## REST data API + the WS split

Five routes, mounted by `CmsRoutes.setup(app)` in `Server.setupRoutes()`
after the session/passport middleware and before the SPA catch-all:

```
GET  /api/cms/tree?backend=&path=   → CmsApi.listTree
GET  /api/cms/read?backend=&path=   → CmsApi.read
GET  /api/cms/stat?backend=&path=   → CmsApi.stat
POST /api/cms/write                 → CmsApi.write   (CSRF-protected)
GET  /api/cms/csrf                  → mint a double-submit token
```

Each route binds **1:1 to a gated `CmsApi` op through the bridge**. **No
authz lives in the route layer** — it validates payload shape, calls the
bridge, and maps the thrown errors onto HTTP status via `sendCmsError`
(`CmsError.code` → 403/404/400, `SourceTreeSandboxError` → 400, else 500).
This is the governing property: *any transport is just another entry path
to the same gated core op*, so REST adds no new authorization surface.

**No GraphQL** — an open-query graph would force re-authorization at every
field; discrete named REST routes map 1:1 to coarse gated ops, which fits
the access model.

**CSRF** — writes require a double-submit token: the client fetches
`/api/cms/csrf` once (stored in `req.session.cmsCsrf`) and echoes it in the
`X-CMS-CSRF` header on every write.

**The WS split.** This build adds **no new WebSocket message**. CMS data is
REST-only; the CMS tab opens no socket. The author→save→live loop still
works live: a wizard with the **game tab** open observes a content
save's effect there because the go-live step mutates the live world that
tab already renders via its existing MQL subscriptions. A CMS-tab WS channel
(live tree/file deltas, cross-tab "someone edited X", SharedWorker
coherence) is deferred; the REST 1:1-to-gated-op shape reserves room for a
future `cms-delta` frame to bind the same `CmsApi` ops without rework.

## The attribution bridge

`CmsSession.runAsSessionPlayer(req, method, fn)` (`backend/CmsSession.ts`)
is the single reusable "run-as-session-player" wrapper. A REST request
arrives *outside* the command pipeline, so it lacks the execution root the
WS command path establishes. The bridge supplies it:

1. **Resolve the acting Avatar.** The session carries a *userId*; a `User`
   owns `playerIds[]`; the in-world Avatar is the already-loaded clone for
   one of those slots (`PlayerApi.findAvatarByPlayerId`). Anonymous-guest
   principals, missing users, and sessions with **no loaded Avatar** resolve
   to `null`.
2. **Run inside `ExecutionContextApi.runRoot(Backend, method, fn)`** so
   every downstream `@CallSecurity` gate resolves against a well-defined
   Root frame, and any event fired during the op (e.g. during
   `restoreFromTemplate`) carries provenance.

This is the same mechanism `Backend.processUserMessage` uses for WS and
`ScheduleApi` uses for deferred work; the bridge lives in `backend/` because
only `backend/**` may push call frames (so only a backend boundary can
stamp `tagActingAuthor`). The stamp is the **single channel** by which a
REST op learns who is acting — feeding both the access gates and provenance
attribution via `getActingAuthor`. A `null` actor (no in-world Avatar) fails
closed → **CMS ops require an in-world Avatar in the session** (a CMS tab
with no game tab in-world is denied — correct: you must be in-world to
author).

## Gating

Server-authoritative, on the **context-derived** actor (never a passed
value — see *§ `CmsApi` / `CmsLogic`*). The whole surface is
authoring-tier: source (engine TS) is wizard-only; content (templates)
is author-tier. Writes mirror `WriteController` verbatim:

| Op | Gate (subject = `getActingAuthor()`) |
|---|---|
| Source read / list / stat | `isAuthor`? no — **`isWizard`** |
| Content read / list / stat | **`isAuthor`** |
| Source write | `isWizard` **and** `can('write', resolveSourceFolderZone(path))` |
| Content write | live Zone → `canMutateZone(zone)`; else `can('write', liveAtPath)` |

A `null` (unattributable) context fails every gate closed. Denials surface
as `CmsError('denied')` → HTTP 403 → an inline error in the editor, never a
silent no-op.

`/auth/status` carries a top-level **`isWizard`** boolean (read via
`AccessApi.isWizard` for the session avatar) so the client can hide the
CMS launcher for non-wizards. This is a **non-authoritative UX hint** —
the REST gates remain the authority.

## Save go-live (template vs source)

Save is authoritative: the server re-validates and persists, then runs a
per-backend go-live step. The two backends diverge because **templates are
Mongo docs, not files**, and `HotReloadApi.reload` reads a *filesystem
path*:

- **Source** → `HotReloadApi.reload(absPath)` (literal). New clones pick up
  the new class. A reload failure (compile error) leaves the file persisted
  but **not live** — the result reports `reloaded: false` with the error,
  rather than 500-ing.
- **Content** → re-hydrate **every** live clone at the path via
  `TemplateApi.restoreFromTemplate` (`StuffApi.findAllByTemplatePath` —
  *all*, not the throws-on-many `findByTemplatePath`). `saveTemplate`
  persists the new `data`; re-hydrating makes it observable in the running
  world. The editor edits `data` only, so the template's `class` /
  `hydratorClass` round-trip unchanged.

## Client surface

The CMS opens as its own tab (`?surface=cms`), a full-screen takeover
bypassing the cockpit and the connection-phase switch, opening **no
WebSocket**. A launcher in the account menu (visible only when
`auth.isWizard`) opens it.

- **Explorer** (`CmsExplorer` + `CmsTreeNode`) — two fixed roots; folders
  lazy-expand via `listTree` (children cached per node); leaves open via
  `read`. Themed via `tokens`.
- **Editor** (`CmsEditor` + `MonacoLazy` → `MonacoInner`) — Monaco wrapped
  in `React.lazy` + `Suspense`, so it code-splits into its own Vite chunk
  loaded **only when the editor first mounts** (the main bundle is
  unchanged). `monaco-editor` is bundled **locally** (not the
  `@monaco-editor/react` default CDN fetch) via `loader.config`, with
  Vite-native worker imports for the stock TS/JSON language services. Save
  is dirty-gated; the go-live note (or the "not live" reload error) surfaces
  inline.
- **Store** — a `cms` slice in the single Zustand store owns its async REST
  IO directly (REST-only, so unlike the WS-handler-registry slices); node
  key is `${backend}:${path}`. A save adopts the draft as the new persisted
  baseline.

## Deferral boundary

What this build does **not** do, and where it lands:

- **Lease-scoped trees, group-managed content, the holodeck, op-log
  (`domain_history`) template versioning** → the access slate
  (access-slate.md). This build gates on
  the current `AccessApi` (dev-tier) and writes HEAD directly — **no
  history**. The eventual group-managed exemplar is the lounge / EU.
- **Engine-typed IntelliSense, the LSP, the VS Code extension** → the
  authoring-intelligence slate
  (authoring-intelligence-slate.md).
  Monaco ships with stock language support only.
- **Content editors** (schema-driven room/zone forms) → Wave 3 of the slate.
- **The law==code / forums-review publish gate** — change-centric review
  riding the forums argument-map ([forums.md](forums.md); see the slate's
  *The review gate* section) → a later wave; it gates *publish*, which
  presupposes a changeset model not built here.
- **Drafts / staging / changeset overlay + atomic publish** → later (depends
  on the versioning/changeset model).
- **Git** → deferred entirely; GitLab becomes a future *runtime* integration,
  not the authoring workflow (the source-agnostic review model is the VCS
  spine).
- **Anon read-only** → deferred, and now harder than a flag flip: reads are
  author-gated on the context-derived actor, so an anon reader has no actor
  → denied. A future anon-browse would need an explicit unauthenticated
  read path (its own gate decision), **scoped to the content tree only**,
  never the source FS — not just dropping `requireAuth`.
- **Cross-tab live-state sync** (SharedWorker / BroadcastChannel) → later.

**Forward constraint (decision A):** the eventual review/versioning model is
**storage-agnostic** — git stays the code backend, Mongo+op-log the template
backend, one uniform changeset/publish workflow over both. Later work should
not entangle the review model with a specific store.

## History

- **Go-live diverged from the requirement's literal wording.** The
  requirement said "save → `HotReloadApi.reload`"; in practice `reload`
  applies only to source files (it reads a filesystem path). Templates go
  live by re-hydrating their live clones — the correct, behavior-equivalent
  mechanism.
- **`isWizard` landed top-level on `/auth/status`, not on its `player`
  payload** — that route doesn't populate `player` today (the WS
  connection-established path does), so a top-level flag was the minimal
  in-grain choice.
- **Monaco is bundled locally, not from its default CDN** — so the editor
  chunk is self-contained and Vite-code-split, with the language workers
  imported Vite-natively.

## "Test in holodeck"

The editor toolbar carries a button that steps the author's game
session into their sandbox circle. It has **no endpoint of its own** —
it sends `go wardrobe` over the game tab's existing socket
(`websocketClient.sendCommand`), the way the forum cards' buttons
dispatch commands, and the ordinary refusal comes back in-fiction when
the author isn't standing at a door.

The sandbox build originally shipped `POST /api/sandbox/test-session` →
`SandboxApi.launchTestSession` for this; both were **deleted** before
merge. The method was `enter` under another name, and the name asserted
something the model denies: there is no act of *launching* a session. A
session is bookkeeping that exists because a body is on the far side of
a door. See [sandbox.md](./sandbox.md).

## ⭐ The CMS is a CARD now

`cms` is a verb (`mud/cmd/author/cms.yaml`, `requiresWizard`) and it
opens a `client`-source card. The **server** owns the card's existence,
identity, lifetime and pinned-ness; only the BODY is the client's,
because the explorer, the tree and Monaco all speak the `/api/cms/*`
routes — which is where what a wizard may see and write is actually
decided. A controller that answered that would be a second
authorization surface for one question.

⚠ Pinned by default, and it declares `noProse`. An editor that aged out
of a relevance window mid-edit would be a data-loss bug wearing a
lifetime rule; and a `terminal`-only `shell.result` must not take
Monaco away, which is what the declaration prevents.

⭐⭐ **`CmsSurface`'s four-tab mode bar is gone.** Files · Kinds ·
Diagnostics · Git was a second switcher, in the second column, doing
exactly what the right column's switcher did — and for the same reason
that had expired. `cms`, `git` and `studio` are three cards in one feed
now, each opened by its own verb. `BuilderLayout` renders that feed
FILLING its column rather than at the rail's fixed width: in `build`
mode the cards are the work.

See [card-surface.md](./card-surface.md).
