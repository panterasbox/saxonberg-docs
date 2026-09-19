# Encumbrance

The encumbrance subsystem answers three questions about a body that
carries things: **how much is it bearing**, **how much can it bear**, and
**what happens when it bears too much**. It is the first *driver* built on
the vitals substrate — the first system that reads vital state
(`endurance`, the condition band) and pushes consequences back into the
world (a lift declines, a climb is vetoed, endurance drains).

The whole gauge is **derived-on-read** — nothing is stored. A bearer's
burden and capacity are recomputed every call from its contents, its worn
/ held items, its body mass, and its physiology.

## The gauge — `LoadBearingMixin`

`lib/encumbrance/LoadBearing.ts` is the single home for the capability:
the gauge, the consequence helpers, the recursive tree-walk (a private
module function), and the engine dials (`LOAD_BEARING_DEFAULTS`, a
top-of-file `const` — the `Vitals` precedent, not a separate Api or a
registry). There is **no `EncumbranceApi`**.

`LoadBearingMixin` **declares and runtime-enforces** its composition
requirement: a load-bearer is `Container + Slotted + Tangible + Reserved +
Vitals`. It reads all five, so it composes **outer** of all of them in the
`Creature` stack (the same placement logic as `Vitals` outer of
`Reserved`). Today only `Creature` assembles the prerequisites, but the
door is open — a future non-`Creature` bearer composes the mixin, not the
class tree. Drivers narrow on `MixinApi.isLoadBearing(actor)` and call the
surface directly:

| Method | Meaning |
|---|---|
| `getBorneBurden(): Quantity<'kg'>` | the weighted weight currently borne |
| `getCarryCapacity(): Quantity<'kg'>` | what the body can comfortably carry |
| `getStrainCeiling(): Quantity<'kg'>` | the absolute lift cap (`capacity × OVERLOAD_FACTOR`) |
| `getLoadRatio(): number` | `burden / capacity` (dimensionless) |
| `wouldExceedCeiling(candidate): boolean` | the lift-gate predicate |
| `drainForTraversal(): void` | the loaded-traversal endurance drain |

### Borne burden — the weighted tree-walk

`getBorneBurden` walks **both** carried stores and sums a weighted
contribution per item. Missing either store would silently undercount, so
both are walked:

- **General contents** (`Container.contents`) — items carried *loose*.
- **Slot occupants** (`Slotted.getAllOccupants()`) — worn / wielded items.

Two coupling factors shape the sum:

- **Transmission** — a running *product* of every enclosing
  `Vessel.transmissionFactor` (default `1.0`) down to an item. A bag of
  holding (`transmissionFactor ≈ 0.05`) multiplies its whole subtree
  toward zero. Only a `Vessel` attenuates (it owns the field); a
  non-`Vessel` container passes weight through at `1.0`.
- **Placement** — a per-attachment surcharge fixed at the top-level
  attach point and carried down the subtree unchanged. **Derived from the
  slot's existing `accepts`**, never stored on `SlotSpec`: a held slot
  (`WieldableMixin`) couples at `LOOSE_CARRY_SURCHARGE`; a worn slot
  (`WearableMixin` / anything else) at the `1.0` floor. A backpack worn on
  the torso applies the worn floor to itself *and* its contents; a chest
  held in a hand applies the held surcharge to both.

Safety: a `visited` set (by `stuffId`) plus `MAX_DEPTH` guard against
cycles and pathological nesting — containment is a DAG via the move
chokepoint, but the walk does not assume it. The same item appearing in
both stores is counted once. The walk reads only the method surface
(`getMass` / `getContents` / `getAllOccupants` / `getSlotSpec` /
`getTransmissionFactor`), never fields.

#### The v1 loose-carry model

`get` does **not** claim a hand slot (building a hand-slot mechanic was out
of scope). Instead, items in a bearer's general `Container.contents`
(carried loose, not in any slot) are treated as carried-in-hand and pay
the **held surcharge** — the same `LOOSE_CARRY_SURCHARGE` a `hand:*` slot
would. This honors "carrying loose is a surcharge" without a new slot
mechanic. Worn / wielded items (in `slots`) get their slot's coupling. A
worn `Pack` is therefore strictly better than carrying the same load
loose: the contents drop from the held surcharge to the worn floor, and
the hands stay free.

### Carry capacity — physiology-derived

```
capacity = bodyMass × CAPACITY_FRACTION × conditionBandMargin × enduranceMargin
ceiling  = capacity × OVERLOAD_FACTOR
```

- **`bodyMass`** is the bearer's `getMass()`, seeded from
  `BodyPlan.baseMass` (see below).
- **`conditionBandMargin`** maps the derived `getConditionBand()`
  (`healthy → 1.0`, down through `critical`) to a multiplier ≤ 1.0 —
  injury lowers what a body can bear.
- **`enduranceMargin`** is the **spiral** term: the `endurance` reserve's
  `current/capacity` fraction maps to `[ENDURANCE_FLOOR, 1.0]`. Low
  endurance shaves capacity — and over-capacity loads drain endurance —
  but never to zero (the floor is "exhausted", not "can't lift a feather").

Both margins pull capacity *down* off the body-mass baseline. Environmental
margins (spo2, gravity) are deferred — nothing drives them off baseline
yet.

## Body mass — `Species.getBaseMass()` + `Creature.getMass()`

`BodyPlan` carries an authored `baseMass` (plain kg number) and, since
the textiles build, a sibling `baseStature` (m). A `Creature` seeds its
`getMass()` lazily, the first time mass is read: `Creature.getMass()`
returns an explicitly-authored mass unchanged (the deviation wins), else
adopts `species.getBaseMass()`, else stays at `0` (sessile / no plan —
graceful, no throw). The "is mass still 0?" check is the idempotency
guard; there is no persistent "seeded" flag.

> ⚠⚠ **The resolution is one layer deeper than it was.** It reads
> **`Species.getBaseMass()`**, which resolves *own → plan → 0* —
> **not** `bodyPlan.getBaseMass()` directly.
>
> Before that, no species overrode `biped`'s 70 kg, so all ten playable
> species massed the same: a halfling and a dragonborn carried the same
> weight, punched with the same energy, ate the same, and cooled at the
> same rate. Fit cannot key on a constant, which is what forced the fix.
>
> ⭐ **Three production reads changed and only three** —
> `Creature.seedMassFromBodyPlan`, combat's mass-scaled fist, and the
> natural-attack `largeBody` threshold. Everything else (capacity here,
> metabolism's basal drain, `Thermal.getTau`) reads
> `Creature.getMass()` and inherited it for free. That is the point of
> the one accessor: lineage will later vary mass on the **individual**
> without either layer changing.

The seed runs at read-time rather than from a post-hydrate hook because the
`postRegister` chain is not uniformly threaded below `CommandGiver`
(`Avatar.postRegister` does not `super`-call), so a `getMass()` override is
the one seam that makes mass honest for *every* reader of *every* Creature
subclass.

> `baseMass` is a **shared body-size signal**, not encumbrance-private.
> Metabolism reads it for basal drain and thermal for the body's thermal
> mass; `BodyPlan` has since gained the sibling `thermalStrategy` field
> (endotherm / ectotherm) the thermal build added.

## `Vessel` — the container-object

A bag, box, chest, cart, and ship are one category — *a `Thing` that holds
things* — at different **scales**; carry / drag / ride / can't-budge is
emergent from mass vs. capacity, never a type flag. `Vessel` is **already
`Tangible`**, so a carriable container works mechanically today. This build
added:

- **`transmissionFactor`** (field, default `1.0`, validated `0..1`) — the
  attenuation the burden walk reads. A bag of holding sets it low.
- **`Adornable` narrowed off the `Vessel` base onto `ExitableVessel`.**
  Fixtures (`getFixtures()`) are needed only by the Door →
  `BoundaryAnchor` retrofit, an `ExitableVessel` concern. Every fixture
  consumer already narrows on `MixinApi.isAdornable`, so a bare `Vessel`
  being `!isAdornable` is observably identical to the old
  Adornable-with-empty-fixtures state.

The deferred per-item refinement — a hip-belt frame pack that couples
*better* than a bare worn slot — rides on the **item** (like
`transmissionFactor`), never on `SlotSpec`.

## The consequence ladder

Encumbrance is an **actor-experience** concern, not a physics-of-
containment concern. The raw move / containment substrate
(`Mobile.traverse`, `ContainmentApi.move` / `placeOn`, `forceMove`) carries
**zero** encumbrance code — a dev or script moving a 200 kg anvil onto a
sparrow Just Works, no block, no drain. The consequences live in the layers
that own the actor experience:

- **Lift gate** → `GetController.pickUpOperand`. Before the move, if the
  giver `isLoadBearing` and `wouldExceedCeiling(operand)`, decline
  diegetically — a `controller-rejected` note (`reason:
  'too-heavy-to-lift'`) + a scene line ("you strain, but it doesn't
  budge"). The item is skipped (a `get all` still picks up lighter items).
  An item over *capacity* but under the *ceiling* still lifts — the body
  is now overloaded (locomotion-gated, drains on traverse), but the lift
  succeeds.
- **Locomotion veto** → `LocomotionApi.canTraverseExit`
  (`#checkEnablementScope`), *alongside* the existing capability check. An
  overloaded body (`getLoadRatio() ≥ HEAVY_LOAD_THRESHOLD`) is refused
  climb / swim / fly with `gate: 'encumbrance'`. Walk has no enablement
  mixin, so it is never gated — load taxes endurance, not pace, and gates
  *modes* (movement-speed effects are a non-goal). **Open seam:** the veto
  could relocate to `ExitableMixin` / the boundary; the predicate moves
  unchanged.
- **Traversal drain** → `LocomotionApi.engageAround`, after a *successful*
  self-powered traverse. `engageAround` is the universal self-powered
  chokepoint, so a loaded body tires whatever initiated the step (player
  command, NPC brain, follow/lead automation). Conveyance riders
  (repositioned by the vehicle's ripple) and raw / dev / `forceMove`
  traverses don't reach `engageAround`, so they never drain — the
  walked-vs-rode exclusion is **structural, not a coded check**. Light
  loads (ratio ≤ `LIGHT_LOAD_FLOOR`) cost nothing.

`Mobile.traverse` and `LocomotionControllerBase` are **not** modified — any
edit to `Mobile.traverse` for encumbrance is a red flag (it would re-couple
the primitive and force walked-vs-rode / skip-forced checks into it).

### Recovery is deferred (v1 drain is one-way)

Endurance **recovery** is a metabolism concern, not an encumbrance one:
encumbrance *drains* a reserve; how a reserve *replenishes* is metabolism
(the convergence of rest + fuel + fatigue clearance). So v1 ships **one-way,
gentle** drain — a body that hauls over-capacity loads loses endurance and
stays diminished until the metabolism build closes the loop with **coupled
recovery** (endurance refilling by burning satiation + hydration,
posture-gated by a rest-quality factor on the furniture, on the in-session
metabolic clock). The `endurance.floorEffect = 'collapse'` likewise stays
**unconsumed** here — metabolism's exhaustion cascade is its designed
driver.

## Observability

The gauge surfaces through the existing `subscribableFields` projection
(the `Tangible.mass` precedent) — the live-query / inspection substrate
reads `borneBurden` / `carryCapacity` (`{ value, unit: 'kg' }`) and
`loadRatio` (number) off any bearer. These are derived reads, no event
wiring in v1.

## Engine dials — `LOAD_BEARING_DEFAULTS`

Numeric tuning is a non-goal this build, so the dials are named, greppable,
and retunable in one record at the top of `LoadBearing.ts`:
`CAPACITY_FRACTION`, `OVERLOAD_FACTOR`, `HEAVY_LOAD_THRESHOLD`,
`LOOSE_CARRY_SURCHARGE`, `LIGHT_LOAD_FLOOR`, `DRAIN_PER_TRAVERSAL`,
`ENDURANCE_FLOOR`, the `CONDITION_BAND_MARGIN` table, and `MAX_DEPTH`. They
live in the capability's own module; **GameConfig is the eventual home**
when it lands.

## Demo content

Three real templates under `generic-objects/content/stuff/thing/gear/`, each exercising
one axis:

- **`backpack`** (`/lib/equipment/Pack`, a wearable `Vessel`) — placement
  coupling: a stowed load counts at the worn floor, hands free.
- **`bag-of-holding`** (a plain `Vessel`, `transmissionFactor: 0.05`) —
  transmission: 100 kg of cargo barely registers.
- **`anvil`** (a heavy `Thing`, 200 kg) — the lift gate: over the strain
  ceiling, `get anvil` declines.
- **`handcart`** (`/lib/equipment/Handcart`, a `Haulable` `Vessel`,
  `draftFactor: 0.04`) — the draft term: 300 kg of cargo hitched behind
  you costs ~9 effective kg.

`biped.yaml` authors `baseMass: 70` and `baseStature: 1.75` (a human-ish
body) — the defaults every species that authors none inherits. The ten
playable species all override; see [race.md](./race.md) § *Size*.

## Haulage — the cart (the draft term)

A **cart** is a wheeled `Vessel` you hitch and pull. It is the encumbrance
consumer the `Vessel` section anticipated (*carry / drag / cart /
can't-budge emergent from mass vs. capacity*) and the "cart hinge" the
slate deferred. The relationship, the verbs, and the tow live in
[conveyance.md § Haulage](./conveyance.md#haulage--pulling-a-cart); the
**encumbrance side is one term**.

Cargo loaded into a cart lives in the *cart's* container, which the burden
walk never visits — so offloading onto a cart removes the weight **for
free**. The only cost is moving the cart itself, modeled as the
pushing-side analog of `transmissionFactor`:

```
draftLoad = (cartSelfMass + cartEffectiveContents) × draftFactor
```

`draftFactor` (`0..1`, on `HaulableMixin`) is the rolling-resistance ×
mechanical-advantage coupling — wheels turn carry-weight into a small
push-weight (a four-wheel cart ~0.03; a dragged sledge ~0.35; carried in
arms 1.0). `cartEffectiveContents` runs the cart's contents through the
**same** weighted walk (a bag of holding inside a cart still weighs ~0).

While a bearer is hitched, `getBorneBurden` adds `draftLoad` as **one
extra term** (read dynamically via `MixinApi.isHauling` so the `Creature`
base carries no haulage dependency — a non-hauler bearer skips it). The
**entire consequence ladder then reuses unchanged**: the draft raises
`getLoadRatio`, gates climb/swim/fly, and drains endurance on
traverse — though a well-wheeled cart's draft usually sits under
`LIGHT_LOAD_FLOOR`, so hauling it costs no endurance (only a sledge /
overload / bad coupling taxes you). Two **move-time gates** (in
`LocomotionApi.canTraverseExit`, the veto layer allowed to read
encumbrance — *not* the raw move) are haulage-specific: **terrain** (the
cart's `wheeled` passage vs the exit) and **breakaway** (`draftLoad` over
the strain ceiling → "won't budge"; the ceiling is endurance-shaved, so a
tiring hauler can hit it). See
[conveyance.md § Haulage](./conveyance.md#haulage--pulling-a-cart) and
[locomotion.md](./locomotion.md).

## Deferred tails

- **Recovery + survival ticks + the `collapse` cascade** → the
  **metabolism** build (slate written; the seam above is concrete).
- **The spo2 capacity-margin** → the **respiration** build (slate written).
- **Cart / conveyance carry handoff** — **shipped** (the draft term
  above; the relationship in [conveyance.md § Haulage](./conveyance.md#haulage--pulling-a-cart)).
- **Environmental (gravity) margins, tissue-derived mass,
  augment-conferred capacity, numeric tuning, the per-item placement
  refinement (a frame pack beating the worn floor)** — genuinely undriven;
  land when a consumer appears.
