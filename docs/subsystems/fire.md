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
`Meltable`, `reconcilePhase`/`reachableHeatFor`), **`lib/material/`**
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
**insulation**: `MaterialLogic.attenuateImpl`'s thermal branch reads each
covering layer's **real `clo`** (`Wearable.getClo()` — thickness over
effective conductivity, loft and wetness included; the same number
thermoregulation reads) and blocks `1 − exp(−clo / ref)` × grade/condition,
so leather/padding turns a burn and plate conducts it — **the armor
inversion, emergent from the R-value** (a metal gauntlet is WORSE against
heat than none), no `isThermal` special case. ⚠ It used to read
`thermalConductivity` through a heuristic of its own; the injury build
made it read the garment's derived clo — see
materials-response.md § One insulation number.
`resolveTraumaImpl`'s heat branch maps surviving heat straight to a `burn`. The
`heat` channel **retired the old magnitude-only `'thermal'` passthrough**
(`InsultKind = Channel | 'tearing'`); the shipped touch-burn producers
(`FeelController`, `GetController`) now route through
`ConditionApi.inflict({mechanism:'heat', energy, site})` (via
`Touch.contactBurnEnergy`), so a glove on the hand insulates before the residual
burns tissue. **No parallel fire-damage path** — heat-to-body is one channel.
Dials: `response.heat.referenceClo` (the pulse reference) and `response.heat.referenceThicknessM` (the slab fallback for a layer with no derived clo).

### 3. The combustion driver (`lib/fire/`)

> ⭐⭐⭐ **A fire is its fuel, its air and its vessel; heat, light and
> exhaust are consequences.** The vessel sets the ceiling, the fuel
> decides whether you reach it, the air decides how clean. ⚠ And the two
> halves of this subsystem keep their fuel DIFFERENTLY, which is the one
> thing to hold in mind reading the rest: a **`Combustible`** (matter that
> burns — a log, a turf) still carries a `'fuel'` `Reserve`, because what
> is being consumed is the object itself. A **`Burner`** (a vessel that
> holds a fire — a forge, a lamp, a smoker) carries a **fuel BED**:
> kilograms, keyed by material. See *the burner's fuel is a bed* below.

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
  beam); the ignition itself is the threshold cross. `tryAutoignite()`
  is the heat-threshold path (spread + tests); `ignite()` is the
  deliberate `ignite`-verb path (a hand-flame; the wetness penalty must be
  below the manual-drying headroom, else "too wet to catch").
  ⭐ The formula is the BODY's (`TangibleMixin.getWetIgnitionPenaltyK`, and
  `getIgnitionPointK` = autoignition + it): `stoke`, `ignite` and a hot
  pour (`canReceiveAtK`) read the one number, so a soaked board survives a
  pour that would light a dry one (ceramics review R2-B1). ⚠ A firing
  chamber catching its contents (`Burner.catchContents`) reads the DRY
  point on purpose: a piece that has sat at that heat in a drawing chamber
  has driven its water off, and the thermal model has no boiling plateau
  to show it.
- **Incandescence is one scale** (`FireApi.incandescenceOf(K)`): dull red
  from Draper's point ~800 K, cherry ~950, orange ~1150, yellow ~1300,
  white ~1500. A kiln's spy-hole (`analyze temperature`), a glass gather on
  the pipe and a lit burner's glow seen in the dark (`look`, by the
  burner's own held heat) all speak it; only a light with no hot body (a
  lamp, a worked flame) is worded by its authored colour temperature
  (ceramics review R2-B3).
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
  its fuel + **radiates heat** (`depositHeat`) into co-located
  combustibles and, **through OPEN boundaries only** (a closed/locked door is a
  firebreak — the `Sealable` read), into the adjacent scope's; a neighbour
  catches iff the delivered heat crossed its wetness-adjusted ignition point,
  so a wet neighbour resists — emergent from the energy balance. A lone
  `Burning` reconciles-on-read for `analyze`; the tick is the authoritative
  spread driver.
- **The oxygen leg + complete/incomplete (D5).** ⭐⭐ **The air DERIVES
  from the scope's own openings** (the fire build, 2026-10). It used to be
  an authored `'air'` `Reserve`, and that was the single worst thing in
  this subsystem: ⛔ **four of the seven rows that authored one could not
  hold the key at all** (no `ReservedMixin` anywhere in a
  `Locale` chain), so the vintner cellar, the brewing
  floor, the cold store and the Crowsfoot floor each *meant* to displace
  their own air and silently did not — and the one room that worked did so
  because somebody remembered the line. A scope's medium now carries
  CONTENTS (litres per litre, keyed by `Material` path) beside its
  identity tag, and `airShare` is `1 − contentsSum`; ventilation is
  `airChangesPerHour()` — `Infinity` under the sky, else
  `exterior × achPerOpening + interior × achInterior + achLeak`. So a shut
  stone cellar starves a fire **because it is a shut stone room**, and
  every scope with a doorway does not. **Complete** (enough air → hot, clean) vs
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
  accumulator) + **`reconcilePhase`** — the bidirectional engine,
  driven by *any* heat source: a solid past its `meltingPoint` holds a
  **latent-heat plateau** (clamp temperature to the melting point, absorb the
  overshoot into the accumulator) then **melts**, destructing and flowing its
  mass to a molten `Bulkable` pool in the scope's `Floor`; a liquid-holding
  vessel **boils** to gas above the boiling point and **solidifies** to a cast
  below the melting point (a **clone of the `/stuff/thing/Casting` template** — a
  re-meltable content object, material/mass/prose stamped per freeze; not a raw
  construction). Bidirectional — **ice → water → steam falls out of one water
  material**.
- ⭐⭐ **It is `BurnerMixin`, and it was `FurnaceMixin` until the
  base-class narrowing (2026-09-29).** A lamp, a lantern, a candle and a
  campfire all compose it, and **none of them is a furnace** — the name
  came from the first consumer instead of the capability, so the arg
  refusal a player read was *"a candle isn't a furnace"*. It is *"{} won't
  hold a fire"* now. A burner is a thing that holds a fire and burns fuel
  to keep it; how hot, whether it encloses, and whether it warms the room
  are dials and other mixins.
- ⭐⭐ **`lib/fire/Firebox` — a BUILT-IN fire**, and the class the six
  hand-written copies of one chain turned out to be:
  `Burner(LightSource(Reserved(Thermal(Thing))))`. `Forge` IS that chain
  and nothing else; `Oven` adds `Container` + `Placing` (it encloses what
  it heats); `Hearth` adds `SpaceHeating` + `Placing` (it warms the air,
  which a forge does not); `Campfire` adds `Postured` + `Slotted` (it
  seats you); `SmeltingFurnace` is `Container(Forge)`; `CharcoalPit` is
  `Container(Firebox)`. ⚠ The composition ORDER is load-bearing and the
  suites know it: `Burner` outermost so ignite/douse see the composed
  answers through `super`, `Thermal` innermost because the fuel's heat is
  what the model integrates.
  ⚠⚠ **A LAMP does not compose it.** A lamp, a lantern and a
  `trade-distilling` `Still` are goods you carry, so they sit on
  `Good` and write the same four mixins themselves. **A shared
  capability chain is not a shared rung** — the same conflation
  `Fitting`/`Station` resolved one wave earlier. Two consumers is not
  three, so a portable twin is declined until something forces it.
- **The furnace family (D8).** `BurnerMixin` generalizes the Campfire pin — a
  `Combustible`-fuelled appliance holding a `burnTemperatureK × bellows`
  temperature while lit + fuelled, releasing to embers on burnout, and
  **heating the Meltables in its scope** (`heatContents`) toward that
  temperature. **`Campfire` is refactored onto it byte-identically** (pin 800
  K, guarded by its own suite). `Forge`/`Kiln`/`Oven` compose it with different
  fuel + bellows dials — **smelting heat (iron's 1811 K) reachable only with the
  bellows**. `ignite()`/`douse()` light/extinguish a furnace (the same face rides
  `BurnerMixin`).

  ⚠⚠ **`getHeldTemperatureK()` is the PIN, not the reading.** It is
  `burnTemperatureK × bellows` and consults neither `lit` nor fuel — the
  temperature this furnace *would* hold, not the one it is at. A
  stone-cold shaft answers 1420 K. Any caller that treats it as the
  current temperature is wrong: `smelt` did, and told a player with an
  unlit furnace to *"work the bellows"* — which then answered *"air
  without fire moves nothing."* Check `isLit()` first; the accessor
  will not do it for you. (Found by charging a furnace in a browser,
  2026-09-16; every unit fixture had lit it first.)
- ~~**The Candle**~~ — ⚠ **retired, unrowed, by the base-class narrowing
  build.** It was the convergence fixture (`LightSource + Combustible +
  Thermal + Reserved(wax)` over a `Thing`'s `Wet` wick) and it was a
  fixture in the literal sense: **no content row ever named it**, in the
  whole life of the class, and nothing but its own test imported it. Its
  per-class `isBurning()` lit-gate is the thing `BurnerMixin` took over
  (see *A fuelled appliance now casts light only while it burns*, below),
  which is what left it with nothing of its own.
  ⭐ **A candle is a `Lamp` row today** — `BurnerMixin(LightSource(
  Reserved(Thermal(Good))))`, which is a fuelled thing that
  lights, burns its reserve and gates its flux on being lit. The general
  store's torch is already one. What no class offers is the wax pool, and
  that was deferred on the Candle too: the flame pins the whole body hot,
  so the wholesale `Meltable` melt is unsuitable and a gradual drip is
  still the follow-on.

### The crafting seam (D9) — **consumed**

**`reachableHeatK(position)`** — the maximum sustained temperature
(the hottest lit furnace) reachable from a position, the crafting
emergent-reachability principle applied to heat. Built inert by this build;
**consumed by the crafting-branches build with zero retrofit**, exactly as
designed: `CraftingLogic`'s heat gate declines any recipe whose
`requiresHeatK` exceeds it (`insufficient-heat`, diegetic — "the forge is
cold"), and the by-hand `heat` step latches it onto the build buffer. See
[crafting.md](./crafting.md).


### The heat scope (the grain-chain build)

A furnace has **two** scopes and they are different mechanisms:

| scope | what it is | what it does | for what |
|---|---|---|---|
| `heatContents()` | the furnace's **room siblings** | deposits joules toward the held temperature, reconciles phase | `Meltable` workpieces only — the forge melting an ingot beside it |
| `restampHeated()` | what the furnace **holds** (`Container`) and what is **placed on** it (`Placing`) | re-stamps each body so it re-resolves its ambient | every `Thermal` body — the loaf in the oven, the pot on the fire |

The second is the **furnace couple**: the reading lives on the body
(`ThermalMixin.heatSourceK`, see thermal.md), and the furnace's job is
only to tell its heat scope that its lit state changed — from
`_setLit()` and from the burnout edge. ⚠ A furnace is deliberately not
`Atmospheric`, so neither scope warms the room.


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
  `ContainmentApi`, `BulkableApi`, `depositHeat`/`reconcilePhase`.
- **No new module categories** — capability mixins in their subsystem folders;
  the `FireApi` facade + the thermal mixin face (the `Material`/`Weather`
  shape);
  declarative demonstrator content.

## Content

The **Hearthworks** (`domain/hearthworks/`, `world-seed/content/world/terminus/hearthworks*`) — a
self-contained fire zone (teleport-reachable, the substation precedent) with a
**woodshed** (spread + wet-resist), a **sealed cellar** (`SealedCellar` — the
CO/ventilation lesson), and a **smithy** (a bellows-fed `Forge` melting an
`Ingot` to a molten pool). `obj/Firewood` (a Combustible log), `obj/Ingot` (a
Meltable metal bar), `obj/Casting` (the re-meltable frozen-pool cast),
`obj/Forge`/`Kiln`/`Oven`. ⚠ The list used to end `obj/Candle`; there was
never a candle row, and the class is retired — see the bullet above.

## Deferred

Glassmaking recipes (cooking / smelting / smithing all shipped in their own
capability packs, reading `requiresHeatK` — see [crafting.md](./crafting.md)
— glassmaking has no pack yet); fire as a combat weapon /
burning-DoT; map-scale wildfire / arson-as-crime / a fire brigade;
vision-obscuring smoke (the fog→visibility seam); cross-room smoke drift;
flammability limits (LEL/UEL); the magic Fire school (actuates this channel);
electricity `Joule → fire`; the candle wax-pool phase-change. **The
oven's own warm-up** — a furnace with thermal mass: today a lit furnace
holds its temperature instantly and what climbs is what is IN it (the
furnace couple, [thermal.md](./thermal.md)); a bread oven that takes an
hour to come to heat is a `ThermalMixin` on the furnace itself, and the
grain chain left it.

## ⭐⭐ The hearth, the lamp, and the rule that survived both

The envelope build (2026-09-24) added two composers and changed one
thing about every existing one.

### A hearth heats where you stand; a forge heats what you put in it

`thermal.md`'s rule — **a lit forge must not warm the room it stands
in** — is right, and the envelope build did not break it to get room
heating. It added a different KIND of object. `SpaceHeatingMixin`
(`lib/thermal/SpaceHeating.ts`) carries `heatOutputW` and
`spaceHeatOutputW()`, which is zero the moment the fire is out or out of
fuel, and it is composed **outermost** so it can read the furnace face.

⚠ **Never on `BurnerMixin`.** That would claim it of the forge, the
oven and the kiln, and the only way back would be a guard asking *is
this a forge* — the tell of a mixin on the wrong host. `Hearth` and
`Campfire` compose it; `Forge`, `Oven` and `Kiln` do not; the envelope
narrows a room's contents with `MixinApi.isSpaceHeating` and nothing
anywhere names a class.

`platform/thing/Hearth` is the commons object — `SpaceHeating + Furnace
+ LightSource + Reserved + Thermal + **Placing**`. ⚠ Placing and not
Container: you put a thing *into* an oven and stand a thing *on* a
hearth, which is the whole difference. `stove.yaml` and `brazier.yaml`
are ROWS on the same class.

### A lamp is a small furnace with a light on it

`platform/thing/Lamp` — `Burner + LightSource + Bulkable + Thermal`
over `Good`. The lantern and the torch moved onto it from
`PortableLight`, which is a `Switchable` and therefore **burned
forever**. The class writes almost nothing: the fuel bed, the drain
against game time, reconcile-on-read, the burnout edge and
`ignite`/`douse` are all the mixin's.

⭐ Its **one** override is `fuelSlot()`, and it is what makes a torch and
a lantern one class and two rows: *this vessel's interior is its fuel
tank*. A lantern is **filled** (`interiorBulk` + an
`interiorMaterial` of lamp oil, so `fill lantern from cask` is the act
and `stoke lantern` is refused in the bed's own words); a torch is a
bundle of pitchy wood, so its row seeds a charged `fuelBed` and leaves
`interiorBulk` off. ⚠ `ReservedMixin` is **not** in the chain any
more — a row that still authors `reserves: { fuel: … }` is authoring an
inert key, and the two that did shipped as lights nobody could ever
light (see *the lantern and the candle* below).

`burnTemperatureK` defaults to **330 K** — the case, not the flame, and
deliberately below the 345 K scalding hook so `get` and `feel` on a lit
lantern do not burn a hand. ⚠ The consequence to know: the fire tick
deposits toward 330 K into any `Meltable` beside a lit lamp, which is
honest at that temperature (wax softens, ice melts).

`PortableLight` survives, narrowed to what it is for: a light that burns
**nothing** — the glowcap jar and its fixture, which are a fungus.

### ⚠⚠ A fuelled appliance now casts light only while it burns

`LightSourceMixin` emits its authored flux unconditionally, and
lit-gating was done per class — `isOn()` on `PortableLight`, and
`isBurning()` on the since-retired `Candle`. **`Campfire`, `Forge`, `Oven` and `Kiln` have
empty class bodies and therefore no gate at all**, so a campfire that
burnt out an hour ago went on casting its full 120 lumens. Nobody caught
it because until this build nowhere was dark enough for it to matter.

Every composer puts `BurnerMixin` *outside* `LightSourceMixin`, so the
gate lives in `BurnerMixin.getEmittedFlux()` and fixes all of them at
once — and cannot come back for the next composer either.

⚠ `BurnerMixin.lit` defaults **true** (the Campfire seed it was written
for). `lint:light-sources` clause (g) makes every furnace row say which
it means. On its first run it found four: the campfire and the practicum
brazier mean it, and **both still rows shipped lit against their own
prose** (*"the firebox swept and ready"*) and their own class docstring
(*"lit with `ignite`"*).


## Cross-references

- [thermal.md](./thermal.md) (passive Thermal + `depositHeat` + phase change),
  [materials-response.md](./materials-response.md) (the `heat` channel),
  [harm.md](./harm.md) (`ConditionApi.inflict` / `burn`),
  [weather.md](./weather.md) (`WetMixin` / the presence-gated fan-out),
  [respiration.md](./respiration.md) (`breathableMedia` / `contaminant`),
  [bulk.md](./bulk.md) (fuel / molten liquid), [light.md](./light.md)
  (`LightSource`), [crafting.md](./crafting.md) (the deferred consumer).

---

## ⭐⭐ `fire` — a chamber you hold at heat; the MATERIAL decides (ceramics M1)

`platform/cmd/device/fire.yaml`, `verbs: [fire, burn]`, afforded on
**`BurnerMixin.commandContributions.peers`**. Since the ceramics build (W0)
it is **`fire [<kiln>] [--draught <word>] [for <hold>]`** and it decides
nothing about the charge:

1. `--draught wide|open|half|low|banked` (an OPTION, so it can never swallow
   the kiln — `fire kiln`, `burn kiln` and `fire in retort` bind as before)
   sets the air;
2. it refuses `no-charge` and `not-lit` (⚠ *unlit is COLD, not cool*);
3. it stamps `kineticRecord.firedBy` on every body in the chamber (and one level
   into a mould standing in it), and holds the chamber at heat — a sealed-down, watched step on the
   attention slot, a minute's effort however long the hold;
4. at the end the chamber SETTLES (`Burner.settleFiring(by)`): each body
   reads itself, the rungs its heat-work reached are applied by its own
   material, casts handed to moulds are awaited, and the Disciplines those
   rungs earned are credited to the firer; the hold is stamped as a dry heat
   (`recordHeat`) on every Soilable from the ACTUAL peak and time above,
   off each body's record.

⭐⭐⭐ **Firing is ONE rule, and it is not in this verb.** Every body that has
been hot carries a heat history (`ThermalMixin.kineticRecord`, see
[thermal.md](./thermal.md) § Kinetics), and every material heat changes says
how on its own row (`Material.kinetics`, evaluated by
`lib/material/Kinetics.ts` — the same evaluator staling runs). Limestone becomes quicklime at 56 % of its
mass; clay becomes terracotta, then earthenware, then stoneware, by three
changes with their own reference heats; wood in a STARVED chamber chars
and its tar goes to whatever takes volatiles; compounded rubber cures by its
own sulfur fraction. A lime lump in a burning barn and one in a limekiln are
the same event. There is no recipe path: the six firing rows (`burn-lime`,
`fire-pot`, the two glass batches, the two retort charges, `vulcanize`) and
`remelt-cullet`/`charcoal` are retired, and `FireController.test.ts` holds
that census by name.

⚠ The old controller's shape census (`requiresHeatK > 0`, item slots, no
tools) also matched eight rows other verbs run (`smelt`'s charges, the
smoker's cures, the chandler's wax) — a kiln could have "fired" a smoke-cure.
The one rule removes that hazard with the path.

### ⭐ A flame needs air — one gate, two arms

`FireLogic.mediumBurns(stuff)` is the one answer to *can a flame burn here*,
called from BOTH `igniteImpl` (a struck light, refused `no-air`) and
`tryAutoigniteImpl` (the heat-threshold path). Its chamber arm (ceramics R3):
a thing whose immediate container is a `Burner` burns only while that
burner's `completeness()` ≥ `fire.air.flamingCompleteness` (0.6). Below it a
combustible shut in the chamber cannot catch, and a material whose transition
is pyrolysis (`requiresStarvedAir`) chars instead — the SAME dial, read by the
heat-work reconcile. At or above it, `Burner.catchContents()` (from the fire
tick and `settleFiring`) puts anything in the chamber hot enough to catch onto
the bed, where the burner's own path burns it and leaves its ash in the grate.
So the collier's lost burn is ordinary combustion. ⚠ The medium arm (nothing
burns under water) is underwater's and lands inside the same function.

### The clamp and the retort — charcoal is the wood's transition (ceramics ruling A)

`trade-fuel`'s two chambers are rows on the one rule. Every wood row
(`base-library/…/wood/*.yaml`) authors a `char` transition —
`requiresStarvedAir`, 650 K reference, keeping a quarter of its mass, wood
tar 0.08 L/kg as its volatile, a ladder of `half-burnt-wood` (degree 0.08,
*half-burnt*) then `charcoal` (0.8, *charred*) — and coal a `coke` one with
coal tar and coal gas. The lengths of cordwood BECOME charcoal; nothing is
minted. Charcoal, coke, ash, the tars, pitch and coal gas live in
`base-library` now (R4: commons name only commons).

- **The clamp** (`CharcoalPit`, `Container(Firebox)`) is a heap at ~720 K
  whose row draught (0.45 → completeness ≈ 0.53) keeps it STARVED; its row
  lays 40 kg of hazel down the chimney, which `char` lights. At 0.45 hazel
  chars through in about two and a half days; at 0.3 the heap runs ~30 K
  cooler and three days only reach brands; opened past ≈ 0.51 a flame can
  burn in it and the wood catches — ash in the grate. `char` sets the
  draught, lights, stamps `firedBy`, holds a three-day watch and settles;
  the threshold table (`outcomeFor`, `yieldFor`) and the three template
  rows it minted from are gone.
- **The retort** is SHUT: its row draught 0.3 (≈ 0.35) keeps the charge
  starved while the firebox still holds ≈ 1260 K on charcoal, so `fire in
  retort for 3 hours` chars wood and cokes coal, and the transition's
  volatiles go to `Retort.receiveVolatiles` (the kernel's `Burner` hook,
  B105) → the condenser on its head → the gasometer on the condenser; what
  none of them took, the thermal transition puts into the air. `draught wide` on a retort is a
  retort fired open: the charge catches.

### The burner's hooks (ceramics D3, D19)

- `heldTemperatureKFor(body)` — the heat this fire holds for one body
  (default `getHeldTemperatureK()`); `Thermal.heatSourceK()` asks it, so a
  clamp can be hotter in its fire channels than at its skin.
- `chamberVapors` — what vaporized in the chamber (`Material.vaporizesAboveK`:
  salt at ~1350 K), cleared when the fire goes out; a transition's `vapor:`
  map reads it at a rung.
- `settleFiring(by)` — the hold's end.

⚠⚠ **A firing still cannot go through `CraftingApi.craft`:** a craft picks
inputs from the actor's reach; a firing reads what is in the chamber.

### `platform/thing/Kiln.ts` is deleted

It was byte-identical to `Forge`, and `Oven`'s own comment gives the only real
distinction — *a chamber you load* versus *a fire you bring work to* — which
puts a kiln on the **oven** side. The generic row is retargeted onto `Oven` and
says why in its own header: **a kiln that cannot hold a charge cannot be fired,
which is why that row stood nowhere in the world for its whole life.**

⚠ A fixture note worth keeping: **`lit` defaults to TRUE on `BurnerMixin`**,
which is exactly why the smelt's unlit branch had no coverage for three builds.

### ⭐⭐⭐ The burner's fuel is a BED, and that is what refuels a furnace

**Closed by the fire build (2026-10).** This section read *"Still
unpriced: nothing refuels a furnace"* for two builds — the extraction
build's risk 7, which had widened rather than closed. The cause was one
field: a burner's fuel was a `'fuel'` `Reserve` carrying a **percentage
of nothing**. ⛔ It could not say what the fire was burning, nothing could
put more in, and a row authored it once — so **a forge arrived
pre-fuelled and could never be fed again, and no fire in the game could
run twice.**

Fuel is a **BED** now: `fuelBed` is kilograms keyed by `Material` path,
`fuelCapacityKg` is the vessel's size, and `maxBurnPowerW` is its
ceiling. From those three everything else derives —

- **heat** from the fuel's `heatOfCombustion` (a number authored on 26
  material rows and read by **nothing** before this) against the
  vessel's ceiling;
- **duration** from the mass in the bed;
- **completeness** as `clamp01(draught × airShare / completeAirShare)`,
  which is what makes the draught and the room's air *one lever seen from
  two sides*;
- **light** from soot, because ⭐⭐ **luminosity is incandescent soot** — a
  clean flame is dim and a sooty one is bright, so one dial moves heat,
  light and exhaust together and opening the vents makes a fire hotter,
  cleaner and **dimmer**;
- **exhaust** as CO₂ (`exhaust.litresPerKg`) plus, when the burn is
  incomplete, smoke — which is what the oxygen leg reads on the next
  tick, closing the loop: the fire fills the room, the room starves the
  fire.

Three verbs carry it, afforded by the burner itself in both
`environment` and `peers`: **`stoke`** (lay fuel in the bed — and it is
all-or-nothing, so an item heavier than the bed is refused), **`draught`**
(the air dial: `wide`/`open`/`half`/`low`/`banked` as SUBCOMMANDS, with a
bare `draught` reading where it stands), and **`cover`** (bank it under
its own ash — *couvre-feu*, which is why the word is not `bank`).

⚠ **What a row must now say.** `maxBurnPowerW` and `fuelCapacityKg` both
default to **0**, which is not dangerous but **inert** — and an inert
object reads as fine in every test and is dead in a player's hands. Two
shipped rows proved it in one week (the general store's lantern and the
chandlery candle, both lights for sale that could never be lit), so
`lint:light-sources` clause (h) now refuses a burner row with no fuel
path or no power, as clause (g) already refuses one that omits `lit:`.
⭐ The pair reads together: **(g)** catches a default that is *wrong*,
**(h)** two that are *empty*.

⚠ **Still open, and narrower than before:** fuel is now a thing somebody
carries, so every room with a fire a player is expected to light needs
fuel in reach — nine rooms did not, and were given some. ⭐ The same
question for a **carried** fire is the one that got missed: a bee smoker
has a 1 kg bed, and nothing under a kilo in the realm was fuel until the
sweep added a roll of sacking. Offered to
metal-chain-slate, which owns
the fuel chain, as a **stocking** question rather than a mechanism one.

## ⭐ Linings — a chamber furnace's part that wears (ceramics D19, W5)

A chamber furnace is lined with firebrick, and the lining is an assembly
**part line**, not a stat: `Oven`, `SmeltingFurnace` and `Retort` compose
`AssembledMixin(DurableMixin(…))` (⛔ not `Firebox` — a lamp, a campfire and
a clamp have no lining). A row authors the line in its `bill`:

```yaml
bill:
  parts:
    - { part: lining, template: /stuff/thing/firebrick, count: 8, role: wear }
  joints:
    - { key: lining, method: dry-laid, members: [lining] }
```

Shipped counts: the smelting furnace 8, the limekiln 24, the retort 12, the
commons `Kiln` (the Rejection glasshouse's) 30. The member row is the
COMMONS `/stuff/thing/firebrick` (generic-objects), so packs that do not
depend on the ceramics trade can name it; the potter's made firebrick
`extends:` it and fits the same line by row lineage.

- **Health** — `BurnerMixin.liningHealth()` = `(count − failed − (1 −
  condition)) ÷ count` over the `lining` line, or `null` with no lining.
- **The ceiling** — `heatCeilingFactor()` = `0.6 + 0.4 × health` (`1` with
  no lining) multiplies the vessel ceiling in `getHeldTemperatureK()`: a
  thinning lining holds less heat, so a worn smelting furnace stops reaching
  the bloom's heat before it stops lighting.
- **Wear** — `wearLining(heldK, heldS)`: `fire.lining.wearPerHour` bricks'
  worth an hour at the lining's own limit (its material's melting point),
  scaling as `(held ÷ limit)³` — a bread oven barely touches its bricks, a
  smelting furnace eats them. Worn brick by brick through the shipped
  `wearLine`, so `failed` counts bricks. `fire` wears it at settle; `smelt`
  wears it per run.
- **Burnt through** — below `fire.lining.burntBelow` (0.15) `ignite` refuses
  `lining-burnt-through`, naming the cure.
- **Relining** — `fit lining to <furnace>` (or `fit firebrick to …`) with
  firebrick in reach: the shipped replace arm. ⭐ A LINE word names the part
  by the line (`FitRequest.line`): the logic takes whatever in reach is the
  line's row or `extends:` it, and renews as many bricks as burnt through.
  The `dry-laid` joint row names `masonry`, so relining credits masonry.
- **The keeper** — `relines` (trade-ceramics brain): on shift, below
  `ceramics.lining.relineBelow` (0.4) the keeper walks an authored way to
  the counter, buys what burnt through on the house's card, walks back and
  fits — every act a literal verb. The Rejection glassblower (glasshouse
  kiln), smelterman (furnace), collier (retort) and storekeeper (limekiln)
  carry it; their rooms are `PersistentLocale`s (the quarry pit is already
  persistable), so a lining's wear survives a restart.
- ⚠ **Wear runs only where firebrick is sold.** A furnace row with a bill
  wears wherever it is propped; Terminus's oilworks props the same retort
  row with no firebrick line and no keeper (a seam for the works build).

**Crucibles** wear too: `SmeltController` wears a crucible
`ceramics.crucible.wearPerMelt` (0.12, about eight melts) per melt and
cracks it at broken (a `crack` flaw — it holds nothing).

## ⭐⭐ An oven on a pipe (the drilling build, 2026-10)

`Oven.fuelSlot()` answers with the interior of a **sealed vessel of
something that burns, standing ON it**. Everything else follows from the
burner substrate with no further code: fuel mass off the slot's litres
and the material's density, energy off its heat of combustion,
`consumeFuel` debiting litres, and `stoke` refusing `'no-bed'` while a
tank is coupled — *the fire is on the pipe; take the bladder off to burn
wood.* A row opts in with `placements: [on]`.

⚠⚠ **Why `Oven` and not `Firebox`.** A `Retort` is a Firebox that is
also a `Placing` host, and what is placed on a retort is its **product**
— the condenser, the gasometer the volatiles run into. On `Firebox` the
hook would need a guard to tell a fuel vessel from a product vessel, and
**a guard that re-narrows the host set is the tell that the host is
wrong.** On `Oven` it needs none: a sealed vessel of something flammable
standing on a cooking fire is fuel, which is true of every oven.

⭐ A **drained** coupled tank is a fire with **no fuel**, not a fire that
falls back to the bed. Silently reverting to the cordwood would make
running out of gas invisible and the bed's depletion inexplicable; the
remedy is to take the vessel off, and the refusal says so.

⚠ A fixture lesson: `ContainmentApi.move` is a no-op inside one
container, so moving a vessel from *on the oven* to *the floor of the
same room* leaves its placement stamp and it is still coupled. Taking
something off a hearth means taking it somewhere.

## The pump build (2026-10-09)

The bellows now speaks the `Pumpable` protocol (`lib/pump/Pumpable.ts`):
`BurnerMixin.planPump` / `completePump`, an instant act that toggles, with
the three refusals and two scenes `PumpController` printed before — moved
verbatim and pinned by a controller test. A furnace implements the
PROTOCOL; it is not a pump. See [pump.md](./pump.md).

## Cleanliness: the flue, the products, the limits (2026-10-17)

- `FluedMixin` on `Hearth` and `Oven` (⛔ not `Firebox`; the `Brazier` is
  its own flueless class): soot by kg burnt × incompleteness; a foul flue
  under a roaring fire catches. `sweep hearth with flue brush` clears it;
  `sweep hearth` rakes the grate's ash (`Burner.takeAsh`).
- A fuel's `combustionProducts` decide what its exhaust puts in the room
  (sulfur burns to sulfur dioxide); a substance's `flammableLimit` flashes
  on a flame (flour dust at the mill house). Incomplete exhaust soots every
  Soilable fixture in its room. See [cleanliness.md](./cleanliness.md).

## History — the potash build (2026-10-19)

A burner's ash carries its **alkali** beside its mass (`ashAlkaliKg`, each
burnt material's `ashFraction × alkaliShare`) and is minted by one public
method, `Burner.mintAshHeaps`, in heaps of `fire.ash.heapKg` carrying the
strength as a composition entry (`fire.ash.alkaliMaterial`). The charcoal
clamp's burnt-through ash uses it too, sized to the charge. `sweep` rakes
ANY burner's grate (it used to require a flue). A firing now concentrates
the constituents it keeps by `1 / massYield` and grades a Graded output by
its weakest Graded input. See [potash.md](potash.md).
