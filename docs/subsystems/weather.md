# Weather substrate (Wave 2)

The atmospheric **dynamics** driver: a procedural, lazy-compute-on-read
weather field that makes biome's static atmospheric *state* vary over
time, coherently, per locality. Weather is to biome what metabolism /
thermal are to vitals — biome is the state, weather is the thin driver
that deviates its reads
(weather-slate).

**Wave 2** (storms-and-wetness) adds the *coexistence resolve* (authored
pins + climate lean folded onto the procgen field, in precedence), a
cross-cutting **wetness** substrate, and the storm consequences —
electricity activation, thermal wet-collapse, Floor puddles, lightning
strikes, cloud light-dimming, and derived cloud forms. The governing
invariant: **every consumer reads the ONE resolved state
(`WeatherApi.resolveWeatherFor`), never the procgen field directly** — so
authored and modelled rain are indistinguishable downstream. See the
Wave 2 section below.

The defining property: **weather stores no state.**
`WeatherApi.weatherAt(time, locality)` is a pure deterministic function —
no simulation, no tick, no stored weather state. The same `(time,
locality)` always yields the same weather; tomorrow is computable today.
The only timer is the D4 cache-invalidation restamp, and it advances
nothing.

Source: `lib/weather/WeatherType.ts` (the one value-object),
`obj/api/WeatherLogic.ts` (the stateless logic singleton holding the
grammar as module-private functions), `api/weather.ts` (`WeatherApi`).
The grammar's segment model mirrors `BiomeLogic`'s chain walk beat-for-
beat; read [biome.md](./biome.md) and [address.md](./address.md)
alongside this.

## Wave 1 vs Wave 2

**Wave 1** shipped the procedural field + the thermal coupling (the
forcing consumer wired to read wind/humidity/temp/pressure statically) +
the `analyze weather` read surface. **Wave 2** (below) lands the "weather
with teeth" consumers — the coexistence resolve, wetness, electricity /
thermal wet-collapse, precipitation→puddles, storm lightning,
cloud→light-dimming, and derived cloud forms. Still deferred (a thinner
Wave-2 tail): fog→visibility, snow depth, vector wind, moving fronts.

## Module placement

| File | Category | Holds |
|---|---|---|
| `lib/weather/WeatherType.ts` | value-object / vocabulary | the `WeatherType` union + `WEATHER_TYPES`, `WEATHER_PROFILES`, `TRANSITIONS`, `SEASON_BIAS`, `ANCHOR_CANDIDATES`, `WEATHER_DEFAULTS` dials, and the `WeatherSample`/`WeatherForecast` I/O shapes — **consts + types only** |
| `obj/api/WeatherLogic.ts` | Api logic singleton (`/obj/api/weather`) | **stateless** (`extends Idea`, no `PostRegistrationMixin`); the gated compute methods + the grammar as **module-private** functions + a bounded pure-function season memo |
| `api/weather.ts` | Api | `WeatherApi` — thin gated forwarding shell |

The grammar **functions** (`segmentIndexAt`, `typeForSegment`,
`nextTypeFrom`, the hash, the seed, the interpolation) are module-private
inside `WeatherLogic` — the `BiomeLogic`/`AddressLogic` free-function
shape, **not** an exported `lib/` module (the export-discipline rule
forbids that). The dials/data live in the value-object; the behavior
lives on the singleton. There is **no** `WeatherRegistry` and **no**
stored handle — weather holds nothing.

## The grammar

### Vocabulary + profiles

Six coherent types: `clear`, `overcast`, `rain`, `storm`, `fog`, `snow`.
Each maps to a `WeatherTypeProfile` carrying a per-field
`WeatherDeviation` (signed Δ from the biome base) plus the Wave-2
`cloud` (0..1) and `precipitation` descriptors. Direction dials are
playtest, not plan decisions — only the **pressure sign** is load-
bearing: storm/rain deviations are **negative** (a storm reads low on
the Barometer). `clear` is zero everywhere, so a clear segment is
biome-identical.

### The segment model (D3)

Game-time is quantized into **segments** of `SEGMENT_LENGTH_S` (a few
game-hours) — a *piecewise-constant ambient*, exactly what thermal's
closed-form cooling assumes, and the natural restamp boundary (D4).

- **Segment index:** `segmentIndexAt(now) = floor(now / SEGMENT_LENGTH_S)`
  — integer, deterministic, process-independent.
- **Type for a segment** (`typeForSegment`, D-C **bounded warmup-anchor**):
  anchor at `floor(idx / GRAMMAR_WARMUP) * GRAMMAR_WARMUP` to a
  deterministic **calm** absolute type (`ANCHOR_CANDIDATES` — never
  rain/storm/snow), then iterate `nextTypeFrom` forward `idx mod
  GRAMMAR_WARMUP` steps. O(WARMUP), pure, process-stable — so a future
  segment computed now equals the segment after advancing the clock (the
  forecast property).
- **`nextTypeFrom(prev, season, roll)`** indexes `TRANSITIONS[prev]` (a
  weighted row, season-biased by `SEASON_BIAS[season]`) and picks by a
  deterministic roll. The transition table forbids implausible jumps —
  `clear` has no `storm` candidate, so it must pass through
  `overcast`/`rain`; `snow` is reachable only from `overcast`/`snow`
  (the coherence criterion).
- **Season** is global in Wave 1 (single `CAMPUS_LATITUDE`), computed by
  the pure `CelestialApi.seasonFor(EARTH_LIKE, t)`, memoized per
  year-relative segment. `SEASON_BIAS` leans snow heavy in winter and
  **zeroes it in summer**.
- **Interpolation** (D-D): within a segment the deviation lerps from the
  previous segment's targets to the current's over a configurable
  lead-in band (`INTERP_BAND`) measured from the start boundary; outside
  the band it is the current segment's targets. This keeps the deviation
  **continuous across boundaries** (no jump) while the *type* stays
  piecewise-constant (the type `analyze weather` reports is the current
  segment's).

## Locality binding (D1)

The procedural **seed derives from the covering Locality's claimed
address** — `localitySeed(locality) = hash(locality.getAddress()) ^
GLOBAL_BASE_SEED`; `null` (no covering Locality) uses the global seed
alone. **No field is added to `Locality`; `lib/address` is untouched.**
Different Localities / Region roots get different deterministic weather
automatically; sibling scopes under one Locality share it. The covering
Locality is resolved through `AddressApi.resolveLocalityFor` (the seam
the address build reserved) — so with no addressing authored every place
resolves `null` → global, and weather upgrades to per-Locality for free
as Localities land.

The **authored** climate-bias field (Narnia is polar) is the reserved
`Locality` tier-field home for a later wave; Wave 1 needs only
*variation*, which the address prefix supplies.

## The biome-deviation seam (D2)

Weather is **felt through biome's existing reads**. In `BiomeLogic`,
`resolveQuantityFor` folds the weather deviation in **after**
`runChainWalk` returns the base value, for the four weather-deviated
fields (temperature / humidity / wind / pressure) and **SkyExposed scopes
only**. Gravity / atmosphere never route here; the trace variants are
left un-weathered (they report biome-chain provenance — weather is a
separate additive surfaced by `analyze weather`).

```ts
const base = trace.value;
if (
  WEATHER_DEVIATED_FIELDS.has(fieldBare) &&   // cheap
  WeatherApi.isActive() &&                    // cheap (presence check)
  skyExposedWalk(scope)                        // cheap (containment walk)
) {
  const locality = await AddressApi.resolveLocalityFor(scope);  // one walk
  const dev = WeatherApi.deviationFor(locality, fieldBare, WorldClockApi.getNow());
  return base.add(dev);
}
return base;
```

- **Gate ordering (D-E)** — the cheap sync checks come **first**, so
  weather-absent or indoor scopes do zero extra work; only when all
  three pass does it resolve the covering Locality and add the deviation.
  Per-locality weather is *felt* (not just reported) — authoring a
  Locality with distinct weather is picked up by thermal/instruments with
  no follow-up.
- **Zero-when-absent / byte-identical** — `clear` is zero in every field
  and `base.add(zero)` equals `base`; and when weather is unconfigured
  the block is never entered at all (`isActive()` false), so the read is
  the very same instance. The regression guard asserts a SkyExposed
  scope's four resolves are byte-identical to a pre-weather baseline.
- **Soft import (D-A)** — `BiomeLogic` statically imports `WeatherApi`
  and calls `deviationFor` (zero when absent). `api/weather`'s **static**
  import graph never reaches `api/biome` (the lone weather→biome edge,
  `onBoundary`→`BiomeApi`, is a **dynamic** import), so there is no
  static init cycle. Enrichment, not a gate.

### Activation — the singleton's presence is the signal

There is no enable flag. Weather is "configured" **iff the `WeatherLogic`
singleton exists**. `WeatherApi.isActive()` is a non-creating
`findByTemplatePath` check, so a process that never touched weather (most
unit tests) reads zero deviation and biome-identical values — the
no-dependency guarantee. **Boot** forces the singleton into existence
when it computes the first boundary in `registerSystemSchedules`
(`WeatherApi.nextBoundaryAfter`), activating weather.

## The thermal coupling (D4) — cache invalidation, not a tick

No weather state is stored or advanced; the segment-boundary timer tells
thermal's stale cached `lastAmbientK` to refresh by re-resolving the
now-weathered ambient.

1. **Boundary schedule (scheduler-owned).** Registered once at boot in
   `WorldClockRegistry.registerSystemSchedules()`:
   `WorldClockApi.every(SEGMENT_LENGTH_S, () => WeatherApi.onBoundary(),
   { startAt: nextBoundary })`. It rides game-time (pause/scale
   propagate; `every` re-arms internally); the `ClockHandle` lives on the
   **scheduler**, not weather. The callback targets the stable
   `WeatherApi.onBoundary` facade, so it survives `WeatherLogic` HMR. The
   boundary is recomputed from `getNow()` every boot, never persisted.
2. **On fire (`onBoundary`)** — the **presence-gated** restamp fan-out
   (D-G): walk each `ConnectionApi.getAllInteractives()` to its avatar's
   room, dedupe by `stuffId`, sky-gate (`BiomeApi.isSkyExposed`), and
   call `BiomeApi.restampThermalContentsOf(room)` (D-F — the gated
   wrapper over the same fan-out `AtmosphericMixin` runs on an ambient
   shift; the mixin method stays private). `restamp` re-resolves
   `BiomeApi.resolveTemperatureFor`, which **already** folds the weather
   deviation (D2) — so `lastAmbientK` refreshes. `BiomeApi` is reached by
   a **dynamic import** inside `onBoundary` to keep weather's static
   graph biome-free (D-A).
3. **Presence-gating.** With no one connected the loop never runs — zero
   restamp work (the metabolism/thermal presence-freeze discipline).
4. **No weather call on thermal's sync path.** `getTemperature()` stays
   sync off the refreshed `lastAmbientK`; the address walk happens inside
   the async restamp, never inside a sync read.

## Read surface (D6)

`analyze weather [<location>]` — no instrument (parallel to `analyze sky`
/ `analyze atmosphere` / `analyze address`). Reports the current weather
**type** + cloud/precip descriptors, the four **per-field deviations**,
the **covering Locality** (or global / off-grid), and a short
**forecast** (the next `FORECAST_SEGMENTS` segment types — free from
determinism). The existing **Barometer** reads weather-deviated pressure
with no new code (a storm reads low — it already calls
`resolvePressureFor`).

## `WeatherApi` surface

```ts
// pure compute (sync, no I/O)
weatherAt(timeS, locality): WeatherSample

// the biome seam (cheap, SYNC — caller pre-resolved the Locality)
deviationFor(locality, field, timeS): Quantity<WeatherFieldUnit>

// forecast / sample reads (async — full AddressApi walk; the verb)
forecastFor(scope, segments?): Promise<WeatherForecast>
sampleFor(scope): Promise<WeatherSample>

// activation / boundary
isActive(): boolean                       // non-creating presence check
nextBoundaryAfter(timeS): Quantity<'s'>   // boot arms the schedule with this
onBoundary(): void                        // the WorldClock system-schedule callback

// the coexistence resolve (Wave 2 — the ONE read every consumer sees)
resolveWeatherFor(scope): Promise<ResolvedWeather>   // pin → procgen(lean) → biome
deviatedFieldFor(scope, locality, field, timeS): Quantity  // pin-aware biome fold
skyReadFor(scope): Promise<SkyRead>       // cloud form + hedged forecast tell
cloudFormFor(current, upcoming): CloudForm // pure genus derivation
onStormTick(): void                       // the weather:strike schedule callback
strikeIntervalSeconds(): number           // boot arms the strike schedule with this
```

The three-tier shape mirrors biome/address: `WeatherApi` (thin) →
`WeatherLogic` (`/obj/api/weather`, stateless, gated
`AnyOf(FromModule('/api/weather#WeatherApi'), SelfOnly)`). There is no
registry tier — weather is stateless, so the singleton itself is the
whole backend.

## Determinism + no-stored-state

- **`weatherAt` is pure** — inputs `(timeS, locality)` only; reads the
  dials, the profile/transition tables, and the deterministic season. No
  `Date.now`, no mutable module state (the season memo is a pure-function
  cache; the forced-type slot exists only for tests). Same inputs ⇒ same
  output across processes.
- **No stored weather state — none at all.** `WeatherLogic` holds no
  handle and no index. The boundary `ClockHandle` lives on the WorldClock
  scheduler and is re-derived from `getNow()` at boot. Weather
  reconstructs everything from game-time.

## Wave 2 — the coexistence resolve

The governing decision of Wave 2: **how authored and procedural weather
coexist.** The answer is a single **resolved atmospheric state** that
folds both as inputs in precedence, `WeatherApi.resolveWeatherFor(scope):
Promise<ResolvedWeather>`. Every Wave-2 consumer (wetness / thermal /
electricity / light / puddle / `analyze`) reads *this*; nobody calls
`weatherAt` / `deviationFor` directly except the resolver and the biome
field-fold. `ResolvedWeather = { sample, provenance, precipitationHere,
cloudForm }`.

**Precedence (outer wins):**

1. **Authored pin** — `{ type, mode }` (`WeatherPin`), authored at two
   tiers: a whole `Locality` (`Locality._weatherPin`, covers its address
   subtree) or a single scope (`AtmosphericMixin._weatherPin`, one room
   overriding within an otherwise-modelled Locality). Resolved by one
   shared sync upward walk (`resolveWeatherPin`: innermost-container-
   outward reading `_weatherPin`, then the covering Locality). Two
   **modes**: `frozen` (fully static — the type's profile deviation
   verbatim) and `alive` (the *type* is forced, but a per-segment
   `ALIVE_ANIM_MIN`-scaled animation breathes the intensity).
2. **Procgen (climate-lean-shaped)** — the Wave-1 field, its distribution
   biased by an authored `Locality._climateLean`
   (`Partial<Record<WeatherType, number>>`, a `SEASON_BIAS` sibling
   multiplied into `pickWeighted`; shapes only the procgen branch a pin
   outranks). Provenance `climate-leaned` when a lean applies, else
   `procgen`.
3. **Biome baseline** — an indoor / no-sky scope with no pin reads the
   static biome baseline (a `clear` sample, provenance `biome`) — weather
   *is* sky dynamics.

**The sky-gate split.** The **field deviations** (temp/humidity/wind/
pressure) stay SkyExposed-gated — Wave-1 byte-identical (an indoor scope
gets no storm wind-chill). The **`precipitationHere` read is *not*
sky-gated for a pin** (an authored indoor "weeping chamber" rains) but
*is* sky-gated for the procgen branch. This split lives in exactly one
place (`computeResolved`).

**The biome fold is now pin-aware.** `BiomeLogic.resolveQuantityFor`
keeps its cheap three-gate ordering (field-in-set → `isActive` →
`skyExposed`) and, only after they pass, calls
`WeatherApi.deviatedFieldFor(scope, …)` — which folds the *pinned* type's
deviation when a pin governs the scope, else the procgen deviation
(byte-identical to the old `deviationFor` when no pin applies).

## Wetness — the cross-cutting gauge

Any physical object — a cloak, firewood, a body — can be **wet**, tracked
as a per-object stored, decaying **saturation** `[0,1]` (`WetMixin`,
`lib/wetness/Wet.ts`; `MixinApi.isWet`). It is the upgrade the electricity
build flagged over its derived `isRainWet` stopgap. **Not** `ReservedMixin`
reuse — it borrows only the decomposed-scalar persistence shape (two plain
scalar fields, no keyed map), so it rides onto plain matter without the
biological reserve machinery.

**The host set is the matter seam.** "Can get wet" ⟺ "is matter" ⟺
`Tangible`, so `WetMixin` is composed at the three matter baselines —
`Thing`, `Vessel`, and `Agent` (which covers every `Creature` / `Avatar`).
A `Location` represents *space*, not matter — it is **not** `Tangible` (a
room has no material or mass, and nothing ever read them) and therefore not
`Wet`; a room's wetness reads as its `Floor` puddle (bulk) and its
contents, not a room-level scalar. (Removing the vestigial `TangibleMixin`
from `Location` — dead weight, nothing consumed it — is what makes
`Tangible` a clean matter seam.)

**Material-driven, from a real number.** Wetness reads its coefficient off
`Material` exactly as thermal reads `specificHeat` and electricity reads
`electricalConductivity` — the mixin holds the per-object *state*, the
Material supplies the *physics number*. That number is
`Material.waterAbsorptionCapacity`, a **real `Quantity<'%'>`** (the tabulated
ASTM-D570 water-a-material-holds-at-saturation, as a percent of dry mass:
wool ≈ 33 %, wood ≈ 28 %, flesh ≈ 25 %, leather ≈ 15 %, metals / glass ≈ 0),
**not** a fake 0–1 index (the discipline race.md's Material substrate
enforces). The **dry rate falls out of evaporation physics**: evaporation
sheds a roughly fixed water *mass* per hour (`wetness.evaporationRatePct`),
so a material that holds more water loses *saturation* slower —
`ds/dt = evaporationRate / capacity`. Wet wool lingers for game-hours; a wet
blade sheds at once. The **saturation gauge stays a normalized `0..1`** (so
even a non-absorbent steel blade still surface-soaks and conducts — the
electricity path is preserved); capacity governs only how fast it *dries*.
A materialless object reads `wetness.absorptionCapacityDefaultPct`.

- **Drainage is reconcile-on-read** over game-time, presence-frozen (the
  metabolism / harm / electricity-sustain idiom — no tick; first-touch /
  linkdead / far-past guards), warmth-accelerated (the object's own sync
  `ThermalMixin.getTemperature`) and **material-modulated**
  (`waterAbsorptionCapacity` above).
- **Accrual is *pushed***, not pulled: "is it raining here" is an *async*
  resolve, which the sync getter can't do — so precipitation exposure
  (the presence-gated boundary fan-out, source-indifferent) and immersion
  (the electricity co-immersion path) call `wet(delta)` from their async
  contexts. **"Shelter dries faster" emerges** — a sheltered object gets
  no accrual push, so the reconcile's drain wins.
- **Banded, never a number** — `getWetnessBand()` → `dry|damp|wet|soaked`;
  a non-dry object shows a band line in its long description (a
  `markupAugmenter`), raw value only via `analyze`.

## Wave-2 consequences

- **Electricity.** `ElectricityLogic.isWet` reads the **gauge**
  (`band ≥ wet`) — source-indifferent (rain wets via the gauge). The
  retired `isRainWet` stopgap + its raw global-procgen read are **gone**.
  A wet body reads ~100× lower resistance → a deadlier shock, driven
  end-to-end through the shipped `conduct`. The pool co-immersion path
  also **accrues** the gauge (see [electricity.md](./electricity.md)).
- **Thermal.** `ThermalRegulation.effectiveAmbient` folds an extra wet
  heat-loss term in the cold branch (`saturation ×
  thermal.wetHeatLossFactor`), read synchronously off the gauge at
  re-stamp — a wet body chills faster; byte-identical when dry (see
  [thermal.md](./thermal.md)).
- **Puddles.** The presence-gated boundary fan-out accrues an occupied
  scope's `Floor` surface-bulk pool under resolved rain
  (source-indifferent) and evaporates it otherwise (a clearer sky dries
  faster). A fresh rain pool (`/obj/material/bulk/water`, ~0.01 S/m) is
  weakly conductive, so a live wire / a strike in it shocks a bridged
  body through `conduct` with **no new glue** — the
  weather→bulk→electricity loop (see [bulk.md](./bulk.md)).
- **Storm lightning.** A `weather:strike` WorldClock system schedule
  fires a presence-gated fan-out: an occupied SkyExposed `storm` scope
  rolls `storm.strikeRate` and, on a hit, mints a transient
  `LightningStrike` (`EnergizedMixin`+`AudibleMixin`, engine-event
  content) routed through the shipped `ElectricityApi.conduct` (never a
  bespoke shock path), biases a direct hit onto the most-conductive
  attractor via `shockContact`, cracks a thunderclap, and is reaped. An
  empty scope is a harmless-but-heard flash. **No weather state stored** —
  the scheduler owns the handle; the callback recomputes from `getNow()`.
  This is the **mundane proof of the magic `Create·Lightning` bolt**.
- **Light.** `AmbientLit` carries a transient cached **weather dim
  factor** (the `lastAmbientK` cache-invalidation precedent); the boundary
  fan-out stamps `1 - cloudDimFactor·cloud` onto SkyExposed AmbientLit
  scopes, and `VisionModality.walkFluxAt` reads it *synchronously* —
  overcast / storm reads dimmer with no async weather call on the
  perception hot path. Default `1` = byte-identical.

## Cloud forms — a derived read + an honest forecast tell

The resolved weather carries a **derived visible cloud form** (a small
genus vocabulary — `CloudForm`: clear / cirrus / cirrostratus / cumulus /
stratus / nimbostratus / cumulonimbus). The Wave-1 `cloud` scalar is the
sky's *coverage*; the genus is its *form*, **described from** the resolved
type — never grown from a vertical-dynamics sim. Pure + deterministic:
`WeatherApi.cloudFormFor(current, upcoming)` = the base genus
(`CLOUD_FORM_BY_TYPE`), upgraded to the presage form
(`cirrus`/`cirrostratus`) when a fair sky (clear/overcast) has a rain/storm
front in the near forecast. Because our weather is deterministic the tell
is a *true, learnable* sky-reading signal (observe→predict→verify), but
the player-facing prose stays **honestly hedged** ("high wisps — a front
*may* be moving in", the barometer honesty caveat). **Presentation only:**
no consequence (light / wetness / electricity) reads the cloud *form* —
those read the resolved type / cloud-coverage (the one-resolve invariant).
Surfaced through `analyze weather` (which also renders provenance) and
`WeatherApi.skyReadFor` (presage-aware only for a genuinely modelled sky);
a dedicated `look up` prose surface is a small follow-up (the same
sync-perception / async-resolve barrier as light — it wants a cached-stamp
seam).

## The demonstrator — the Weeping Moor

`domain/moor/` (declarative content, the substation precedent): a
`CartesianZone` with a Locality-tier **alive storm** pin (the always-stormy
heath) and a scope-tier **frozen rain** pin (an *indoor* weeping chamber,
no sky) + authored humidity, each with an initially-dry surface-bulk
`Floor`. The integration test carries the **spine invariant** end-to-end:
an authored indoor-rain scope and a modelled procgen-storm scope produce
the **same** wetness accrual and the **same** puddle. Fixtures via
`adornments:`; no imperative construction; reachable by teleport (no
inbound exit — the treeline / substation content-standup precedent).

## Still-deferred seams

- **fog → visibility** (senses), **snow depth**, and other hazards
  (flood / blizzard / heat wave).
- **vector wind** (direction) — Wave-2 wind stays a scalar
  `Quantity<'m/s'>` (D5); direction feeds sailing / scent / fire spread.
- **a bulk-weight tie for wet garments** — cheap and optional; the
  electricity + thermal reads are the load-bearing consumers.
- **wet firewood / the fire coupling** — needs the Fire noun; the gauge
  ships the `wet` read, not the fire consumer.
- **a dedicated `look up` sky-prose surface** (above).
- **per-region latitude/longitude + moving fronts** — rides celestial's
  deferred planetary anchor, not weather (season is global).
- **magic `Create·Lightning`** — the frontier noun; this build ships the
  mundane strike on the reserved seam
  (capability-magic).

## Cross-references

- weather-slate — the seeding slate;
  this build is its Wave 1. Its open questions (the grammar, scalar wind,
  additive-deviation coupling, the Locality tier) are resolved here for
  Wave 1.
- [address.md](./address.md) — the locality substrate weather binds to
  (`resolveLocalityFor` → `Locality`/`null`; `Locality` as the home for
  the deferred authored-climate field).
- [biome.md](./biome.md) — the atmospheric state weather deviates; the
  `resolve*For` chain, `isSkyExposed` (the indoor gate),
  `restampThermalContentsOf` (the D-F seam).
- [thermal.md](./thermal.md) — the forcing consumer; the cached-ambient +
  `restamp` machinery the coupling rides; the Wave-2 wet heat-loss fold.
- [time.md](./time.md) — `WorldClockApi` (game-time + the boundary /
  strike schedules) and `CelestialApi.seasonFor` (season bias).
- [electricity.md](./electricity.md) — the `isWet` gauge activation + the
  `EnergizedMixin`/`conduct`/`shockContact` seam the strike + puddle loop
  reuse (the mundane `Create·Lightning`).
- [bulk.md](./bulk.md) — the `Floor` surface-bulk puddle sink the
  precipitation fan-out fills / evaporates.
- [reserve.md](./reserve.md) — the decomposed-scalar persistence shape the
  `WetMixin` gauge borrows (not `ReservedMixin` reuse).
- [perception.md](./perception.md) / light — the `AmbientLit` cached
  weather-dim factor + `VisionModality` sync read (cloud → light dimming).
