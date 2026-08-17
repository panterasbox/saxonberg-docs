# Thermal

The third vitals **driver**: a generic `Thermal` heat-exchange
capability (lazy Newton's-cooling-on-read) that drives object
temperature everywhere — algor mortis on a corpse, a thermos that holds
coffee hot, a campfire that burns down to embers — and, layered over it,
Option-C thermoregulation that finally **drives** `coreTemperature` on
the living body.

Source: `lib/thermal/` (`Thermal.ts`, `ThermalRegulation.ts`); content
in `obj/` (`Flask.ts`, `Campfire.ts`, `Receptacle.ts`). Mirrors
`MetabolicMixin` beat-for-beat — read [metabolism.md](./metabolism.md)
alongside this.

## The two layers

| Layer | Mixin | Drives | On |
|---|---|---|---|
| Generic capability | `ThermalMixin` | a bulk `getTemperature()` | any Tangible+Containable Stuff (thermos, corpse, campfire, body) |
| Living regulation | `ThermalRegulationMixin` | `coreTemperature` (Vitals) | every `Creature` (composes over `ThermalMixin`) |

A bare `Thermal` object drifts toward ambient. The regulation layer
defends a setpoint by spending metabolism's reserves; when it fails (out
of fuel, an ectotherm, a corpse) the body falls back to the passive
`ThermalMixin` drift.

## `ThermalMixin` — the generic capability

Lazy reconcile-on-read mirroring metabolism: a stamped temperature
(`stampedTemperatureK`) + a game-time stamp (`thermalClockStamp`), a
`WorldClockApi.getNow()` now-source that returns `null` (idle) when no
clock is bootstrapped, a first-touch seed, a linkdead freeze, a far-past
absence guard (`MAX_REASONABLE_GAP_SEC`), and the `_thermalReconciling`
reentry guard.

- **`getTemperature(): Quantity<'K'>` is SYNC.** It reconciles against
  the **cached** ambient (`lastAmbientK`) with the closed-form Newton's
  step `T(now) = ambient + (T0 − ambient)·e^(−Δt/τ)` — no biome call on
  the read path. Exact for a piecewise-constant ambient, so no
  sub-stepping.
- **`restamp(): Promise<void>` is async** — the one `await` in the model.
  It freezes the current temperature under the *old* `lastAmbientK`,
  resolves the new scope's ambient (`BiomeApi.resolveTemperatureFor`),
  and re-anchors. Every ambient discontinuity fires it.
- `getSurfaceTemperature()` blends core↔ambient by the medium exposure —
  a sealed, high-R vessel reads ~ambient though its contents scald (the
  surface-vs-contents sensory gate); a bare object reads ~its own core.
- `getContentsTemperature()` is the held-fluid temperature (= the
  object's own temperature for a vessel whose Thermal IS its contents).

### Cached-ambient (the load-bearing decision)

`getTemperature()` reads `lastAmbientK`, never the async biome chain.
This keeps the read sync — critical because the body's `getVitalSign`
override (below) would otherwise force the *entire* vitals read surface
async. The cache is refreshed only at **re-stamp events**; correctness
rests on every ambient change firing one:

1. **Containment move** — `ThermalMixin.onMoved` (the witness
   `ContainmentApi.move` fires on every mover) calls `restamp()`. This is
   the one event thermal listens to (the genuine divergence from
   metabolism's pure-lazy model — a sync read can't lazily re-resolve, so
   a move that didn't re-stamp would drift toward a stale ambient
   forever).
2. **In-place ambient shift** — `AtmosphericMixin.setTemperature` fans
   out `restamp()` over the scope's Thermal contents.
3. **Seal toggle** / **bulk transfer** — see the thermos.

### τ = R·C

- `C = mass × specificHeat` — `Tangible.getMass()` × the host's
  `Material.getSpecificHeat()` (falling back to a dial). A `Bulkable`
  vessel derives `C` from its **contents** (more fluid → larger C →
  slower cooling).
- `R` = series resistance: the surrounding medium's conductivity
  (`BiomeApi.conductivityOf`, dominant) + the wall material's (minute). A
  sealed `Sealable` host switches its barrier to `vacuum` (τ in hours);
  open collapses to the air term (τ in minutes).

## The thermos (`Flask`)

`ThermalMixin(SealableMixin(BulkableMixin(Thing)))`. The vessel's Thermal
temperature IS its contents. Sealing is the barrier switch (vacuum vs
air); seal toggles re-stamp. The bulk couplings ride a gated thermal tier
on `BulkableApi.transfer`:

- **Refill** → the destination adopts the incoming temperature.
- **Partial pour** → the source's C shrinks → the remainder cools faster
  (re-anchored at the reduced capacity).
- **Mix** → same-material calorimetric blend (specific heats cancel →
  volume-weighted average), via `ThermalMixin.setContentsTemperature`.

A plain `Receptacle` (`ThermalMixin(BulkableMixin(Thing))`) is the
non-sealable case (a mug): no barrier → it cools in minutes.

## Senses (`feel`, burn)

`feel <thermal object>` reports the object's **surface** band
(`Touch.bandFor`). A sealed thermos reads ~ambient though scalding inside
— "measuring requires unsealing" is a fact of touch. The general
**scalding-band (≥ 345 K) burn hook** afflicts a `burn` trauma on contact
— wired on `feel` and a bare-handed `get`, gating on `Touch.bandFor`
(no new verb, no `ThermalApi`). The campfire is one consumer of this
hook, not its own mechanic. Contact-touch (external) stays separate from
the body's interoceptive shiver/sweat cues (sensing its own core).

## `ThermalRegulationMixin` — the living body (Option C)

Composes on `Creature` outer of `ThermalMixin`/`MetabolicMixin`, inner of
`LoadBearing`. Drives `coreTemperature` by overriding **`getVitalSign`**
(the `getReserve` analogue) — and it stays **SYNC** because the reconcile
reads a **cached effective ambient** (`effectiveAmbientK`), never a live
biome call. Every cockpit poll, condition cascade, and `getConditionBand`
read sees a reconciled core temp.

The per-slice reconcile (mirroring metabolism's skeleton) branches on the
thermoneutral dead-band:

- **Within `[setpoint ± band]`** → pin core at setpoint, zero cost.
- **Below band (cold stress)** → spend **satiation** ∝ gap to hold the
  setpoint; shiver cue. Out of fuel → drift (the "starving = cold"
  cliff).
- **Above band (heat stress)** → spend **hydration** ∝ gap to sweat,
  capped by the **wet-bulb evaporative ceiling**; sweat cue. Past the
  ceiling or out of water → drift up.

**Strategy split** (`BodyPlan.thermalStrategy`): an **endotherm** defends
the setpoint; an **ectotherm** / dead / no-spendable-reserves body floats
to the effective ambient (the robot/corpse limiting case — no special
case). `effectiveAmbient()` (a `protected` re-stamp-time resolver, **not**
an Api) sums biome ambient + occupied warming-slot `warmth` + the
wind-chill (cold) / heat-index (hot) transforms read through the
surrounding-medium conductivity (immersion: cold water chills far
faster). Worn `clo` (on `WearableMixin`, summed over the worn-slot walk)
widens the comfort band downward.

### Cascade → conditions

`reconcileThermalCascade` spawns/clears seeded `Condition` Ideas off the
driven core (the metabolism cascade pattern): **hypothermia** (endotherm)
or **torpor** (ectotherm — alive but immobile, read by
`requiresConscious`) below `survivableMin`; **hyperthermia** above
`survivableMax`. Lethal dwell → `setCauseOfDeath` + `setLifecycleState
('dead')`. Seeds: `seeds/lib/thermal/conditions/`.

### Q10 (the metabolism edit)

`MetabolicMixin.thermalMultiplier()` (previously inert) reads the driven
`coreTemperature` and scales basal drain by `Q10 ^ ((core − reference) /
10)` (dials in `METABOLIC_DEFAULTS`). An endotherm pinned at setpoint ≈ 1;
an ectotherm whose core floats cold burns far less fuel.

## Dials

All tuning constants live in `THERMAL_DEFAULTS` (`lib/thermal/Thermal.ts`)
— τ geometry, band half-width, spend-per-degree, body specific heat,
wet-bulb ceiling, wind-chill, torpor band, lethal dwell — except the Q10
coefficient + reference, which live in `METABOLIC_DEFAULTS` (its consumer
is `basalDrain`). **Rates are playtest-tuned, not plan decisions.**

## Non-goals (deliberate)

Object-to-object conduction (a hot pot doesn't warm the table),
ventilation (no inter-room air mixing — weather-adjacent), installed
thermal gear (augment cooling), temperature-blending glob merge, heated
vehicle cabins, phase change / latent heat, per-region frostbite, sauna
rooms (the heat-index/wet-bulb *model* is in; rooms are not), campfire
fidelity tiers (smoke/cooking/spread), behavioral AI, fever/magic content
(the movable `setpoint` is the structure only). Each rides an existing
seam when wanted.

The Wave-2 indoor convection room-bump and a standalone
radiant-from-nearby-Thermals helper (Steps 2.1 / 2.4 indoor) are partial:
the outdoor warming-slot `warmth` path is wired end to end; the indoor
room-ambient bump is a follow-on.

## Phase change (the fire build)

`ThermalApi.depositHeat` + `ThermalApi.reconcilePhase` (the gated
`api/thermal.ts` + `obj/api/ThermalLogic.ts` pair — the heat-delivery /
phase-change surface `ThermalMixin` deliberately keeps off its own class).
**`depositHeat(joules)`** is the heat-DELIVERY primitive the sync model lacked
(the reconcile only *cools toward ambient*): `ΔT = Q / C`, thermal inertia
gating the rise — the same joules barely warm a heavy log but shove a shaving
hot, which is what makes ignition a derivable energy balance. **`MeltableMixin`**
(`lib/thermal/Meltable.ts`) + **`reconcilePhase`** is the bidirectional
transition engine, driven by *any* heat source (hearth / sun / fire, not
fire-specific): a solid past its material's `meltingPoint` holds a **latent-heat
plateau** (clamp temperature to the melting point, absorb the overshoot into the
accumulator until `mass × latentHeatOfFusion`) then **melts** — destructing and
flowing its mass to a molten `Bulkable` pool in the scope's `Floor`; a
liquid-holding vessel **boils** to gas above its boiling point and **solidifies**
to a cast `Thing` below its melting point. Bidirectional — **ice → water → steam
falls out of the shipped water material**. The **furnace family** (`FurnaceMixin`,
generalizing the `Campfire` pin — see [fire.md](./fire.md)) heats the Meltables
in its scope toward its held temperature; `ThermalApi.reachableHeatFor` reads the
hottest reachable furnace — the crafting-control read `CraftingLogic`'s heat
gate consumes (`recipe.requiresHeatK`; see [crafting.md](./crafting.md)). See
[fire.md](./fire.md) for the combustion driver + the full high-heat physics.
