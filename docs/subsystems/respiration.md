# Respiration

Respiration is the body's **air-exchange substrate** and the acute crisis
when that exchange stops (**asphyxiation**: drowning, vacuum). It is the
vitals *driver* that finally **drives [`spo2`](./vitals.md)** — a live,
banded vital sign (`Quantity<'%'>`, baseline 98, survivableMin 70) that
already gated *consciousness* but had **no driver**, exactly as
`coreTemperature` waits on thermal.

Unlike the lazy reconcile-on-read drivers ([metabolism](./metabolism.md),
thermal), respiration is an **event-triggered scheduled-engagement crisis
driver**: benign ambient runs *nothing* (no scheduler, no `spo2`
movement, zero reconcile — the idle case is provably zero-cost); a
hostile-medium event engages **one** bounded, cancellable drain that
self-terminates the instant air exchange resumes. **Respiration is the
first concrete engagement producer in the codebase** — the
[activity](./activity.md) framework shipped inert; respiration is its
first real `SustainedEngagement` + `ScheduledEmission`.

Everything lives on one mixin — **`RespirationMixin`**
(`lib/respiration/Respiration.ts`) — plus its colocated engagement
classes (`lib/respiration/RespirationDrain.ts`). There is **no
`RespirationApi`**: every caller is the host body itself, exactly like the
merged `MetabolicMixin`. The dials are a module const-object
(`RESPIRATION_DEFAULTS`, the `METABOLIC_DEFAULTS` precedent); **rates are
playtest-tuned, not engine invariants**.

## Compose, don't absorb

Respiration owns only **no-oxygen → `spo2`**. Cold → thermal; pressure →
the deferred shared layer; toxin → metabolism's toxin-burden; opacity →
perception. It does **not** read `MetabolicMixin.spo2Throttle()` — that is
metabolism's *read* of the now-driven sign (the aerobic coupling), an
explicit non-goal. **Plumbing, not anatomy**: it reads channel-state
(exchanging or not), never orifices; there is no modeled lung/throat
part. The `respires` opt-out on the body plan opts a body out entirely.

## Composition

`RespirationMixin` composes on a `Creature`-shaped host **immediately
outer of `MetabolicMixin`** (it drives `Vitals`/`spo2`, reading the same
vital surface metabolism does). The host cast is `Stuff & Vitals &
Organism` (the `MetabolicHost` idiom). The crisis drain needs
`EngagedMixin` and the move-reassess needs `MobileMixin` — both
**Character-tier** — so those surfaces are **runtime-narrowed**
(`MixinApi.isEngaged` / `isMobile` / `isHasInteractive` / `isSensor`) at
their use sites, not folded into the host type (the metabolism guard
precedent).

A bare non-Character `Creature` (a frog, a corpse) carries the breathable
config + engine but, lacking an `Engaged` slot, holds **no scheduled
drain** and never dies of anoxia — a **documented degenerate** (the proof
drownable is a `Character`). If a content case ever needs drowning frogs,
give the frog `EngagedMixin` — composition is orthogonal, not an engine
edit.

## Determining the medium — the breathable read

Whether a body exchanges gas turns on **`medium ∈
getBreathableMedia()`** — the species default from
`BodyPlan.getBreathableMedia()` (default `['air']`; a water-breather
*inverts* by authoring `['water']`). The current medium is the
engaged-mode `LocomotionMode.medium` while swimming (you are *in* the
water you swim through), else the atmosphere resolved outward from the
body's containment chain via `BiomeApi.resolveAtmosphereFor`.

The biome **`breathable` column** (`BiomeApi.breathableOf(tag)`, the
sibling of `densityOf`; throws on unknown tags) is the engine's
**known-medium gate**: a medium not in the body's set is hostile **only
if it is a known atmosphere** — an unmodeled medium raises no crisis. For
a default air-breather, `medium ∈ ['air']` and `breathableOf(medium)`
agree; the species set is what makes the water-breather inversion work.

The **derived effective set** (species default ∪ media granted by active
conferred mixins) is the seam a future **gills** breather adds — a
`getEffectiveBreathableMedia()` walking `MixinApi.getActiveMixins` for a
grant marker, swapping the engine's one read site. **Gills are cut from
this build** (no consumer); the species inversion ships, so a real
water-breathing creature is demonstrated without gear.

## The crisis engine — one engagement, lifecycle-parameterized

The central entry is the async **`RespirationMixin.reassess()`**: resolve
the current exchange state, then engage or cancel. It is idempotent
(re-entrant reassess is a no-op when already in the right state), guards
the dead (a corpse never drowns again) and the non-respiring (constructs
never engage), and cancels the opposite engagement before starting one
(the drain and recovery share the `'body'` slot).

- **engage** — a reassess that resolves "not exchanging" starts a
  **`RespirationDrain`** (`SustainedEngagement`, `type:
  'respiration-drain'`, `slots: {'body'}`). It carries a **cause**
  (`'medium'` | `'supply'` | `'channel'`) — the engine is built
  medium-agnostic / three-trigger-ready, but only `'medium'` (W1) and
  `'supply'` (W2 tank-exhausted) are produced; `'channel'`
  (strangulation) is accepted and **never fired** (no struggle substrate
  exists). The `'voice'` slot is reserved for that deferred channel.
- **tick-drain** — a recurring `ScheduledEmission` (every
  `EMISSION_INTERVAL_MS` game-time) delegates to
  `respirationDrainTick`: re-stamp game-time (the linkdead / far-past
  freeze, metabolism's guard verbatim — re-stamp, integrate nothing),
  burn any breath-hold grace, then `setVitalSign('spo2', max(0, current −
  drainPerSec·Δt))`.
- **cross `survivableMin`** — `getConsciousness()` flips to
  `'unconscious'` **automatically** (the free blackout — respiration only
  lowers `spo2`; the consciousness seam already reads it). The emission
  emits the blackout cue once (debounced) and force-ends any held breath.
- **fatal floor** — once `spo2` dwells at/below `ANOXIA_FLOOR_PCT` for
  `ANOXIA_LETHAL_SEC`, `reconcileAnoxiaCascade` fires the death seam
  (`applyDeath('asphyxiation')` — cause-of-death + lifecycle, once,
  exactly `MetabolicMixin.applyDeath`) and cancels the drain.
- **cancel-on-resume** — a reassess that resolves "breathable" cancels
  the drain and starts a brief **`RespirationRecovery`** (same shape,
  opposite-sign emission) that re-raises `spo2` toward baseline at
  `RECOVER_PER_SEC` and self-cancels at baseline (no overshoot). Keeping
  recovery an engagement makes the unconscious→conscious transition
  observable on the same cadence the drain used; once it self-cancels the
  benign steady state runs nothing.

### The anoxia cascade

Mirrors `MetabolicMixin.reconcileCascade`: while `spo2` dwells at/below
the floor, find-or-`afflict` the single `asphyxiation` `AfflictionRecord`
(`TemplatePaths.respirationAsphyxiation`), accrue `record.elapsed +=
elapsedSec`, and fire `applyDeath` past `ANOXIA_LETHAL_SEC`; clear the
affliction on recovery above the floor. The condition Idea is seeded YAML
(`seeds/lib/respiration/conditions/asphyxiation.yaml`, the `collapse.yaml`
shape: `resolution: { by: air }`, `observableSigns: [gasping,
blue-lipped, unconscious]`).

## The reassess triggers

The reassess is driven by three triggers, **all in-scope**:

1. **Traverse** — `RespirationMixin.onTraversed(exit)` (the duck-typed
   mover hook `Mobile.traverse` calls on every move) fires `reassess`.
   Covers walk-into-water and swim-into-water.
2. **Periodic re-resolve** — each drain/recovery emission ends with a
   fire-and-forget `reassess()`, so a resurface, a mode flip
   (surfacing/diving in place), or a tank refill ends the crisis within
   one tick **without** a traverse event (the in-place trigger).
3. **Gear (W2)** — the tank-tap path re-resolves the worn supply from the
   same emission.

**Out of scope** (no fan-out wiring exists): `Mobile.teleport` (fires
`autoSenseOnArrival`, not `onTraversed` — no content teleports a body into
a hostile medium) and an **in-place atmosphere flip** (a room turning to
water under a standing body — the airlock fast-follow owns it). Proof
content uses statically-water rooms a body *enters*.

## Voluntary breath control — `inhale` / `exhale`

Single dedicated object-free **posture-category** verbs (deliberately not
`hold breath` — `hold` is a contended item verb). `inhale` sets/extends
the held-breath grace (`_heldBreathUntil`, a game-time deadline); `exhale`
clears it. A drain arms an **automatic** `UNPREPARED_GRACE_SEC` grace on
engage (you instinctively hold your breath); a voluntary `inhale` sets the
longer `INHALE_HOLD_SEC`, and the drain honors `max(existing, auto)` — so
a prepared dive crosses a `spo2`-gap an unprepared body cannot. A blackout
force-ends the hold; both verbs validate `requiresAnimate` +
`requiresConscious` (a blacked-out body can't hold its breath). The verbs
ride the contextual-affordance surface via
`RespirationMixin.commandContributions` (the `EngagedMixin` precedent).

## Wave 2 — the carried supply (the scuba loop)

A worn **`AirTank`** (`obj/AirTank.ts` —
`WearableMixin(SlottableMixin(BulkableMixin(Thing)))`) carries an
`interior` [bulk](./bulk.md) of `air`. When the ambient medium is
unbreathable, `assessExchange` produces `cause: 'supply'` if a worn tank
is present (with or without air); the drain tick then **taps the tank**
(`debit` at `TANK_TAP_PER_SEC`, holding `spo2` steady) while it has air,
and **falls through to the `spo2` drop** once it runs dry — the
supply-exhausted crisis. Precedence: a breathable ambient is checked
first, so a tank is never burned in clean air.

- **Refill** is `BulkableApi.transfer` from an air source (the dive-shop
  `air-compressor` seed) into the tank's interior slot — no new
  machinery.
- **The gauge** is `AirTank.getAirGauge()` (interior fill fraction,
  `[0,1]`), exposed as an MQL-projected field (`subscribableFields`,
  marked `static`). Live-push on every tap is **deferred**: the bulk
  substrate does not fire `FieldChangedEvent` on `debit` today, so the
  gauge re-reads on projection rather than streaming — a refresh-on-debit
  hook is a future bulk-substrate wiring.

**Gills are cut** (the confer-based permanent-water-access archetype — the
first real `confers()` breathing consumer); W2 ships only the tank (the
carried-supply/budget archetype, the actual scuba loop).

## Presence freeze + persistence

The drain/recovery emissions inherit metabolism's **linkdead / far-past
freeze**: a linkdead body re-stamps and integrates nothing, and the
scheduler's game-time timers already pause on a paused clock (zero
connection work). Respiration carries almost **no persistent state** —
`respirationClockStamp` only (the drain's elapsed anchor); the
engagements are runtime-only (`EngagedMixin._engagements` is not
persisted), so a server restart drops in-flight drains and the next
reassess re-engages. `spo2` rides Avatar save/restore (Vitals). No new
persistence machinery.

## Dials — `RESPIRATION_DEFAULTS`

`EMISSION_INTERVAL_MS`, `DRAIN_PER_SEC_BY_CAUSE` (`{medium, supply,
channel}`), `UNPREPARED_GRACE_SEC`, `INHALE_HOLD_SEC`, `RECOVER_PER_SEC`,
`ANOXIA_FLOOR_PCT`, `ANOXIA_LETHAL_SEC`, `MAX_REASONABLE_GAP_SEC` (reuses
metabolism's 4h), `TANK_TAP_PER_SEC`, and `DEATH_CAUSE_BY_CAUSE`. All
playtest-tuned.

## Deferred (explicit non-goals)

Each rides an existing seam when wanted; none is blocked by v1's shape.

- **Strangulation / the channel trigger** — no struggle/grapple substrate
  exists; the engine accepts `cause: 'channel'` and never fires it. The
  `'voice'` slot exists for the later voice-theft.
- **Gills / confer-based breathing** — the effective-set derivation seam +
  an `_augmentGated` `WaterBreathing` mixin + a `Gills` augment. This
  build touches no augmentation code.
- **The contaminant channel (breathable ≠ safe)** — gated on metabolism's
  toxin-burden. Only the `contaminant` column is **laid, unread** (`api`
  / `BiomeLogic` `ATMOSPHERE_CONTAMINANT`); no engine reads it.
- **The airlock / moon-pool interlock** — the in-place atmosphere-flip
  reassess (no fan-out wiring) is the next build.
- **Pressure injury / altitude / CO₂ / radiation / combat anatomy** —
  later fidelity layers, no consumer.
- **The aerobic coupling** — metabolism's read of the now-driven `spo2`
  (`spo2Throttle`), inert until metabolism reads it.

## Files

- `lib/respiration/Respiration.ts` — `RespirationMixin`,
  `RESPIRATION_DEFAULTS`, the `Respiration` interface.
- `lib/respiration/RespirationDrain.ts` — `RespirationDrain` /
  `RespirationRecovery` engagements, the `RespirationCause` type, the
  activity-class registrations.
- `obj/AirTank.ts` — the worn air-bulk tank + the air gauge (W2).
- `seeds/lib/respiration/conditions/asphyxiation.yaml` — the anoxia
  condition Idea.
- `seeds/lib/material/bulk/air.yaml`,
  `seeds/obj/gear/{air-tank,air-compressor}.yaml`,
  `seeds/domain/eternal/duncan-hall/cistern.yaml` — proof content.
- `cmd/posture/{inhale,exhale}.yaml` +
  `obj/command/posture/{Inhale,Exhale}Controller.ts` — the verbs.

## History

The build landed on the `feature/respiration` branch (W1 crisis core +
W2 carried supply in one feature commit, followed by a cast-cleanup
refactor). The plan (`docs/plans/respiration-plan.md`) was written against
**pre-refactor** master; the build adapted it to the merged
surface-architecture refactor — most visibly, the biome media table now
lives in the `BiomeLogic` logic singleton, so the `breathable` column
landed there with a `BiomeApi` facade forwarder (`Mml`/`MessageApi` stayed
in `api/`, so the player-cue path was unchanged). The respiration slate
was fully absorbed and retired at sweep; its deferred surface (gills /
confer-based breathing, the inhaled-toxin and strangulation channels,
CO₂ / rebreather, the airlock interlock, pressure / altitude) lives in
[Deferred (explicit non-goals)](#deferred-explicit-non-goals) above.
