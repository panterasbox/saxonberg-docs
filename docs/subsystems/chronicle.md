# Chronicle — the append-only identity ledger

The **chronicle** is a character's accumulating, append-only record of
identity-shaping narrative events — the **dumb ledger** every future
identity readout (recognition, reputation, alignment, traits,
achievements) will project from. This build ships the ledger plus one
real way to read it, and **none** of the readouts.

It is the direct sibling of the belief store, and copies its spine
exactly: read [belief.md](./belief.md) first. The one deliberate
divergence is **no mixin** — see [No mixin](#no-mixin-the-thinnest-surface).

## Dumb store, smart consumers

The substrate stores entries and fetches them owner-scoped. **All the
intelligence lives in the consumers.** There is no per-tag meaning, no
readout logic, and no MQL provider in the substrate. `tags` and `who`
are persisted and owner-fetchable but otherwise **inert in v1** —
designed in for the deferred reputation / alignment / traits /
achievements readouts that will query them later.

This is the same dumb-store / smart-consumers split the belief store
makes: a plain-data `Document` subclass, an owner-indexed collection, a
single gated Api as the only mint/read surface, and a write-path
find-then-save upsert for the idempotent case.

## `ChronicleEntry` — the Document

`lib/chronicle/ChronicleEntry.ts` is a plain `Document` (not Stuff): the
row **is** the entry. It mirrors `BeliefDocument` — `static
collectionName = Collections.Chronicles`, a `persistent` entry in `fieldMeta`,
public scalar/array fields the `Hydrator` reflects into by name.

One document per entry, **never a growing array** (the `ContactsMixin`
16 MB / whole-array-rewrite anti-precedent — the choice the belief store
also made). The `chronicles` collection is indexed on `owner` (declared
centrally in `PersistenceManager.createIndexes`) so the owner-scoped read
(`ChronicleEntry.find({ owner })`) and a future per-player cleanup
cascade (`deleteMany({ owner })`) stay O(rows-for-this-owner).

| field | meaning |
|---|---|
| `owner` | the character's `templatePath` — the durable owner key, **indexed** |
| `kind` | `'deed'` or `'claim'` — provenance (below) |
| `when` | game-time **seconds magnitude** (deeds only); `null` for claims |
| `order` | authored prologue order (claims only); `null` for deeds |
| `where` | place `templatePath` ref, or `null` |
| `who` | entity `templatePath` refs — **inert in v1** |
| `text` | the rendered line (deed: `ProseApi`; claim: authored) |
| `tags` | open vocabulary — **inert in v1** |
| `key` | `recordOnce` category-first dedup key, or `null` |

All scalars / arrays-of-scalars, so the scalar-default marshalling rule
applies — no field marshaller needed. `when` stores the **seconds
magnitude as a plain number** (`WorldClockApi.getNow().rawValue()`),
keeping the Document scalar.

### deed vs. claim — by provenance

The two kinds are distinguished by **where they come from**, not by any
runtime difference:

- **deed** — something that *happened*. `when` is the game-time witness;
  `text` is rendered by `ProseApi` at mint time (so raw interpolations
  are MML-escaped). Minted by content at identity-impactful moments.
- **claim** — an authored *prologue* line, seeded at char-gen from the
  chosen aspiration. `order` places it; `text` is the author-trusted
  seed from `char-gen.yaml`. `when` is `null` (a claim is backstory, not
  a timestamped event).

## `ChronicleApi` / `ChronicleLogic` — the gated mint/read seam

The mint/read surface follows the platform's Api ↔ logic-singleton
split. `api/chronicle.ts` (`ChronicleApi`) is a **thin forwarding
shell**; the logic lives in the hot-reloadable `ChronicleLogic`
singleton at `/obj/api/chronicle` (`obj/api/ChronicleLogic.ts`), reached
synchronously via `StuffApi.singletonSync` — exactly as
`BeliefStoreApi` forwards to `BeliefStoreLogic`. `dest /obj/api/chronicle`
reloads it.

Like belief, the logic's internal helpers (`active()`, `ownerKey()`, the
shared mint path) are **module-private free functions**, not
intra-singleton `this.x()` calls — a self-call would trip the
`FromModule('/api/chronicle#ChronicleApi')` gate (the caller would be
`ChronicleLogic`, not the allowed `ChronicleApi`). Every public method
no-ops without a durable owner key (`owner.getTemplatePath()`) or an
active Mongo connection (`PersistenceManager.get().isConnected()`).

| method | role |
|---|---|
| `record(owner, fields)` | the always-append primitive — build one entry and save it |
| `recordDeed(owner, fields)` | convenience: forces `kind: 'deed'`, so `text` renders from `template` via `ProseApi` and `when` defaults to `WorldClockApi.getNow()` when omitted |
| `recordOnce(owner, key, fields)` | category-first idempotent: the first entry under `key` (for this owner) wins; later calls no-op |
| `entriesFor(owner)` | the owner-scoped reader — **the only reader v1 ships** (no MQL provider); `[]` when keyless/disconnected |
| `seedClaims(owner, seeds)` | char-gen helper: mint `kind: 'claim'` prologue entries with `order` + authored `text` |

There is a **single build seam** inside the logic: it renders prose when
a `template` is given (the one "deed text via `ProseApi`" point) and
stamps the game-clock witness onto a deed when `when` is omitted (the one
"timestamp is the witness" point), so callers never re-derive game-time.
A claim forces `when = null` and keeps `order`; a deed forces `order =
null`.

`recordOnce`'s first-time check is a **write-path read only** (find by
`{ owner, key }`, then save) — the find-then-save mirror of the belief
upsert; it never sits on a hot read path. The race is benign under
per-socket command serialization (the same argument
[belief.md](./belief.md) makes).

### Singularity comes from the trigger, not universal dedup

Three patterns, chosen by how the mint site fires:

- **event-singular** — `record` / `recordDeed` riding a naturally-once
  trigger (e.g. enroll fires once per character by construction). **No
  `key`.**
- **category-first** — `recordOnce(owner, key, …)`: the first under `key`
  wins. For "the first time you ever did X."
- **repeatable** — plain `record`, always appends.

## The identity-impact authoring gate

**No engine auto-subscription to an event bus.** What is worth
chronicling is an **authoring principle**, not an engine-enforced rule —
the substrate offers only the `record(…)` seams, and content mints
deliberately at the moments that matter.

The gate: an event earns a chronicle entry iff it shapes **who the
character is** — recognized, regarded, aligned, trait-relevant,
affiliated, or part of the story worth telling — **not** iff it is a
"feat."

**Chronicle vs. achievements.** Achievements answer *"what have you
done"* (a checklist of feats); the chronicle answers *"who are you / your
story"* (a narrative of identity). They overlap but aren't equal:

- **Both:** identity-shaping feats — graduated, made guildmaster, met
  Gus, your first real conversation.
- **Achievement-only** (NOT chronicle): completion / grind / skill-flex —
  "100% the map," "100 h played," "no-damage boss."
- **Chronicle-only** (no trophy): quiet identity beats — "Dave gave you a
  job," "you made your first friend," "you took the dark god's bargain."

The engine never decides what's identity-impactful; content mints at the
moments that are. Achievements, when built, is simply a readout over the
achievement-`tag`ged subset — one more deferred consumer.

**The gate doubles as the anti-gaming property.** Everything downstream
is *measured from reality* (the chronicle is what you did; reputation is
how others actually reacted), so the metric **is** the reality. Identity
events are intrinsically scarce/organic (you graduate once, meet Gus
once), so they resist farming by nature; the farmable categories
(grind, completion %, time-played, kill-counts) are exactly what the gate
excludes.

## Char-gen claim-seeding

Each aspiration in `mud/config/char-gen.yaml` carries a `claimSeeds: [{
text, order }]` array (the prologue), **distinct from `bioSeed`** — both
read the same aspiration, neither touches the other. At
`EnrollController.commit`, after the avatar is cloned and registered (so
`getTemplatePath()` resolves), `ChronicleApi.seedClaims(avatar,
aspiration?.claimSeeds ?? [])` mints the `claim` entries.

## The `chronicle` verb — the self-view

A single-token, zero-arg, **self-only, read-only** verb (the MVC triple:
`cmd/charactergen/chronicle.yaml` + `obj/command/charactergen/
ChronicleController.ts` + the controller seed). It subsumes the *view*
role the char-gen subsystem sketched as the deferred `records` verb;
`records`-style bio **editing** stays deferred.

The verb is afforded by **`PersonaMixin`** (via a mixin-level
`commandContributions.self`, collected by the affordance walk
`collectBucketDefs` → `MixinApi.queryMixins`, exactly like
`PerceiverMixin`'s self verbs) — Persona is its conceptual home because
it already owns the `bio` / `aspiration` the view reads.

The controller reads `ChronicleApi.entriesFor(actor)` and renders, in a
fixed order and **never interleaved**:

1. **bio** — the Persona-owned claimed self-narrative (escaped: it is
   player-editable free text);
2. **prologue** — `claim` entries by `order` ascending;
3. **timeline** — `deed` entries by `when` ascending.

Each entry's `text` is MML-safe (claim authored; deed `ProseApi`-rendered
with raw interpolations escaped), so it re-wraps via `Mml.fromMarkup`.
The readout emits on the existing self-facing `world.identity` topic
(reuse, not a new topic). Empty state still renders the bio plus a
"your chronicle is just beginning" line.

## Three demo minters

To prove the seam and guarantee every character has a timeline, three
minters ride moments that already fire:

- **enroll founding deed** (`EnrollController.commit`) — `recordDeed`,
  **event-singular** (enroll fires once per character; no `key`).
- **first-arrival deed** (`Avatar.enter`) — `recordOnce` keyed
  `'first-arrival'`, called **unconditionally** (not gated on the
  `firstArrival` greeting flag): the `recordOnce` key is the dedup
  authority, so the first ever arrival mints once and every re-login
  `enter` no-ops. Minted next to `BeliefStoreApi.hydrate`; no-ops when
  disconnected, so a Mongo hiccup never blocks the welcome scene.
- **first-introduce deed** (`IntroduceController.execute`) — `recordOnce`
  keyed `'first-introduce'`, after the scene `.send()` and the
  `learnIdentity` loop. (Adding the awaited mint made `execute` `async` —
  the dispatcher `await`s controllers.)

## No mixin — the thinnest surface

Belief carries a `BeliefStoreMixin` **because** it serves a no-Mongo
naming **hot path** with a live in-memory working set. The chronicle has
**no hot path** (mint is a rare identity moment; the `chronicle` verb
reads on demand and may read Mongo) and **no owner-side runtime state** —
entries exist only as Documents, fetched owner-scoped when needed.
Therefore the thinnest surface is **no mixin**: the owner key is read
directly off `owner.getTemplatePath()` in the logic, exactly as
`RecognitionApi.learnIdentity` reads `subject.getTemplatePath()`.
`lib/chronicle/` holds only the `ChronicleEntry` Document.

## `who` needs a referent for people who aren't Stuff

`who` holds entity `templatePath` refs and is **inert in v1** — and it
can only ever point at something that exists as `Stuff`. The dead, the
absent, the historical and the merely mentioned have nowhere to live, so
most of what a chronicle entry would want to name is unnameable.

The proposed answer is an **unrealized person record** — a path-addressed
`Document` carrying identity without embodiment, so a chronicle entry, a
contract and a wiki article can all reference the same person whether or
not anyone simulates her. See
lineage-slate § *The parents are
unrealized*, which needs the same primitive for a different reason.

## Deferred consumers

Everything that *reads* the chronicle is deferred. **reputation**,
**alignment**, **traits**, and **achievements** are all readouts that
will project from the ledger via `entriesFor` (and, later, MQL over the
owner-scoped set), each interpreting `tags` / `who` in its own way. The
substrate ships designed-for-them but ships none of them.

## Death and return (2026-07-31)

The ledger now records both edges of the mortality arc — a `death` deed at
the transition, a `passage` deed on re-embodiment. See
[mortality.md](./mortality.md).

A death inside a sandbox circle still writes (the collection is PASS(mark),
so the row persists carrying its circle stamp): "you died in a holodeck" is
a true thing about you, and readers may lens the mark. Only the *crime*
derivation declines to convict on it.
