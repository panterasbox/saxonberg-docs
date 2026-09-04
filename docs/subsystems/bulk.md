# Bulk — continuous matter as a holder attribute

The substrate for **continuous, formless, measured matter** (liquid in
v1) — the **`Bulkable`** holder attribute: water, coffee. Bulk is the sibling of the discrete fungible-stack
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

## ⭐ `category` — the vessel kind, and the tie between an empty and a product

A vessel declares **what kind it is**: `coupe`, `can`, `keg`, `sack`,
`spirit-bottle`. It is a property of the vessel, independent of what is
in it — a coupe is a coupe whether it holds a martini or nothing.

⚠ **It lives on `VesselKindMixin` (`lib/bulk/VesselKind.ts`), NOT on
`BulkableMixin`.** It rode the volume mixin until the bulk decomposition,
which handed a par key to every floor puddle, garden bed, plant pot, air
tank and watering can in the game. It is composed by the classes that
actually have a kind — `Bottle`, `Vat`, `CraftVessel` and everything
below them — and `MixinApi.isVesselKind` is how a reader asks.

It exists because **template inheritance does not exist**, so the empty
vessel row (`/trade/bottling/thing/can`) and the product row that is
that vessel filled (`…/can-of-cola`) are otherwise strangers that happen
to share a class. The shared `category` string *is* the relationship,
and three things read it:

- **The census** ([residency.md](./residency.md)) — an emptied vessel
  counts under `vessel:<category>`, so a drained can of cola joins the
  factory-fresh empties instead of hiding under `vessel:cola`. That is
  the count a deposit or returns market reads, and it is why draining
  the world's gin makes the floor genuinely short.
- **The pool** ([crafting.md](./crafting.md)) — `claimGlass` takes any
  clean empty of the right kind. A washed-out vessel and a new one are
  the same input to a fill, which is what a real line does.
- **The par sheet** ([employment.md](./employment.md)) — glassware and
  kegs are counted by kind, not by product.

Authored on **both** the vessel row and every product row over it. A
holder that declares none falls back to its primary keyword, which is
right for a fixture whose interior is permanent (a plant pot's soil).

### ⚠ OPEN — the word "vessel" is squatted by the enterable class

The vocabulary this section defines is called the **vessel kind**
everywhere (`outputVesselKind`, the `vessel:` census prefix, the
archetype `vesselKind` need) — but the bare class name `Vessel` is the
ENTERABLE container (a boat, a wagon; conveyance-domain), which is
arguably the squatter on the word. A nothing-is-legacy rename of that
class (to something conveyance-flavored) would free the term; parked
here so it isn't lost (surfaced in the fermentation MR review).

### ✅ RESOLVED — `category` now has a home a `Crate` can reach

> Was: *"`category` lives on `BulkableMixin`, so only bulk holders carry
> it. `Crate` is a `Container` and not `Bulkable`, so authoring
> `category: crate` would be silently discarded."*

⭐ The bulk decomposition moved the kind onto its own
**`VesselKindMixin`**, which depends on nothing. A `Crate` can compose it
and author `category: crate` whenever the census wants that convergence —
the blocker was never the crate, it was the kind living on the volume.
The rest of this section stands as the reason to bother.

The consequence is narrow but real: an emptied crate derives
`vessel:<primaryKeyword>` (`vessel:grapefruits`), so **empties do not
converge across a Container holder's kinds** the way empty cans converge
on `vessel:can`. Convergence is the whole point of the vessel kind — a
produce crate is a returnable transport item exactly as a can is one — so
this wants fixing before anything counts or trades `vessel:*`. Nothing
does today, which is why it is deferred rather than urgent.

⚠ **`DetailedMixin` is NOT the home** (ruled out 2026-08-30). Detailed is
about descriptions and detail keys; the vessel kind is a structural fact
about the holder, and parking it there because the three classes happen
to share it would be an accident of composition, not a design.

The real candidates, for whoever picks this up:
- **A small dedicated mixin** composed by `Bottle`, `CraftVessel` and
  `Crate`. A mixin for one field is heavy, but the field *is* the
  concept, which is the bar this codebase sets.
- **Reconsider whether the pool-claim needs `category` on non-Circulating
  vessels at all.** `CirculatingMixin` already owns `censusKey`,
  `isEmptyHolder()` and `holderKind()` — it would be the natural home
  except that a bar's glasses (`CraftVessel`) are not Circulating. If
  they should be, the field has a home already and this dissolves.

Whichever way: it ripples through the pool-claim (`claimGlass`) and the
par sheet (`stockSheetFor`), so it is a change, not a row edit.

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
   just discards. A fresh fill (empty destination) also carries the
   **batch's identity** — a Graded source holder stamps the
   destination's grade band, and a Crafted source stamps the maker's
   mark (maker/recipe/craftedAt) too. The rule mirrors the payload
   rule exactly: identity rides into an *empty* destination only; a
   top-up keeps the destination's own identity (the fermentation grade
   seam, D6).

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

### `BulkPayload` — what a blend IS, and nothing else

A **derived mixture** (a plated stew, a mixed cocktail) can't be a
Material row without making the material library boundless (the
fixed-vocabulary rule, [race.md](./race.md) § Material). So each slot
optionally carries a **`BulkPayload`**, while the slot's material stays
ONE generic substance (`/platform/idea/material/cooked`,
`cocktail/mixed`). Cleared whenever the slot empties; `transfer` carries a
copy into an empty destination. A payload-less slot behaves
byte-identically to no payload at all.

⭐⭐ **It carries only what cannot be derived.** It used to hold twelve
fields — a Material row's whole identity and metabolism face — which is
how a continuous-volume type came to name seven subsystems' vocabulary
and why `lib/bulk` imported `lib/metabolism`. The bulk decomposition
reduced it to facts nothing else can recover:

| field | why it cannot derive | declared in |
|---|---|---|
| `composition` | **what went in** — Material PATHS + servings. Everything below reads this. | `lib/bulk` |
| `recipeId` | which recipe made it. Not a property of the ingredients: the same inputs worked by a cook and a bartender are different makings. | `lib/bulk` |
| `cookedAtK` | the temperature the working REACHED. The heat-labile toxin kill depends on it. | `lib/bulk` |
| `appearance` | ⚠ a boundary: rendered by `bulkContentsAugmenter` in `lib/bulk`, which may not import `lib/craft` to ask the recipe. | `lib/bulk` |
| `keywords` | ⚠ how the blend is FOUND. A missed lookup is not a degraded reading — it is an object that stopped existing. | `lib/bulk` |
| `formedToxins` | toxins that AROSE (a spoiled batch's ptomaine). Nothing in the composition implies them. | `lib/metabolism` |
| `freshness` | live state: microbial load + its clock stamp. | `lib/material/Freshness` |

Everything else is **read**: the tastes and the tags (`BlendLabel` unions
the ingredients'), the whole nutrition label (`BlendLabel`, shares scaled
by servings — which is why the composition carries them), the name and
the discipline (`BlendIdentity`, off the recipe, falling back to the
Material).

⭐ **The last two rows are the interesting part: a value object cannot
compose a mixin, so the subsystems that own those facts declare their
fields by DECLARATION MERGING from their own folders.** That is the
payload-shaped equivalent of composing a mixin, and it is a technique the
tree already used — `Engaged`, `CombatSession`, `AbortReason` and
`Bulkable` itself (on the MQL types). It is why `lib/bulk` imports no
subsystem but `lib/material`, and `pnpm lint:imports` holds that.


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
seed (`generic-objects/content/stuff/thing/surface/default-floor.yaml`) carries an uncapped
surface slot, so any room with a floor can pool.

## Verbs (`content/platform/cmd/bulk/`, `platform/idea/cmd/bulk/`)

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
  rows (`generic-objects/content/stuff/thing/vessel/`): the unbounded coffee `urn`, the portable
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
| `platform/idea/cmd/perception/LookController.ts` | room-view puddle line (the only non-bulk file the build touches) |
| `lib/material/Material.ts` | `PerceptibleMixin` + `appearance` |
| `lib/quantity.ts` | `mL` / `cup` units + converters |
| `api/quantity.ts` | `resolveUnitToken` / `isUnitToken` |
| `api/mql/{lexer,parser,resolver,desugar,scope-walk,types}.ts` | `:b`, material-keyword, `:{N unit}`, measure variant |
| `obj/Floor.ts` | surface-bulk on the floor |
| `content/platform/cmd/bulk/`, `platform/idea/cmd/bulk/` | the verb roster |
| `lib/creature/Creature.ts` | the `ingest` seam |
| `base-library/content/stuff/idea/material/bulk/`, `generic-objects/content/stuff/thing/vessel/` | demo content |

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
