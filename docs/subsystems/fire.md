# Fire & combustion

The high-temperature physics subsystem: the **combustion driver** (fire as a
real fire-triangle process over shipped `Thermal`/`Wet`/`Material` numbers) +
the high-heat materials physics the crafting system will later stand on (phase
change, the furnace family, an inert heat-as-crafting-control seam). Built for
its own sake — the electricity → mundane-`conduct` precedent — and the exact
channel the magic **Fire school** will later actuate (inject heat → the same
physics). Design surface in the
fire-combustion-slate.md; the 11
surface decisions (D1–D11) the build's retired requirements settled are now
captured below.

Homes: **`lib/fire/`** (combustion — `Combustible`/`Burning`/`Furnace`, the
`FireApi`/`FireLogic` gated pair), **`lib/thermal/`** (the phase-change layer —
`Meltable`, `ThermalApi.reconcilePhase`/`reachableHeatFor`), **`lib/material/`**
(the new `Material` props + the materials-response `heat` channel).

## The five layers

### 1. Real `Material` properties (the numeric foundation)

Six real, tabulated `Quantity` properties on `Material` (the
`electricalConductivity`/`hardness` precedent — `0`-until-authored, so an
unauthored material never ignites and never melts): `autoignitionTemperature`
(K), `heatOfCombustion` (MJ/kg), `meltingPoint` (K), `latentHeatOfFusion`
(J/kg), `boilingPoint` (K), `latentHeatOfVaporization` (J/kg). Two new units —
`MJ/kg`, `J/kg`. Authored with real figures on the base-library roster (paper
506 K, wood 570 K / 16 MJ/kg, iron mp 1811 K / bp 3134 K, water mp 273 / bp
373 / 334 kJ·kg⁻¹ / 2.26 MJ·kg⁻¹).

### 2. The materials-response `heat` channel

`heat` joins the `Channel` vocabulary as the **second non-mechanical channel**
(after `shock`) — its own `THERMAL_CHANNELS` subtype. It does **not** fold
through the hardness/toughness mechanical response; it resolves by
**insulation**: `MaterialLogic.attenuateImpl`'s heat branch reads each covering
layer's `thermalConductivity` (inverted: `refCond / (refCond + conductivity)`)
× the construction's layer depth × grade/condition, so leather/padding turns a
burn and plate conducts it — **the armor inversion, emergent from conductivity**
(a metal gauntlet is WORSE against heat than none), no `isThermal` special case.
`resolveTraumaImpl`'s heat branch maps surviving heat straight to a `burn`. The
`heat` channel **retired the old magnitude-only `'thermal'` passthrough**
(`InsultKind = Channel | 'tearing'`); the shipped touch-burn producers
(`FeelController`, `GetController`) now route through
`ConditionApi.inflict({mechanism:'heat', energy, site})` (via
`Touch.contactBurnEnergy`), so a glove on the hand insulates before the residual
burns tissue. **No parallel fire-damage path** — heat-to-body is one channel.
Dials: `response.heat.*`.

### 3. The combustion driver (`lib/fire/`)

- **`CombustibleMixin`** — the capability on matter: a `'fuel'` `Reserve`
  (the Campfire precedent) + a **`Burning`** value-object active state
  (`{ignitedAtGameSec, complete}`). It reads its **material** (not authored
  coefficients) for the ignition point + fuel value. Composed *outside*
  `Thermal`+`Reserved`, it overrides `getTemperature()` to pin the flame
  temperature while aflame (the generalized Campfire pin) and delegates to
  `super` (the cooling embers) once out. Fuel drain is **reconcile-on-read**
  over game-time — a fire left alone burns down to char even unobserved. The
  gated `FireLogic` is the **single external writer** of the Burning state
  (`ApiOnly` mutators); the host's own burnout is internal.
- **Ignition is a derivable energy balance (D3).** An object ignites when its
  temperature crosses `autoignitionTemperature` — **raised by the latent heat
  of the water it holds**: `ΔT_wet = saturation × capacity% × L_vap /
  specificHeat` (the fuel mass cancels between the water-boil energy and the
  thermal capacity, so a soaked log resists regardless of size — the
  wet-firewood, now derived). Reaching the threshold is the energy balance
  (`depositHeat` gates the rise by thermal inertia — a match can't out-heat a
  beam); the ignition itself is the threshold cross. `FireApi.tryAutoignite`
  is the heat-threshold path (spread + tests); `FireApi.ignite` is the
  deliberate `ignite`-verb path (a hand-flame; the wetness penalty must be
  below the manual-drying headroom, else "too wet to catch").
- **Consumption end-state (D4).** Fuel drains → the material transforms to its
  `charMaterialPath` (ash/char, embers cooling via passive `Thermal`); a
  **structural** object (a door, a bridge) burns through and destructs
  (`hasBurnedThrough` seam — content decides the after-state).
- **The three extinguishers.** Water/`douse` (extinguish + wet against
  re-ignition), smother (no O₂ — layer 4), fuel-starvation (burns to embers).
- Verbs `ignite`/`douse` (`device` category), gated `FireApi`/`FireLogic`.

### 4. The presence-gated fire tick + real chemistry

- **Spread (D1/D6).** `FireApi.onFireTick` — a game-time
  `WorldClockRegistry.every` fan-out over **occupied** scopes (the
  weather-boundary / storm-strike precedent; an unwatched fire **freezes**,
  zero work in empty rooms, no offline-arson grief). Each burning object drains
  its fuel + **radiates heat** (`ThermalApi.depositHeat`) into co-located
  combustibles and, **through OPEN boundaries only** (a closed/locked door is a
  firebreak — the `Sealable` read), into the adjacent scope's; a neighbour
  catches iff the delivered heat crossed its wetness-adjusted ignition point,
  so a wet neighbour resists — emergent from the energy balance. A lone
  `Burning` reconciles-on-read for `analyze`; the tick is the authoritative
  spread driver.
- **The oxygen leg + complete/incomplete (D5).** An enclosed scope authors an
  `'air'` `Reserve`; a fire consumes it, a ventilated boundary (sky-exposed or
  an open exit) replenishes. **Complete** (enough air → hot, clean) vs
  **incomplete** (starved → cooler flame + soot **smoke** + **carbon
  monoxide**). Smoke lands as a `smoke` atmosphere tag (breathable:`false`,
  contaminant:`carbonMonoxide`) set via `Atmospheric.setAtmosphere` — the
  scope's medium turns un-breathable (the existing respiration medium crisis
  **asphyxiates** for free) and **`RespirationMixin` folds the contaminant into
  the breather's metabolism toxin burden** (`BiomeApi.contaminantOf` → the
  `carbonMonoxide` `Condition` — the laid-unread `ATMOSPHERE_CONTAMINANT`
  seam's first consumer). Air floors → the oxygen leg fails → **self-smother**.
  An enclosed fire kills by CO, not flame. Dials: `fire.air.*`,
  `respiration.contaminantBurdenPerBreath`.

### 5. Phase change + the furnace family

- **Phase change (`lib/thermal/`, D7).** `MeltableMixin` (a solid + a latent
  accumulator) + **`ThermalApi.reconcilePhase`** — the bidirectional engine,
  driven by *any* heat source: a solid past its `meltingPoint` holds a
  **latent-heat plateau** (clamp temperature to the melting point, absorb the
  overshoot into the accumulator) then **melts**, destructing and flowing its
  mass to a molten `Bulkable` pool in the scope's `Floor`; a liquid-holding
  vessel **boils** to gas above the boiling point and **solidifies** to a cast
  below the melting point (a **clone of the `/obj/Casting` template** — a
  re-meltable content object, material/mass/prose stamped per freeze; not a raw
  construction). Bidirectional — **ice → water → steam falls out of one water
  material**.
- **The furnace family (D8).** `FurnaceMixin` generalizes the Campfire pin — a
  `Combustible`-fuelled appliance holding a `burnTemperatureK × bellows`
  temperature while lit + fuelled, releasing to embers on burnout, and
  **heating the Meltables in its scope** (`heatContents`) toward that
  temperature. **`Campfire` is refactored onto it byte-identically** (pin 800
  K, guarded by its own suite). `Forge`/`Kiln`/`Oven` compose it with different
  fuel + bellows dials — **smelting heat (iron's 1811 K) reachable only with the
  bellows**. `FireApi.ignite`/`douse` light/extinguish a furnace.
- **The Candle** — the convergence fixture: `LightSource + Combustible +
  Thermal + Reserved(wax)` over a `Thing`'s `Wet` wick. A dry wick lights (the
  wet-wick gate refuses a soaked one, keyed on the wick material's water
  absorption) → it emits light (flux gated on `isBurning`) and burns its wax;
  snuff/douse darkens it; under a sealed (air-limited) jar it self-smothers.
  *(The wax phase-change pool is deferred — the flame pins the whole body hot,
  so the wholesale `Meltable` melt is unsuitable; a gradual candle-specific drip
  is the follow-on.)*

### The crafting seam (D9) — **consumed**

**`ThermalApi.reachableHeatFor(position)`** — the maximum sustained temperature
(the hottest lit furnace) reachable from a position, the crafting
emergent-reachability principle applied to heat. Built inert by this build;
**consumed by the crafting-branches build with zero retrofit**, exactly as
designed: `CraftingLogic`'s heat gate declines any recipe whose
`requiresHeatK` exceeds it (`insufficient-heat`, diegetic — "the forge is
cold"), and the by-hand `heat` step latches it onto the build buffer. See
[crafting.md](./crafting.md).

## Constraints honored

- **Presence-freeze / no runaway** — the `fire:tick` fan-out is occupied-scope
  only, per-scope deduped; an unwatched scope does zero work; the `Burning`
  read is presence-frozen via the world-clock now-source guard.
- **No parallel damage path** — heat-to-body routes only through the `heat`
  `Channel` → `ConditionApi.inflict`; combustion-of-object routes only through
  the gated `FireLogic`. One writer each.
- **Real Quantities under a banded surface** — the six props are real
  `Quantity`s; players see bands, raw numbers on `analyze` only.
- **Go through the Api layer** — `StuffApi.create`/`destruct`,
  `ContainmentApi`, `BulkableApi`, `ThermalApi.depositHeat`/`reconcilePhase`.
- **No new module categories** — capability mixins in their subsystem folders;
  `FireApi`/`ThermalApi` gated Api pairs (the `Material`/`Weather` shape);
  declarative demonstrator content.

## Content

The **Hearthworks** (`domain/hearthworks/`, `seeds/domain/hearthworks*`) — a
self-contained fire zone (teleport-reachable, the substation precedent) with a
**woodshed** (spread + wet-resist), a **sealed cellar** (`SealedCellar` — the
CO/ventilation lesson), and a **smithy** (a bellows-fed `Forge` melting an
`Ingot` to a molten pool). `obj/Firewood` (a Combustible log), `obj/Ingot` (a
Meltable metal bar), `obj/Casting` (the re-meltable frozen-pool cast),
`obj/Forge`/`Kiln`/`Oven`, `obj/Candle`.

## Deferred

The crafting recipes (cooking / smelting / smithing / glassmaking — the
downstream consumer of this substrate); fire as a combat weapon /
burning-DoT; map-scale wildfire / arson-as-crime / a fire brigade;
vision-obscuring smoke (the fog→visibility seam); cross-room smoke drift;
flammability limits (LEL/UEL); the magic Fire school (actuates this channel);
electricity `Joule → fire`; the candle wax-pool phase-change.

## Cross-references

- [thermal.md](./thermal.md) (passive Thermal + `depositHeat` + phase change),
  [materials-response.md](./materials-response.md) (the `heat` channel),
  [harm.md](./harm.md) (`ConditionApi.inflict` / `burn`),
  [weather.md](./weather.md) (`WetMixin` / the presence-gated fan-out),
  [respiration.md](./respiration.md) (`breathableMedia` / `contaminant`),
  [bulk.md](./bulk.md) (fuel / molten liquid), [light.md](./light.md)
  (`LightSource`), [crafting.md](./crafting.md) (the deferred consumer).
