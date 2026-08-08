# Bulk — continuous matter as a holder attribute

The substrate for **continuous, formless, measured matter** (liquid in
v1): water, coffee. Bulk is the sibling of the discrete fungible-stack
substrate ([glob.md](./glob.md)), built on a fundamentally different
premise — **bulk is not a Stuff.**

A discrete glob (30 coins) is one Stuff with a `quantity`. Bulk (0.3 L
of coffee) has no containment node and no Stuff identity at all: it is a
`{ material, amount }` **attribute of its holder** — a thermos, a mug,
the floor. Matter moves between holders through one primitive,
`BulkableApi.transfer`; every bulk verb is a thin direction over it.

Design of record: bulkable-slate.md (the
holds-as-attribute reframe, the per-affordance model, the deferred
tails). This doc is the operational reference for what shipped — the
thermos slice.

## The model

### `BulkableMixin` (`lib/bulk/Bulkable.ts`)

A host composes `BulkableMixin` to carry up to two **bulk slots**, one
per affordance, each gated by an authored boolean flag:

| Affordance | Flag (`data:`) | Meaning | Example |
|---|---|---|---|
| `interior` | `interiorBulk: true` | a vessel holds liquid | thermos, mug, urn |
| `surface` | `surfaceBulk: true` | liquid pools on a surface | the floor's puddle |

The two affordances are **independent of the spatial mixins**:
interior-bulk does not require `Container` (a fluid-only thermos holds
no pens), and surface-bulk does not require `Surfaced` (the floor
carries a puddle without being a discrete-resting surface). Composition
is explicit per host — the auto-compose-on-every-Container question is
deferred.

A slot's persistent state (flat fields, per affordance):

- `interiorMaterial` / `surfaceMaterial` — the contained Material's
  templatePath (an identity ref; resolved on read, HMR-safe), `null` ⇒ empty.
- `interiorAmount` / `surfaceAmount` — `Quantity<'L'>`, marshalled via
  `QuantityMarshaller.pathFor('L')`. Defaults `0 L`.
- `interiorCapacity` / `surfaceCapacity` — `Quantity<'L'> | null`;
  `null` is an **uncapped** slot (a puddle). Authored only when bounded
  (omit ⇒ uncapped — the persistence layer skips absent fields).
- `closure` — the retention scale (below). Default `liquidTight`.

An **inexhaustible** source (a slot you can draw from forever that never
depletes — the coffee urn) is NOT a flag on the base substrate. It is a
focused capability mixin, `UnboundedSourceMixin` (`lib/bulk/UnboundedSource.ts`),
composed only on source fixtures (`obj/UnboundedReceptacle`). It
overrides three generic slot-policy seams the base exposes —
`getBulkAvailable` (→ `∞`), `isBulkEmpty` (→ false while filled), and
`debitBulk` (→ no-op) — so the base `Bulkable` interface carries nothing
source-specific. The richer regenerating-well model (scheduled refill
over game-time) is deferred; `∞` is the simpler demonstration.

Canonical storage unit is **`L`** ({@link BULK_VOLUME_UNIT}). Authored
or player-typed `cup` / `mL` measures convert to litres at the boundary
(`Quantity.parse` / `Quantity.to`); the converters live in
`lib/quantity.ts`. `cup` / `mL` are tagless volume units (like `m³`).

### `BulkSlot` — the live handle

`host.getBulk(affordance?)` returns a `BulkSlot` handle that reads and
writes the host's flat fields. With no argument it returns the single
present slot and throws when the holder has both or neither. The handle
is what `BulkableApi.transfer` operates on (and accepts `null` for the
discard sink — `drink`):

- `available()` — litres to draw FROM (`∞` for an inexhaustible
  source — see `UnboundedSourceMixin`).
- `remaining()` — litres of headroom to pour INTO (`∞` when uncapped).
- `isEmpty()` — no material, or non-positive amount (an unbounded
  source with a material is never empty).
- `getMaterial()` / `getAmount()` / `getCapacity()` / `getClosure()`.
- `setMaterial()` / `setAmount()` — the low-level primitives `transfer`
  composes (the bulk analog of `Globbable.setQuantity`).

### Closure scale — gated retention

A per-vessel ordered `closure` level governs liquid retention. It is the
vessel's inherent **construction** (a steel bucket vs a steel sieve), so
it can't derive from material; it lives on the `Bulkable` holder, **not**
on `Sealable` (which is a dynamic lid state, orthogonal and out of this
slice):

```
open < liquidTight < sealed        (default: liquidTight)
```

A holder retains matter when `closure ≥ requiredClosureFor(material)`.
v1 bulk is all liquid (`requiredClosureFor → 'liquidTight'`), so an
`open` vessel doesn't retain it — it **drains through** (below).
`sealed` (gas) and the phase→required-level mapping are defined on the
scale but unexercised until gas content lands.

## `BulkableApi.transfer` (`api/bulk.ts`)

The one primitive — the bulk analog of `GlobbableApi.applyQuantity`
(not of `split` / `merge`). It is **ungated** so controllers call it
directly; the raw slot writes it composes are the low-level layer.
Programmatic-contract violations throw; user-input failures ride the
structured-notes envelope, reusing glob's canonical `@saxonberg/types`
note kinds (`quantity-clamped`, `quantity-clamped-rejected`,
`empty-result`, `target-declined`) — **no new note kinds**.

```
transfer(from: BulkSlot, to: BulkSlot | null, amount): TransferResult
```

`amount` is a `TransferAmount`: `{ kind: 'all' }` (the whole source,
clamped to the destination's room — `drink` / `spill` / default `pour`)
or `{ kind: 'measure'; litres; mode }` (a specific volume — `fill`-to-
capacity, `sip`, `pour 2 cups`). Pipeline:

1. **Empty / unresolved source** → `empty-result`, declined.
2. **Material compatibility** — a non-empty destination must hold the
   same material → `target-declined { reason: 'material-mismatch' }`,
   declined. (Mixing is deferred.)
3. **Closure** on an `interior` destination — if its closure can't
   retain the matter, **drain through** to the floor's surface puddle
   (one level of redirection, not recursive); status `'drained'`,
   `drainedTo` set. Surface destinations always accept.
4. **Clamp** `applied = min(requested, from.available(),
   to.remaining())`. A measure shortfall: `strict` rejects entirely
   (`quantity-clamped-rejected`); `lenient` moves what fits
   (`quantity-clamped`, status `'partial'`).
5. **Apply** — debit the source (skipped for an unbounded source; a
   bounded slot that hits zero clears its material), credit the
   destination (adopting the material when it was empty); a `null` sink
   just discards.

Verb-facing helpers on the same Api:

- `slotFor(holder, affordance)` — resolve a holder's slot from
  `model.field.via?.bulk?.affordance`.
- `amountFromQuantity(quantity, fallback)` — translate an MQL measure
  hint to litres (via the converters).
- `floorSurfaceNear(near)` — walk the containment chain to the
  location's floor surface slot (the `spill` target and the
  drain-through destination).
- `floorPuddleSummary(location)` — the room-view puddle line for
  `LookController` (the floor is excluded from the contents list).
- `ingest(actor, material, litres)` — hand consumed matter to the
  actor's `ingest` seam.

## Material identity (`lib/material/Material.ts`)

`Material` composes `PerceptibleMixin` (a keyword pool) and carries an
`appearance` phrase. It does **not** compose `Visible` / `Named` —
substance identity stays out of the perception-target machinery, so
material keywords never leak into room scope.

- **keywords** drive material-keyword resolution (`drink coffee` finds
  the holder).
- **appearance** is the rendered phrase, composed two ways: the verbs
  speak it (`You drink the dark, steaming coffee.`), and it rides a
  holder's **description**.

Demo Materials (`coffee`, `water`) are **flat** — appearance + keywords
only, no composition / chemistry depth (fidelity is demand-driven;
nothing in this slice reads past appearance + keywords).

### `BulkPayload` — the per-instance blend face

A **derived mixture** (a plated stew, a mixed cocktail — the crafting
branches' blends) can't be a Material row without making the material
library boundless (the fixed-vocabulary rule, [race.md](./race.md) §
Material). So each slot optionally carries a **`BulkPayload`** — a
plain persisted record with a Material row's identity + metabolism
face (`name`/`appearance`/`keywords`, `nutrients` routing tags, label
`nutrientAmounts`, per-serving `toxicity`, `edible`) — while the slot's
material stays ONE generic substance (`food/cooked`,
`cocktail/mixed`). Every reader treats `payload ?? material`
uniformly: the MQL bulk candidate (`look stew`), the contents
augmenter, the NutritionLabel, the drink/sip prose, and metabolism's
`ingest`. Cleared whenever the slot empties; `transfer` carries a copy
into an empty destination (same-material pours into a non-empty vessel
keep the destination's payload — blend *merging* is out of scope). A
payload-less slot behaves byte-identically to before.

### Description composition

`BulkableMixin` contributes a `markupAugmenter` (the same
`getMarkupLong` pipeline `DetailedMixin` rides), so a non-empty slot's
contents surface wherever a holder's long description renders:
`look thermos` → "… It holds dark, steaming coffee."; `look floor` → "…
A puddle of water pools here." A puddle also surfaces in the bare `look`
room view via `BulkableApi.floorPuddleSummary(location)`, since the
floor is an `Adornment` excluded from the room's contents list. (Making
the floor's contents the *primary* room presentation is the deferred
containment-inversion, not this — see non-goals.)

### `getContentsDescriptionFor(viewer, affordance?)` — the one viewer-aware read

Bulk is otherwise **viewer-blind**, and rightly so: a puddle of water is
a puddle of water to everybody. The augmenter reads its phrase from this
method, which resolves in two tiers:

| Slot holds | Phrase |
|---|---|
| an **identifiable** material | `RecognitionApi.describe(viewer, material)` — per-viewer |
| anything else | `payload?.appearance ?? material.appearance` — the shipped read, unchanged |

The case that earns the tier is the **potion**: what a draught *looks*
like and what it *is* are different facts, and which one you get depends
on what you have learned. Identity rides the **material**, not the
flask, so one identification covers every flask of that substance and
decanting carries the knowledge with the liquid — see
[magic-items.md](./magic-items.md) § Identification. `MagicLogic`'s
identify effect makes the **same** redirect, so *looking at* and
*identifying* a vessel agree about what the subject is.

Two article rules follow from the two tiers, and they are not
interchangeable: the identification phrase is a **full noun phrase**
carrying its own article ("an iridescent crimson potion"), while a plain
material's `appearance` is a **bare phrase** the verb articles itself
("You drink **the** dark, steaming coffee."). `DrinkController` branches
on `MixinApi.isIdentifiable(material)` for exactly this reason; branching
on "did a phrase come back" silently drops the "the" from every ordinary
drink.

`null` for an empty slot, a slotless host, or an ambiguous affordance —
the description pass runs on every render, so the boring cases are
silent rather than throwing.

## MQL surface

A holder's bulk is a non-Stuff facet reached exactly like a `Detail`:
the **holder** Stuff lands on the target field, and a `via.bulk` facet
(declaration-merged onto `MqlMatchVia`, colocated in `Bulkable.ts`)
marks "you reached this holder through its bulk":

- **`:b` transform** (`thermos:b`) — keeps the holder, stamps
  `via.bulk = { affordance: 'interior' }`; capability-gated (drops
  non-holders), like `:i`. `:B` is unallocated (a bare `B` lowercases to
  a `'b'` keyword filter → empty).
- **material-keyword** (`drink coffee`) — `scope-walk.pushBulkMaterials`
  emits one candidate per non-empty slot scored on the Material's
  keywords, the holder as the matched Stuff, `via.bulk` set.
- **`MqlQuantity` measure variant** — `value.kind: 'measure'` carries a
  serializable `{ value, unit }` (the `Quantity.toJSON` shape, not a
  live `Quantity`).
- **`:{N unit}` formal** (`water:{2 cups}`) — `parseQuantity` reads a
  unit bareword after the integer via `QuantityApi.resolveUnitToken`.
- **natural-language measure** (`pour 2 cups water`) — a `desugar`
  branch consumes `<int> <unit>` before the bare-count capture
  (prep-split happens first in the matcher, so desugar only ever sees
  the field's own text). The unit lexicon lives on `QuantityApi`
  (`resolveUnitToken` / `isUnitToken`) — one home for both parser and
  desugar, no free module.

## The Floor (`obj/Floor.ts`)

The floor composes `BulkableMixin` with a `surfaceBulk` slot, so a
spilled, over-poured, or drained-through liquid pools as the floor's
**surface** bulk (a puddle). The floor stays an `Adornment` fixture
(excluded from the room's enumerated contents), **NOT** `Surfaced`:
discrete containment is untouched — an apple dropped in the room is
still `container = room`, a sibling of the desk, not `restingOn` the
floor. A puddle is the floor's attribute, not a Stuff. The default-floor
seed (`seeds/obj/surface/default-floor.yaml`) carries an uncapped
surface slot, so any room with a floor can pool.

## Verbs (`mud/cmd/bulk/`, `obj/command/bulk/`)

A `bulk/` command category — `fill` / `pour` / `spill` / `drink` /
`sip` — each a thin direction over `transfer` (`void` + envelope, no
`success` flags). The verbs are **carried by the holder** (the
Thermometer-carries-`measure` pattern): `BulkableMixin` contributes them
in the `inventory` / `peers` / `environment` buckets, so they light up
whenever a Bulkable is reachable — not minted from a bespoke verb mixin.

| Verb | Shape | Amount |
|---|---|---|
| `fill X from Y` | fill to capacity | measure = X's remaining headroom |
| `pour X into Y` | holder → holder | the source measure hint, else all |
| `spill X` | holder → floor surface | all |
| `drink X` | holder → discard sink | all; fires `ingest` |
| `sip X` | holder → discard sink | a fixed small measure; fires `ingest` |

### The ingest seam (`lib/creature/Creature.ts`)

`drink` / `sip` hand the consumed `{ material, amount }` to
`Creature.ingest(material, amount)` (via `BulkableApi.ingest`). v1 is a
deliberate **no-op** — the socket exists; nothing is plugged in. A
future `Metabolic` / `Digestive` capability overrides it. Per-entity
method, not a registry (substrate has no content hooks).

The [respiration](./respiration.md) **air tank** (`obj/AirTank.ts`, a worn
`Bulkable` whose `interior` is `air`) is another consumer of this surface:
the body taps the tank with `BulkSlot.debit` as a depleting carried supply
while submerged, and a depleted tank is refilled via `BulkableApi.transfer`
from an air source — no new bulk machinery.

## Demo content

- Materials: `coffee`, `water` (`seeds/lib/material/bulk/`).
- Holders: one `Receptacle` class (`obj/Receptacle.ts` =
  `BulkableMixin(Thing)`, fluid-only — not a discrete `Container`, and
  named to stay clear of the existing enterable-`Vessel`) backing four
  seeds (`seeds/obj/vessel/`): the unbounded coffee `urn`, the portable
  `thermos`, the destination `mug`, and the `open`-closure `colander`.

## Deferred tails

Each lands in a named home later; none is in this slice.

- **Mixing / solutions** — same-material accumulation only; cross-
  material is rejected, not blended.
- **The `sealed` / gas closure level** and the per-`Material` phase →
  required-level mapping (gas → `sealed`, granular → `open`).
- **The "everything is on a surface" containment inversion** — this
  build adds surface-*bulk*, not the discrete-resting refactor.
- **`Container` + `Bulkable`** (ice cube in water) — documented, not
  built; the demo vessels are fluid-only.
- **Universal auto-compose** into Containers / Surfaces.
- **Amount-aware `appearance`**, capacity↔collision unification,
  food-prep conversion verbs, thermal (hot coffee staying hot).
- **Consumption consequences** — the `ingest` seam stays a no-op until
  Dave's bar.

## File map

| File | Role |
|---|---|
| `lib/bulk/Bulkable.ts` | mixin + `BulkSlot` + closure scale + `via.bulk` + `requiredClosureFor` + `getContentsDescriptionFor` |
| `lib/bulk/UnboundedSource.ts` | `UnboundedSourceMixin` — inexhaustible-source override (urn) |
| `obj/UnboundedReceptacle.ts` | the urn's class (`UnboundedSourceMixin(Receptacle)`) |
| `api/bulk.ts` | `BulkableApi.transfer` + `slotFor` / `amountFromQuantity` / `floorSurfaceNear` / `floorPuddleSummary` / `ingest` |
| `obj/command/perception/LookController.ts` | room-view puddle line (the only non-bulk file the build touches) |
| `lib/material/Material.ts` | `PerceptibleMixin` + `appearance` |
| `lib/quantity.ts` | `mL` / `cup` units + converters |
| `api/quantity.ts` | `resolveUnitToken` / `isUnitToken` |
| `api/mql/{lexer,parser,resolver,desugar,scope-walk,types}.ts` | `:b`, material-keyword, `:{N unit}`, measure variant |
| `obj/Floor.ts` | surface-bulk on the floor |
| `mud/cmd/bulk/`, `obj/command/bulk/` | the verb roster |
| `lib/creature/Creature.ts` | the `ingest` seam |
| `seeds/lib/material/bulk/`, `seeds/obj/vessel/` | demo content |

## Cross-references

- [glob.md](./glob.md) — discrete sibling; reused notes + result-slot
  patterns.
- [quantities.md](./quantities.md) — `Quantity<U>`, unit catalog,
  marshallers (volume units added here).
- [race.md](./race.md) — `Material` (gains `PerceptibleMixin` +
  `appearance`).
- [mql.md](./mql.md) — `MqlMatchVia`, the `via.detailPath` precedent,
  the `:b` transform home.
- [response-envelope.md](./response-envelope.md) — note kinds.
- bulkable-slate.md — design of record.
