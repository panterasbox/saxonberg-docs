# Vitals subsystem

The basic-biology layer: a living body's state modeled honestly, so it
can be healthy, hurt, sick, exhausted, unconscious, or dead. This build
ships the **substrate** — the models, type systems, resolution chains,
and seams — and defers the *behaviors* (live progression, the death
driver, condition content, instruments, treatment verbs). Bodies that
*can* be sick/hurt/strong/depleted/dead; nothing that *makes* them so
yet.

Source: `lib/vitals/Vitals.ts` (the `VitalsMixin`),
`platform/idea/Condition.ts` (the condition type system),
`lib/reserve.ts` (the reserve substrate — see [reserve.md](./reserve.md)),
plus `Species.vitalProfile` and `BodyPlan.bodyParts`.

## The load-bearing decision: no stored health scalar

There is no `hp` field. "How hurt am I" is **derived on every call** from
the substrate — blood volume + vital signs + active conditions +
reserves. The accessible band (`healthy` / `hurt` / `serious` /
`critical` / `dead`) is a *rendered view*, never the source of truth.
This is the same move the HP scalar's removal forced everywhere:
differentiation is the product, so the substrate stays honest and the
familiar bar is a projection.

Corollary, stated plainly: **vitals are body-state, not agent-state.**
Agency is gated separately (animacy / consciousness), so corpses,
unconscious patients, and anesthetized patients are all just bodies with
full vital/anatomy state and reduced or zero agency — no special-casing.

## The `Creature` hierarchy

That corollary is reflected in the class hierarchy. The body layer sits
below the agent layer:

```
Stuff
 └─ Agent              (runtime active object)
     └─ Creature       (the BODY: Organism + Vitals + Reserved + anatomy
     │                   slots + posture + description + containment)
         └─ Character  (the AGENT: + commands, perception, speech,
         │              movement, engagement, persona/pronouns)
             └─ Avatar
```

`Creature` (`lib/creature/Creature.ts`) is **concrete** — a bare Creature
is a valid non-agent body (a frog, a corpse, a simple animal). It
composes `VitalsMixin` + `ReservedMixin` + the body mixins; `Character`
extends it with the agency stack. The body/agency split is **sex (body)
vs. gender/persona (social)**: `SexedMixin` (reads species biology) is on
Creature; `GenderedMixin` (pronouns) + `PersonaMixin` are on Character.
See [docs/architecture.md](../architecture.md) for the full mixin
partition.

`VitalsMixin`/`ReservedMixin` compose only onto animate biological
organisms (every Creature → Character → Avatar; a future bullfrog NPC is
a `Creature`). Non-animate species (the Plantae peace lily, the
Constructa tutor-bot) have no Stuff class and never become Creatures.

## `VitalsMixin` surface

`VitalsMixin` requires `OrganismMixin` (it reads `getSpecies()` for the
per-species band profile and `getLifecycleState()` for the dead
readout). The constraint is a **runtime guard**, not a comment.

```ts
interface Vitals {
  // Vital signs — Quantity per sign
  getVitalSign(sign: VitalSign): Quantity<Unit>;
  setVitalSign(sign: VitalSign, value: Quantity<Unit>): void;
  getVitalBand(sign: VitalSign): VitalBand;      // species profile / default

  // Derived readouts — computed every call, NEVER stored/cached
  getConditionBand(): ConditionBand;             // healthy…critical…dead
  getConsciousness(): Consciousness;             // conscious/unconscious/dead

  // Anatomy — resolves instance-delta → BodyPlan structure
  getParts(): ResolvedBodyPart[];
  getPart(key: string): ResolvedBodyPart | null;
  getMissingParts(): ResolvedBodyPart[];         // was getInjuredParts
  severPart(key: string): void;                  // the one writer of `missing`
  isSlotDisabledByAnatomy(slot: string): boolean;

  // ⭐⭐ The function axis — what a wound COSTS (see harm.md)
  functionAt(key: string): FunctionBand;         // min along the supply path
  capacity(k: BodyCapacity): FunctionBand;       // governs=min · serves=mean
  canGrip(slot: string): boolean;
  canBearWeight(): boolean;
  slotRefusalReason(slot: string): string | null;

  // Conditions — both kinds, one collection
  getConditions(): readonly ActiveCondition[];
  hasCondition(pred: (c: ActiveCondition) => boolean): boolean;
  afflict(condition: ActiveCondition): void;     // add
  relieve(condition: ActiveCondition): boolean;  // remove

  // Death seams (no driver)
  getCauseOfDeath(): string | null;
  setCauseOfDeath(value: string | null): void;
  getPostmortemProgressions(): readonly string[]; // ships []
}
```

## Vital signs + `vitalProfile`

Seven first-class `Quantity` fields, each with a `static fieldMarshaller`
(`QuantityMarshaller.pathFor(unit)`) and a per-sign survivable band:

| Sign | Unit | Notes |
|---|---|---|
| `coreTemperature` | `K` | fever ↑, hypothermia ↓ |
| `heartRate` | `bpm` | 0 → cardiac arrest |
| `respiratoryRate` | `bpm` | 0 → respiratory arrest |
| `bloodPressureSystolic` | `mmHg` | two fields — both clinically load-bearing |
| `bloodPressureDiastolic` | `mmHg` | |
| `spo2` | `%` | hypoxia below |
| `bloodVolume` | `L` | hemorrhage drains it; floor → lethal |

New `Quantity` units `bpm` / `mmHg` / `L` (with `mmHg ↔ Pa`, `L ↔ m³`
converters + tag tables) ship in `lib/quantity.ts` — see
[quantities.md](./quantities.md). Per-sign invariants (unit match,
non-negative) live on the `setVitalSign` setter.

Healthy baselines + survivable bands are **per-species**, read from
`Species.vitalProfile` (`{ baseline, survivableMin, survivableMax }` per
sign, a flat record — no marshaller, like `visionProfile`). Authored for
the Animalia roster (7 humanoids + the bullfrog, an ectotherm with
distinct bands). `UNIVERSE_DEFAULT_VITAL_PROFILE` (engine constant,
Homo-sapiens-shaped) backstops any animate species without one — mirrors
the sessile-bodyplan backstop. The profile shape reserves room for a
later age-curve.

## Anatomy + tissue (the load-bearing centerpiece)

Anatomy is load-bearing twice — the injury *site* (vitals) and the
strength/dexterity *substrate* (deferred physical attributes,
capability-magic-slate) — so it is modeled in full.

- **`BodyPlan.bodyParts: BodyPart[]`** — typed part descriptors declared
  ONCE on the shared `biped`/`quadruped` body-plan flyweight, parallel to
  `slots`. Each `BodyPart` carries `{ key, parent, tissues, governs?,
  serves?, severable?, innervatedBy?, suppliedBy? }`. Roster since the
  injury build: head / torso / limbs (+hands/feet) + **brain · spine
  upper/lower · heart · lungs · liver** — sixteen parts on a biped.
  - ⭐⭐ **`governs` vs `serves`, and they are not interchangeable.**
    `governs` means *this organ RUNS the thing* (combines by **min** — one
    brain) and makes the part **interior**. `serves` means *this limb is
    FOR the thing* (combines by **mean** — two legs) and does **not**:
    a hand that `governs`ed manipulation would be an internal organ,
    silently dropped from every covering walk. Both are validated against
    `VITAL_SIGNS ∪ BODY_CAPACITIES` at registration — a typo throws
    instead of producing an organ that runs nothing.
  - ⭐ **`BodyPlan.isInterior(key)`** is the one predicate (it replaced
    five inline copies of `part.governs?.length`): a part is inside you if
    it governs something **or** if some other part's `innervatedBy` /
    `suppliedBy` names it. That second clause is what lets the **spine** —
    which governs nothing and conducts everything — be interior without a
    field nobody else would read.
  - **`innervatedBy` / `suppliedBy` are authored only where the supply
    path DIVERGES from the tree.** For a limb the parent chain already is
    it; the function walk recurses, so naming the spine once at the arm
    carries to the hand.
- **Tissue composition** — each part carries named tissues with masses
  (`{ tissuePath, mass }`), not a single material. Tissues are authored
  Materials under `/stuff/idea/material/tissue/` (`flesh`, `muscle`, `bone`).
  The mass-per-tissue is the substrate a future strength reading
  aggregates — and, since the injury build, a LIVE input:
  `BodyPlan.partArea(key)` is Meeh's `mass^(2/3)`, which both the
  surface-fraction walk and the **depth ladder's ordering** read. An
  author tunes which organ a deep wound reaches first by authoring its
  mass, which is a physical fact they would author anyway.
- **Instance-delta resolution** — the instance carries only deltas
  (`VitalsMixin.bodyPartDeltas: Record<key, { missing? }>`); structure
  lives on the shared `BodyPlan`. `getParts()` walks
  instance-delta → BodyPlan-structure (the `getMaterial` / `getSpecies`
  resolution shape) and returns `ResolvedBodyPart` (structure + `missing`).
- **Slot↔part relations live on the slot, not the part.** Anatomy is its
  own axis; slots *reference* it. `SlotSpec.bodyPart` (a `body.*` key) is
  the attach/enable edge — a missing part disables the slot via a coarse
  consult in `SlottedMixin.canOccupy` (`MixinApi.isVitals` narrows the
  host; no-op unless the part is gone). `SlotSpec.covers` is the coverage
  edge (one slot → many parts, for future armor / hit-location — declared
  edge — four live readers via `getSlotsCovering`, and the outside-in
  covering fold walks it). `BodyPlan` validates both references
  (referential integrity) and exposes the reverse query `getSlotsAt` /
  `getSlotsCovering`. Parts stay pure anatomy — no slot knowledge. The
  organ→vital coupling stays on the part (`governs`, heart →
  `[heartRate, circulation]`). Non-anatomical affordances
  (a saddle surface, a cranial implant bay) carry neither edge — slots
  remain a distinct axis, not something anatomy owns.
- **Stable `body.*` keys** are the identity anchor everything downstream
  points at — trauma `site`, the couplings, and the deferred graph /
  part-promotion. Locked now.

**Live since the injury build:** the innervation/vascular graph
(`innervatedBy` / `suppliedBy` — read by `BodyPlan.isInterior` and by the
function walk) and the **sever** (`severPart`, the one writer of
`missing`). **Still deferred-with-seam:** part-as-Stuff promotion (a
severed limb is an absence, not yet an object; transplants, prosthetics);
MQL anatomy queries.

## Conditions — the three-kind type system

A condition is a discrete affliction on the body. All kinds present
behind one `ActiveCondition` collection (`getConditions` / `afflict` /
`relieve`); they differ only in where *behavior* lives. (Kind C —
`SustainedShock` — was added by the [electricity](./electricity.md) build.)

- **Kind A — afflictions** (`Condition` in `platform/idea/Condition.ts`):
  identity-bearing authored content as `Condition extends Idea` templates, resolved
  by `findByTemplatePath` like Materials/Species. The instance record is
  `{ kind: 'affliction', templatePath, stage, elapsed }`; behavior lives
  on the Idea. The vitals build shipped **zero content**; the
  [metabolism](./metabolism.md) build adds the first authored conditions
  (`starvation`/`dehydration`/`collapse` + the toxin conditions) and a
  `toxinBehavior?: ToxinBehavior` field on `Condition` (null for non-toxin
  conditions) carrying a toxin's per-body rate params — read by
  metabolism's reconcile, the only consumer.

  > **⚠ HISTORICAL — closed by `ConditionCatalogue`.** Left standing
  > because the failure mode is the interesting part, not because it is
  > still true: condition seeds were inserted as template ROWS and nothing
  > cloned them into Ideas at boot, so
  > `findByTemplatePath` answered `null` for **every** condition in a
  > running world — `starvation` as much as `recovering`. Every consumer
  > written so far quietly tolerates it (`Metabolic.resolveToxinBehavior`
  > `?.`-chains to null; `MagicLogic` null-checks its seed; `assess`
  > falls back to the path leaf), which is why nothing has ever failed
  > loudly and why it went unnoticed until the mortality build's
  > end-to-end pass read one back through the client. The consequence is
  > that **authored `Condition` behavior is inert**: signs, names,
  > progression and `toxinBehavior` are all read off an object that isn't
  > there. ✅ **`ConditionCatalogue` closed it** — a self-warming
  > `postRegister` stands every authored row up as a live singleton, so
  > the sync resolve-on-read seams hit from the first frame. ⭐ The
  > durable lesson is the one the banner keeps: the reads all
  > `?.`-chained past a null, **so CI was green over a dead subsystem**.
- **Kind B — trauma** (the `Trauma` value in `platform/idea/Condition.ts`):
  a parameterized value `{ kind: 'trauma', type, site, severity, bleeding?, dressed? }`
  with a closed `TraumaType` union (`laceration | fracture | contusion |
  avulsion | burn`) and the `TRAUMA_BEHAVIOR` strategy table
  (`onset`/`tick`/`resolve`/`describe`). The table ships its **no-op
  exemplar** for every type — the shape, not live behavior.
- **Kind C — sustained shock** (the `SustainedShock` value, added by
  [electricity](./electricity.md)): `{ kind: 'shock', current, source?,
  sites, tetany?, tickedAt? }` — the reconcile-on-read state of a *persisting
  closed circuit*. It integrates current × time on the `reconcileConditions`
  read path (a contact burn), re-verifies the circuit each read (tetany holds
  it closed), and at the fibrillation band drives `heartRate` to the
  electrocution death seam. The first condition kind with live drive on the
  read path beyond trauma's bleed.

All records are plain-serializable → the collection persists with no
marshaller.

### ⭐⭐ The infection arm (the food-safety build)

An affliction may carry a live **population** rather than a stage
somebody set: `pathogenLoad` (`[0, 1]`) plus `symptomsAt`, present iff
the row under `/platform/idea/Condition/pathogen/` declares
`pathogenBehavior.reach: 'infect'`. `reconcileConditions` grows it at a
NET rate — the organism's authored in-host growth against the body's own
clearance — closed-form logistic, with full presence-freeze parity
(linkdead re-stamp, far-past guard). See
[spoilage.md](./spoilage.md).

The distinction from `stage` is the whole of it: `stage` is the banded
severity a medic reads, and the load is the thing that is actually
happening. The band derives from the load, so an infection getting worse
and a treatment starting to work are the same number moving.

- ⭐ **Nothing shows until `symptomsAt`.** Illness arrives hours after the
  meal, not at the table — the information is in what you *did*.
- ⭐ **D12, resistance is thin:** clearance scales with one read of
  `getConditionBand()`. No immune memory, no exposure history, no
  per-pathogen resistance; those are the disease build's.
- ⭐ **Nothing new kills anyone.** A severe infection drains `hydration`,
  which is what dysentery does, and dehydration already ends in the
  shipped rescuable dying window.
- `treat` gained an illness arm that spends no dressing — you do not
  bandage dysentery — and knocks the population back by the medic's band.

### `ProgressionSpec`, finally read

`ProgressionSpec` (`{ intervalMs }`) shipped with the comment *"no live
scheduler is built here"*, was authored by three rows
(`starvation` / `dehydration` / `recovering`) and was **read by nothing**
— so a body three days into starvation staged identically to one that had
missed lunch. `reconcileConditions` now advances `elapsed` and derives
`stage` on the authored cadence.

⚠ **Only for what nothing else drives.** A toxin's stage is a live band
read off its burden (`reconcileToxinConditions`), so a row carrying a
`toxinBehavior` is skipped here on purpose — a dwell counter fighting the
burden read would make the answer depend on which arm ran last.

## Death & consciousness seams

> **Update (metabolism build).** These shipped as seams with no driver;
> [metabolism](./metabolism.md) is now the **first driver** that uses
> them. Its reconcile cascade spawns/clears conditions off floored
> biological reserves, stamps `setCauseOfDeath` + flips
> `setLifecycleState('dead')` when `starvation`/`dehydration` progress
> past a lethal accrual, and reuses `getConsciousness()` as the surface
> the `requiresConscious` validator + the acute `collapse` condition gate
> on. Metabolism drives *only* its own cascade — Vitals still owns no
> general driver. The seam descriptions below remain accurate.

race.md ships the `lifecycleState` machine (`alive`/`dead`/`undead`) and
defers the *transition flow*; Vitals will own the general driver, but this
build shipped only the **seams** (metabolism is the first consumer):

- **Death ≠ destruction.** A corpse is the same Stuff with
  `lifecycleState: 'dead'`; never route death through `StuffApi.destruct`.
  Animacy gating (`SpeciesApi.isAnimate`, the `requiresAnimate` validator)
  already stops animate verbs on non-`alive`/`undead` state — not
  re-modeled here. See [lifecycle.md](./lifecycle.md).
- **Cause-of-death field** — stamped at transition by the future driver;
  this build provides the field.
- **Living-stop + postmortem-start** — `getPostmortemProgressions()`
  exists and returns `[]` (zero postmortem conditions ship); the seam is
  there for a future forensics wave.
- **Derived readouts** — `getConditionBand()` reflects the substrate:
  a corpse reads `dead`, and a floored vital reads `critical`/`dead`
  **with no lifecycle transition** (the reading is a view; a *driver*
  owns transitions). `coreTemperature` is now driven by the Thermal
  build's `ThermalRegulationMixin` (a SYNC `getVitalSign` override) —
  hypothermia/hyperthermia/torpor cascade to the death seam; see
  [thermal.md](./thermal.md). `getConsciousness()` →
  `conscious`/`unconscious`/`dead` from blood volume + SpO₂ + head trauma;
  unconsciousness gates animate verbs like death but is recoverable.
- **`spo2` is now driven** — [respiration](./respiration.md) is the first
  driver to move a vital sign to the death seam: its crisis drain lowers
  `spo2` (the consciousness-blackout below `survivableMin` falls out for
  free, this seam reused untouched), spawns the `asphyxiation` condition,
  and fires the cause-of-death/lifecycle transition on sustained anoxia.
  The other signs remain substrate-only, awaiting their drivers.
- **`bloodVolume` is now driven** — [harm](./harm.md) is the injury driver
  that moves `bloodVolume` to the death seam: a bleeding laceration
  drains it **reconcile-on-read** (integrated on the `VitalsMixin` read
  path — the metabolism/thermal precedent, no recurring tick), the
  `conscious → unconscious` waypoint
  falls out of `getConsciousness()` for free (this seam reused untouched),
  and the floor stamps `setCauseOfDeath('exsanguination')` +
  `setLifecycleState('dead')`. Harm is the `inflict` producer + the five
  live `TRAUMA_BEHAVIOR` behaviors + the medic vertical (assess/treat).
- **`heartRate` is now driven** — [electricity](./electricity.md) claims the
  previously-undriven `heartRate` death seam: a fibrillating current (a
  `SustainedShock` at/above the fibrillation band) drives `heartRate` toward
  arrest on the same reconcile-on-read path; when it floors below the
  survivable band the seam stamps `setCauseOfDeath('electrocution')` +
  `setLifecycleState('dead')` (a `getVitalSign('heartRate')` read arms it,
  mirroring the `bloodVolume` bleed read).

## Reserves

Endurance / satiation / hydration are biological instances of the
generalized **`Reserve`** substrate (`lib/reserve.ts`) — see
[reserve.md](./reserve.md). A floored biological reserve feeds the
derived `getConditionBand` (a derived reading, like consciousness reads
vitals). The reserve *producers* (consumption, exertion wiring) are
deferred.

## What's deferred (the applications)

> **Update (harm build).** The trauma applications shipped: the
> [harm driver](./harm.md) is the `inflict` producer, the five live
> `TRAUMA_BEHAVIOR` behaviors (laceration bleed flagship + clot gate,
> contusion, fracture-impairs-slot, burn, avulsion), the recurring
> wound-tick (presence-frozen, re-armed on hydrate) → death by
> `exsanguination`, the limp + coverage couplings, and the medic vertical
> (`assess` / `treat` / `undress` + `DressingMixin`/`Bandage`). No longer
> deferred. The remainder below still is.

Remaining deferred: reserve drain/replenish producers, disease/poison/
affliction content, measure-on-patient instruments, the physical-attribute
readings, the anatomy graph + part-promotion (avulsion sever — see
harm.md), postmortem fidelity, and contagion.

**Mechanism of injury — LANDED (materials-response).** The parked "with
combat" seam is now built: `afflict()` is still the door an insult comes
through, and `ConditionApi.inflict` builds the `Trauma` — but severity and
type now derive from the **materials-response function**
(`f(mechanism, material, construction) × grade × condition`), resolving a
`Channel` insult outside-in through the covering stack into the per-part
tissue Materials. `Material` gained real `hardness`/`toughness` `Quantity`s
(NOT the old `resistance.<type>` 0–1 scalars — that convention is dead);
`TraumaType` grew `puncture`. **"Damage type" is explicitly not the model** —
the interactions are (channels-not-nouns). See
[materials-response.md](./materials-response.md). Still deferred: the combat
loop / playstyle, the tissue-vulnerability term, other channels
(crush/heat/cold), and the magic-side mirror
(capability-magic-slate.md).

## Cross-references

- [harm.md](./harm.md) — the injury driver over this substrate: `inflict`,
  the live trauma behaviors, the wound-tick, death by exsanguination, the
  limp + coverage, the medic vertical
- [reserve.md](./reserve.md) — the `Reserve` substrate
- [race.md](./race.md) — `OrganismMixin`, `Species`, `BodyPlan`,
  `OrganismMixin.isAlive` + `SpeciesApi.isAnimate`
- [quantities.md](./quantities.md) — `Quantity`, the new vital units
- [activity.md](./activity.md) — `ScheduleApi.recurring` cadence
- [lifecycle.md](./lifecycle.md) — death ≠ destruction
- [docs/architecture.md](../architecture.md) — the `Creature` hierarchy
- docs/slates/builds/vitals-slate.md —
  the seeding slate
- capability-magic-slate.md
  — physical attributes + mana ride this substrate

## The dying arc (shipped 2026-07-31)

The transition driver this doc deferred now exists — see
[mortality.md](./mortality.md). What changed here:

- **A fatal threshold opens a window, it does not kill.** `DyingRecord`
  (condition Kind E) + the `dying` `ConditionBand` between `critical` and
  `dead`. A floored vital reads `dying`, not `dead`: a rescued body must
  not still read as a corpse.
- **The dying arm of `reconcileConditions` opts OUT of the linkdead freeze
  and the far-past guard.** Every other arm opts in. Inheriting either
  would make disconnecting a cure for death; the divergence is commented at
  the site and pinned by a test that also asserts a bleeding body in the
  same fixture still freezes.
- **`getConsciousness()` reads a dying body as `unconscious`.** Without it,
  six of the nine drivers left a dying body walking and talking — this
  readout only knows blood volume, SpO₂ and head trauma.
- **The postmortem-progression seam is filled** by `PostmortemMixin`
  (decay stages, forensic readability, the `canEvict` terminus).
- **The material fork slices** (`MATERIAL_FORK_SLICES`) + the gated
  `adoptMaterialState`. They are fork-only, and the *absence* of a
  `mergeSlice_` is what makes a corpse un-reanimatable.
