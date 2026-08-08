# Vitals subsystem

The basic-biology layer: a living body's state modeled honestly, so it
can be healthy, hurt, sick, exhausted, unconscious, or dead. This build
ships the **substrate** — the models, type systems, resolution chains,
and seams — and defers the *behaviors* (live progression, the death
driver, condition content, instruments, treatment verbs). Bodies that
*can* be sick/hurt/strong/depleted/dead; nothing that *makes* them so
yet.

Source: `lib/vitals/Vitals.ts` (the `VitalsMixin`),
`lib/vitals/Condition.ts` (the condition type system),
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
  getInjuredParts(): ResolvedBodyPart[];
  isSlotDisabledByAnatomy(slot: string): boolean;

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
  `slots`. Each `BodyPart` carries `{ key, parent, tissues, governsVital?,
  severable?, innervatedBy?, suppliedBy? }`. v1 roster:
  head / torso / limbs (+hands/feet) + heart / lungs.
- **Tissue composition** — each part carries named tissues with masses
  (`{ tissuePath, mass }`), not a single material. Tissues are authored
  Materials under `/obj/material/tissue/` (`flesh`, `muscle`, `bone`).
  The mass-per-tissue is the substrate a future strength reading
  aggregates.
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
  seam). `BodyPlan` validates both references (referential integrity) and
  exposes the reverse query `getSlotsAt` / `getSlotsCovering`. Parts stay
  pure anatomy — no slot knowledge. The organ→vital coupling stays on the
  part (`governsVital`, heart → `heartRate`). Non-anatomical affordances
  (a saddle surface, a cranial implant bay) carry neither edge — slots
  remain a distinct axis, not something anatomy owns.
- **Stable `body.*` keys** are the identity anchor everything downstream
  points at — trauma `site`, the couplings, and the deferred graph /
  part-promotion. Locked now.

**Deferred-with-seam:** the innervation/vascular graph (`innervatedBy` /
`suppliedBy` declared, no reader); part-as-Stuff promotion (severed
limbs, transplants); MQL anatomy queries.

## Conditions — the three-kind type system

A condition is a discrete affliction on the body. All kinds present
behind one `ActiveCondition` collection (`getConditions` / `afflict` /
`relieve`); they differ only in where *behavior* lives. (Kind C —
`SustainedShock` — was added by the [electricity](./electricity.md) build.)

- **Kind A — afflictions** (`Condition` in `lib/vitals/Condition.ts`):
  identity-bearing authored content as `Condition extends Idea` templates, resolved
  by `findByTemplatePath` like Materials/Species. The instance record is
  `{ kind: 'affliction', templatePath, stage, elapsed }`; behavior lives
  on the Idea. The vitals build shipped **zero content**; the
  [metabolism](./metabolism.md) build adds the first authored conditions
  (`starvation`/`dehydration`/`collapse` + the toxin conditions) and a
  `toxinBehavior?: ToxinBehavior` field on `Condition` (null for non-toxin
  conditions) carrying a toxin's per-body rate params — read by
  metabolism's reconcile, the only consumer.

  > **The Ideas are not live yet.** Condition seeds are inserted as
  > template ROWS and nothing clones them into Ideas at boot, so
  > `findByTemplatePath` answers `null` for **every** condition in a
  > running world — `starvation` as much as `recovering`. Every consumer
  > written so far quietly tolerates it (`Metabolic.resolveToxinBehavior`
  > `?.`-chains to null; `MagicLogic` null-checks its seed; `assess`
  > falls back to the path leaf), which is why nothing has ever failed
  > loudly and why it went unnoticed until the mortality build's
  > end-to-end pass read one back through the client. The consequence is
  > that **authored `Condition` behavior is inert**: signs, names,
  > progression and `toxinBehavior` are all read off an object that isn't
  > there. Instantiating the catalogue at boot is its own small build —
  > and until it lands, treat "the Idea resolves" as an assumption to
  > verify, not a given.
- **Kind B — trauma** (the `Trauma` value in `lib/vitals/Condition.ts`):
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
marshaller. Progression shapes (`ProgressionSpec`) target
**`ScheduleApi.recurring`**, NOT the engagement-bound `ScheduledEmission`
(a condition occupies no engagement slot — see [activity.md](./activity.md)).
**Nothing ticks** in this build — `afflict`/`relieve` are pure add/remove.

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
