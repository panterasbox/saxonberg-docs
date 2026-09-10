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

## The owner face on `PersonaMixin` — the mint/read surface

The Api OO sweep retired `ChronicleApi`/`ChronicleLogic` whole: the
owner face lives ON `PersonaMixin` (`lib/character/Persona.ts` — the
mixin that already affords the `chronicle` verb), its mint path
module-private beside it. The generic any-kind `record` primitive did
NOT survive — every caller declares claim or deed (P4: `recordDeed`
unqualified belongs to the chronicle, the subsystem whose doc owns the
word). Every method no-ops without a durable owner key
(`owner.getIdentityPath()`) or an active Mongo connection.

| method (on the owner) | role |
|---|---|
| `recordClaim(fields)` | mint a claim (`when = null`, `order` kept) |
| `recordDeed(fields)` | mint a deed: `text` renders from `template` via `ProseApi` and `when` defaults to `WorldClockApi.getNow()` when omitted |
| `recordChronicleOnce(key, fields)` | category-first idempotent: the first entry under `key` (for this owner) wins; later calls no-op |
| `chronicleEntries()` | the owner-scoped reader — **the only reader v1 ships** (no MQL provider); `[]` when keyless/disconnected |
| `seedChronicleClaims(seeds)` | char-gen helper: mint `kind: 'claim'` prologue entries with `order` + authored `text` |

The mutators are sealed `@Final @Unshadowable` (the append-only
invariant) and **ungated** — the writer set is open by design (content
packs mint claims and deeds as a normal authoring act: arcana's study
claim, the retail menu's known-of, the script interpreter's can-make
deed), so a closed FromX arm list would couple the kernel to optional
packs. There is a **single build seam** (module-private in Persona.ts):
it renders prose when a `template` is given (the one "deed text via
`ProseApi`" point) and stamps the game-clock witness onto a deed when
`when` is omitted (the one "timestamp is the witness" point), so
callers never re-derive game-time. Callers narrow with
`MixinApi.isPersona`.
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

## Authored history — the dossier seeder

Char-gen is no longer the only source of claims. A `Cast` row carries a
**dossier** (`archetype` · `prologue` · `competence` · `renown`) that
`CastMixin` seeds at `postRegister` — the shipped `dispositions:`
precedent exactly: **idempotent, once**, skipped when any `claim` row
already exists, so a re-clone, a reboot or a CMS go-live cannot mint a
second history.

⭐⭐ **Every seeded row carries the archetype that minted it.** `kind`
separates *authored* from *earned*; it does **not** separate an
**archetype claim** from a **deviation claim**, and an author who writes
"a bartender, but unusually blunt" is stating a departure from a
baseline. `deviation = current derived − archetype baseline` is
uncomputable without knowing which baseline. ⚠ It costs one field now and
is **unrecoverable later** — a sum with no seams cannot be unwound
afterwards at any price. The stamp is on `ChronicleEntry`,
`TranscriptEntry` and `DispositionEntry` alike.

`lint:dossiers` checks assert-vs-derive: what the author says a character
should read as, against what the estimator actually derives. ⭐ That is
the one thing stopping a dossier drifting back into a stat sheet, because
a declared value cannot disagree with itself and a seeded history can.
See [advancement.md](./advancement.md) and
lint-family.md.

## Char-gen claim-seeding

Each aspiration in `mud/config/char-gen.yaml` carries a `claimSeeds: [{
text, order }]` array (the prologue), **distinct from `bioSeed`** — both
read the same aspiration, neither touches the other. At
`EnrollController.commit`, after the avatar is cloned and registered (so
`getTemplatePath()` resolves), `avatar.seedChronicleClaims(
aspiration?.claimSeeds ?? [])` mints the `claim` entries.

## The `chronicle` verb — three readings, one question

A read-only verb taking an **optional subject** (the MVC triple:
`cmd/charactergen/chronicle.yaml` + `platform/idea/cmd/charactergen/
ChronicleController.ts` + the controller seed). It subsumes the *view*
role the char-gen subsystem sketched as the deferred `records` verb;
`records`-style bio **editing** stays deferred.

It answers one question — *what does the world remember about you* — and
the answer's shape follows the subject rather than the command:

| subject | reading |
|---|---|
| none | your own: bio → prologue → deeds |
| **a person in reach** | theirs, with one refusal |
| ⭐ **a body of people** | its accountability record — what it has lost, and what it answers for ([accountability.md](./accountability.md)) |

⚠⚠ **It was zero-arg and self-only until the identity build, and so was
`competence`.** Meanwhile the permission model already said an authored
character's history is a fact about the world that any viewer may learn.
⭐ **The gate was open and there was no door**: nobody could ask what
Dave is good at or what the collier has been through, so seeding authored
history would have shipped unreachable — the `feel` / `taste` shape
exactly, a capability that ships and has never run.

**The refusal is the feature.** Asking about another **player** is
declined — a person's own history is theirs to tell; an authored
character's is a fact about the world. `getPlayerId()` is the
discriminator, structurally, so a future player-bearing class behaves the
same without being enumerated.

### ⭐ Resolving the subject — `RecordControllerBase`

`chronicle` and `competence` share it (the `BankingControllerBase`
precedent: a base controller, not a free helper). The rule is **not**
"whatever MQL returns first", and the live drive found both failure
directions in one run:

- `competence dave` answered **"Dave's Bar"** — the Business Idea whose
  `name` also contains *dave* — and reported that a bar knows nothing
  about bartending;
- `chronicle the watch` answered **"a watchful sentry"**, because MQL
  matches a prefix and *watch* is a prefix of *watchful*. The watch the
  sentry answers to was unreachable behind the sentry.

> ⭐⭐ **A subject must be addressed by a word it CALLS ITSELF.** A person
> by a word of their presentation; a body of people by a word of its
> label.

Both halves are "what the thing says its name is", which is the honest
symmetry — and it settles both cases with no special rule for either:
*watch* is a whole word of "the Watch of the Last Counted Mile" and only
a prefix inside "a watchful sentry"; *dave* is a whole word of "Dave" and
only a fragment of "Dave's". A tie goes to the person, because someone
standing in front of you is the likelier subject; when neither matches on
a whole word MQL's loose match still answers, so `competence sentr` keeps
working.

⚠ The subject is a greedy **string** arg, not `type: object`. Two
reasons: a body of people is an `Idea` and stands nowhere, so
`scope: reachable` finds none of them; and an object arg with
`requires: A|B` deletes a check. It also sidesteps issue **#43** — a
definite article breaks every non-greedy object arg in the game, so
`chronicle the watch` parses where `look the sentry` does not.

The verb is afforded by **`PersonaMixin`** (via a mixin-level
`commandContributions.self`, collected by the affordance walk
`collectBucketDefs` → `MixinApi.queryMixins`, exactly like
`PerceiverMixin`'s self verbs) — Persona is its conceptual home because
it already owns the `bio` / `aspiration` the view reads.

The controller reads the subject's `chronicleEntries()` and renders, in a
fixed order and **never interleaved**:

1. **bio** — the Persona-owned claimed self-narrative (escaped: it is
   player-editable free text);
2. **prologue** — `claim` entries by `order` ascending;
3. **timeline** — `deed` entries by `when` ascending.

Each entry's `text` is MML-safe (claim authored; deed `ProseApi`-rendered
with raw interpolations escaped), so it re-wraps via `Mml.fromMarkup`.
The readout emits on the existing self-facing `world.identity` topic
(reuse, not a new topic). Empty state still renders the bio plus a
"your chronicle is just beginning" line — and for a subject who is not
you, *"Nothing is written down about …"*, ⭐ so an `Extra` reads as a
**role** rather than as an empty person.

⚠ Each entry is wrapped with `Mml.li`. `Mml.unorderedList` wraps a
sequence of `<li>` items; handed bare bodies it emits one run-together
blob, which nobody noticed while a prologue was a single line and which
the live drive made obvious the moment a character had three.

## Three demo minters

To prove the seam and guarantee every character has a timeline, three
minters ride moments that already fire:

- **enroll founding deed** (`EnrollController.commit`) — `recordDeed`,
  **event-singular** (enroll fires once per character; no `key`).
- **first-arrival deed** (`Avatar.enter`) — `recordOnce` keyed
  `'first-arrival'`, called **unconditionally** (not gated on the
  `firstArrival` greeting flag): the `recordOnce` key is the dedup
  authority, so the first ever arrival mints once and every re-login
  `enter` no-ops. Minted next to the belief hydrate (`viewer.hydrateBeliefs()`); no-ops when
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
`learnIdentityOf` reads `subject.getTemplatePath()`.
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
