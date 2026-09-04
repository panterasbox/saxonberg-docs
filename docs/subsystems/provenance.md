# Provenance: the authorship ledger

Who authored a content path — **derived from an append-only ledger, never
read from a mutable stamp**. This is the **first concrete brick of the
provenance substrate** (see
provenance-slate.md); its immediate
consumer is the producer influence stock (see [influence.md](./influence.md)),
which routes attributed-engagement credit to a content path's author.

Authorship as a ledger (not a `createdByPlayerId` field) is deliberate. A
mutable field is **not an authority** (the author owns the `data` blob and
could overwrite it), is **re-stampable** on every save, and is
**audit-gapped**. The ledger answers all three: append-only, one row per
authoring act, `authorOf` *derives* the original author.

## The ledger

- **`AuthoringEvent`** (`lib/standing/AuthoringEvent.ts`, `authoring_events`)
  — one append-only row per authoring act: `path` (the authored template
  path), `author` (the authenticated author's durable `templatePath`),
  `kind` (`'save'`, open vocab), `at`/`realAt`. Indexed on `path` (derive
  authorship of a path) and `author` (a creator's body of work). Nothing is
  ever overwritten.
- **`ProvenanceApi`** (`api/provenance.ts`) → **`ProvenanceLogic`**
  (`platform/idea/api/ProvenanceLogic.ts`, `/platform/idea/api/provenance`):
  - `recordAuthoring({ path, … })` — append one row (the author is NOT a
    parameter; see below).
  - `authorOf(path): string | null` — the **earliest** row's author (the
    original author; a later save by a different player never changes it).
    The contributor-set / team split is the deferred enrichment behind this
    derivation seam.
  - `eventsFor(path)` — the raw per-path log reader.

## The author is context-derived, never passed

`recordAuthoring` takes **no `author` parameter** — it cannot be passed, so
it cannot be a value a caller asserts. The author is resolved from the
dispatched execution context via **`ExecutionContextApi.getActingAuthor()`**,
which is **transport-agnostic** and reads only frames the framework itself
stamps:

- **in-world (command) path** — the command-frame stack's giver, but only on
  a **non-forced, single-consistent-giver** chain. A forced dispatch
  (`CommandApi.forceCommand`) or a cross-actor cascade (A's command
  triggering B) **fails closed** → `null`. This is "look at the giver *and*
  the stack around it" — the bare top giver is not trusted alone.
- **REST / CMS path** — no command frame, so the acting principal is whatever
  a transport boundary stamped via **`ExecutionContextApi.tagActingAuthor()`**
  — a frame-**metadata** tag, **decoupled from the frame's security
  `target`**, so a REST boundary names its author without disturbing what
  downstream `@CallSecurity` gates resolve as the caller.

`getActingAuthor` returns the principal (a `Stuff`) or `null`;
`ProvenanceLogic` accepts it only if it carries a durable `templatePath` (a
real authoring identity) — otherwise no row. The author is never read from
the author-controlled `data` blob.

## The centralized, gated writer

`ProvenanceApi.recordAuthoring` is the **single** writer of provenance, gated
**`FromTemplate('/platform/idea/api/template')`** — its sole legitimate caller is the
`TemplateLogic` save singleton (a registered Stuff, so its caller frame
resolves to its template path, not a module id). Every authoring transport —
the in-world authoring verbs (`write`, …) and the REST CMS — funnels through
`TemplateApi.saveTemplate`, which appends one row after the save commits; no
other code may write a row. The author is resolved **DB-free first**
(`getActingAuthor` is an execution-context read), so a non-authoring save (no
acting avatar, forced, system) is a pure no-op — it never touches
persistence.

The operator-trust residue (a developer forging rows) is the *same*
irreducible boundary as `renown_events`, answered by tamper-evidence +
transparency + exit later — it meets the existing bar, doesn't lower it.

### Cross-worktree CMS contract

The REST CMS (build-2 `cms-editor`) reaches authorship through
`TemplateApi.saveTemplate`, so it is covered automatically. Its `runRoot`
boundary (`CmsSession.runAsSessionPlayer`) **does** call
`ExecutionContextApi.tagActingAuthor(actor)` (`backend/CmsSession.ts`), so
`getActingAuthor` resolves the session's in-world Avatar and CMS authoring
writes are attributed today — **not** "safely unattributed until then"
(that earlier note was stale). The same `getActingAuthor` seam is what the
wizard-authority **code-field gate** reads to identify the CMS author (see
[access.md § The code-trust lockdown](./access.md)); a context with no
derivable actor still yields `null` → no row, never *misattribution*.

## CreditRouting — the producer faucet's routing input

`lib/standing/CreditRouting.ts` resolves **who gets producer credit** for an
engagement at a location, Layer-1 (authorship) → Layer-2 (faucet):

- **`CreditShare`** `{ author, weight }` — one author's share; `author` is a
  durable `templatePath`, `weight` is the fraction (v1 always 1; the
  team-split seam).
- **`CreditRouting.resolve(locationTemplatePath): CreditShare[]`** — resolve
  the covering zone (`ZoneApi.resolveZoneForPath`) → its template path →
  `ProvenanceApi.authorOf` → `[{author, weight: 1}]`, or `[]` when there is
  no covering zone, no recorded author, or the content is unreleased.
- **The released-content gate** (`CreditRouting.isReleased`) — only a
  player's personal homedir (`/home/…`) is unreleased; **`/platform/… + /stuff/…` (released
  core content) and `/world/…` both earn**. Zero new infra (a path prefix);
  the richer team-sandbox + explicit `release` action is deferred with the
  team split.

`resolve` is pure routing over the gated `ZoneApi` + `ProvenanceApi`; v1
returns a single owner, the deferred team split returns a weighted vector
behind the same `CreditShare[]` seam, so the producer faucet is built once
and untouched.

## Deferred (the rest of the provenance substrate)

The contributor-set / co-authorship model (multiple weighted authors per
path); the ownership hierarchy and the dependency-DAG credit graph; the
git-in-runtime VCS; the explicit `release` action + team sandbox. See
provenance-slate.md.
