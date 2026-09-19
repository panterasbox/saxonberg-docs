# Author diagnostics

The **author-diagnostics** subsystem is the feedback surface for people
authoring content: it surfaces errors in the content they wrote —
compile-time TypeScript errors, runtime throws in a brain / script /
controller, and (for wizards) the raw server console — in a searchable
store read from both the in-game shell and the CMS.

The problem it solves: a content author working through the CMS browser
surface or the shell had **no feedback path** for errors in their own
content. A runtime throw in an NPC brain dead-ended at `console.warn` on
the server terminal (invisible to a browser-connected author); a command
that blew up reached the player as a generic *"Command execution failed"*
with the real error swallowed; ambient TypeScript type errors only
appeared in CI. This subsystem closes that gap.

Seeded by the author-diagnostics slate, requirements, and plan (all
retired at merge — this doc is the live reference; see git history at the
commit range in *History* below).

## Shape at a glance

Three producers → one store → two-audience delivery → two readers.

```
PRODUCERS                        STORE                READERS
runRootGuarded ─┐                                  ┌─ errors verb (shell)
console tap ────┼──record──► diagnostics ◄─────────┤
compile watcher ┘            (Mongo, TTL)          └─ CMS panel (REST poll)

DELIVERY (on each live structured write, via Events.Diagnostic)
  author ← ProvenanceApi.authorOf → online Avatar → MudlogApi.error
  (giver's surface on the command path is the existing controller-error note)
```

The store is the source of truth; delivery (the author push) is a
best-effort side-effect — a missed push never loses the row.

## The store

`DiagnosticApi` (`mud/api/diagnostics.ts`) is the gated forwarding shell
over the hot-reloadable `DiagnosticLogic` singleton at
`/platform/idea/api/diagnostics`. The store is the `diagnostics` Mongo collection,
accessed through the raw `getCollection` surface (plain-scalar rows carry
no marshalled value-objects, so there is no `Document` subclass — the
`HotReloadApi` convention). Each row is a `DiagnosticDoc`
(`@saxonberg/types`): `source` (`runtime` | `compile` | `console`),
`severity`, `channel`, `path`, `author`, `versionId`, `code`, `line`,
`col`, `message`, `stack`, `ts`, `expiresAt`.

Four indexes (in `PersistenceManager.createIndexes`): `{channel, ts}`
(tail-by-channel), `{author, ts}` (the "my content" read), `{path,
versionId}` (the compile supersede), and the **first-of-kind TTL**
`{expiresAt}` with `expireAfterSeconds: 0` (Mongo evicts each row at its
`expiresAt` — free rotation, default 7 days, no cron).

Two write paths:

- **`record(RuntimeDiagnostic)`** — one runtime row, always appended,
  always live (fires `Events.Diagnostic`).
- **`recordDiagnostics(path, versionId, diags, {live})`** — a compile
  batch that **supersedes**: it fully replaces the file's prior compile
  rows, so a fixed file (empty `diags`) clears. `live` gates the event
  (cold-start batches pass `false`).

### The channel taxonomy

`lib/diagnostics/DiagnosticChannel` is the pure value-object that maps an
absolute path to a channel (first match wins): a `/zones/<zone>/` segment
or the lounge tree → `zone.<zone>`; `/lib/<subsystem>/` → `lib.<subsystem>`;
`/platform/idea/cmd/` or `/platform/cmd/` → `command`; `/api/` → `api`; else → `global`.
It also owns `expandSubscription` (resolving the synthetic `$cwd`) and
`matches` (the `*` / `lib.*` prefix / exact grammar). Pure and
unit-tested; `DiagnosticApi` re-exports the three on its face.

## Producer 1 — the runtime execution-context guard

`ExecutionContextApi.runRootGuarded(target, method, fn, policy)` is a
**sibling** of `runRoot`, not a change to it — `runRoot` stays a hot
no-catch primitive so framework-internal roots pay nothing. The guard
awaits `fn`, and on a throw/rejection records a runtime diagnostic
(channel + author derived from the `target`'s content path, via a
**dynamic import** of `DiagnosticApi` to avoid the bootstrap-Api load
cycle) and applies a **per-caller policy**:

- `'absorb'` — swallow, return `undefined` (the command inbound path).
- `'rethrow'` — record and re-throw (the REST bridge, so `sendCmsError`
  still maps the status).
- `'swallow'` — record and return without re-throwing (scheduled /
  background work — a timer tick has no caller to rethrow to).

The three adopters:

- **Command path** — a two-seam design, because controller throws are
  **already absorbed** inside `CommandGiver.executeCommand` into
  `controller-error` notes (the giver's real-error surface) before they
  reach `runRoot`. So the substantive capture is `DiagnosticApi.record`
  at the three controller-error emission sites (path
  `/platform/idea/cmd/<controller>`), via a swallowing fire-and-forget helper;
  the `runRootGuarded('absorb')` at `backend/inbound/command.ts` catches
  only **residual framework escapes** and retires the generic *"Command
  execution failed"* socket frame. The note stays the giver's surface —
  there is **no** duplicate `MudlogApi.error` to the giver.
- **Scheduled work** — `ScheduleApi.planRun` and the `SchedulerRegistry`
  emission root use `'swallow'`. (This also fixed a latent bug: a sync
  throw under the old `runRoot` killed `fireFixedDelay`'s reschedule.)
- **REST** — `CmsSession.runAsSessionPlayer` uses `'rethrow'`.

## Producer 2 — the console tap

`backend/ConsoleTap.ts` is a backend singleton installed as the **first
statement in `main()`** (so the boot baseline is captured). It wraps
`console.log/info/warn/error` into a bounded in-memory ring and **passes
through** to the original console (terminal output unchanged). The ring is
**raw and unattributed** — engine internals, third-party chatter — and is
**wizard-read-only** (via `DiagnosticApi.rawTail` / `errors raw`). It
writes only to the ring, never the `diagnostics` collection; instrumented
seams keep their `console.warn` for terminal visibility, so they appear in
the ring too (an accepted overlap — the ring is a terminal mirror; the
store is the curated view).

## Producer 3 — the TS compile watcher

`backend/CompileWatcher.ts` runs a long-running `tsc --watch` via the
TypeScript **Compiler API** (`createWatchProgram` +
`createSemanticDiagnosticsBuilderProgram`), streaming **semantic**
diagnostics into `recordDiagnostics`. Dev-only (booted in `AppBootstrap`
when `NODE_ENV !== 'production'`; prod has no TS source). Cold-start
batches flow through `{ live: false }` so authors aren't firehosed with
the project baseline; `#ready` flips on TS status codes 6193/6194, after
which rechecks are live. Semantic-only in v1 — parse errors are already
surfaced by tsx at module load (`Events.ModuleReloadFailed`) and the CMS
save-time reload error. A missing tsconfig / watcher failure logs and
continues (diagnostics never blocks boot).

## Delivery — the author push

`DiagnosticApi.startRouter()` (booted once in `AppBootstrap`, idempotent)
registers the single `Events.Diagnostic` listener. For each live row it
resolves the content's author via `ProvenanceApi.authorOf(path)` (which
returns a `templatePath`), looks up the **online** Avatar
(`StuffApi.findByTemplatePath`), and if present pushes a `MudlogApi.error`
frame. Offline author → store-only (they see it via `errors list --mine`
or the CMS panel). Store-is-truth, push-is-courtesy. The **giver** on the
command path is served by the existing `controller-error` note, not by
this push.

## Gating

Server-authoritative on the **context-derived** actor
(`ExecutionContextApi.getActingAuthor`, the CMS anti-spoof property —
never a passed value):

| Op | Gate |
|---|---|
| `list` (structured) | **per row** — the within-your-extent pattern: a row is yours to read when its `path` is under an extent you hold (`AccessApi.heldExtents`), or its channel is `pack.<id>` and you maintain that pack (`PackApi.maintainersOf` — a member of the maintainers group, or staff/head of the maintaining organization); `compile` rows (no path) read on the wizard axis alone, and non-wizards never see them. `mine` bypasses the filter (the author is the context actor) |
| `rawTail` (console ring) | `isWizard` (enforced in `ErrorsController`) |
| `clear(path)` | `isWizard`, or the provenance author of the path — else `-1` |
| producer writes | the `DiagnosticApi` facade → `DiagnosticLogic` `FromModule` gate |

There is **no author tier** (content-packs wave 3; [access.md](./access.md)):
`ErrorsController` has no door of its own — the verb is afforded to every
avatar, and a nobody's `errors` is simply an empty list, not an error.
A `null` (unattributable) context fails every read closed (`list` → `[]`).

**Pack channels route to maintainers.** A diagnostic on channel
`pack.<id>` (an install failure, a manifest error) is pushed to the
pack's maintainers — the maintainers group's online members, or an
organization's staff and head — and, when the pack is **unstaffed**, to
the executive (`/compact/executive`'s committee). Anything else routes
to the row's provenance author, as before.

**Where a runtime throw lands.** A throw with a *local* handler is caught
there first: an NPC brain throw is caught by `BehavedMixin` (which
`console.warn`s it), so it surfaces in the **console ring** (`errors raw`),
not the structured store. The structured runtime guard captures throws that
reach a guarded `runRoot` uncaught — command-controller throws (via the
`CommandGiver` seam) and other unguarded background work.

## Reader A — the `errors` verb

`content/platform/cmd/system/errors.yaml` + `ErrorsController` (category `system`,
AuthorMixin-afforded, author-or-wizard gated in the controller).
Subcommands: `list` (default) —
`--channel` / `--severity` / `--source` / `--mine` / `--limit` filters and
a greedy `path` prefix arg; `raw` — the wizard-only console ring
(`--grep`); `clear <path>`. Live tailing is the mudlog author-push plus
the per-author `compile.subscribe` List setting on `AuthorMixin` (default
`['$cwd', 'global']`; mutation awaits a future `errors subscribe`
subcommand — the `settings set` list-type gap).

## Reader B — the CMS panel

`GET /api/cms/diagnostics` (on `CmsRoutes`) binds 1:1 through the
`CmsSession.runAsSessionPlayer` attribution bridge to `DiagnosticApi.list`
— **no new authorization surface**. The client `CmsDiagnosticsPanel` is a
third tab in `CmsSurface`; since the CMS tab opens no WebSocket it
**polls** the route on an interval, defaulting to the spoof-safe `mine`
lens (author resolved server-side) with an "all" toggle. A `cms-delta` WS
push channel is the reserved upgrade.

## Deferrals

- Promoting raw `console.error` into the structured store (v1 keeps the
  ring strictly raw; structured coverage is the runRoot guard).
- Syntactic compile diagnostics; compile-watcher crash-restart backoff;
  the ring-size / poll-cadence AppSettings; the `cms-delta` WS.
- The `errors subscribe` subcommand (the `compile.subscribe` schema ships
  now; mutation awaits List-type `settings set`).
- ESLint diagnostics (the `source`/channel/store shape extends without API
  changes).

## History

Built as `feature/author-diagnostics` (commit range `3c7d1c6f..HEAD`).
Three shifts from the plan/requirements were load-bearing:

- **The command "chokepoint" is not `runRoot`.** The plan assumed a single
  `runRootGuarded` at the command boundary would catch authored throws; the
  code review found controller throws are already absorbed into
  `controller-error` notes *inside* `CommandGiver` first. So command capture
  became the **two-seam** design (record at the note sites; the `command.ts`
  guard only catches residual escapes + retires the generic frame).
- **The reader gate widened from author-tier to author-or-wizard** —
  and then the author tier itself went (content-packs wave 3). Driving
  the feature live showed a wizard (who edits engine source/TS — exactly
  who needs compile diagnostics) was locked out by `requiresAuthor`; the
  verb dropped the validator. Wave 3 replaced "author or wizard" with the
  per-row extent filter above: `compile` rows stay on the wizard axis,
  everything else is read by whoever holds the row's path.
- **The `errors` verb renders plain escaped text, not an MML list.** The
  nested `Mml.unorderedList`/`compose` round-trip through `fromMarkup`
  dropped the list body (only the count rendered); switched to a plain
  escaped multi-line block.
