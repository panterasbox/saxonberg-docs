# Biome substrate

Saxonberg's first "physics of places" subsystem. Biomes are leaf
Idea templates carrying atmospheric defaults (temperature / pressure
/ humidity / gravity / atmosphere) plus ambient sensory texture
(sound + smell MML); Locations and Vessels compose `AtmosphericMixin`
to override those defaults at their own scope or per-Detail;
`BiomeApi.resolveXFor` walks innermost-container-outward through
containment ancestors, then the spatial zone, then the root universe
biome — returning the first override at any layer.

The whole substrate produces honest engineering numbers — Kelvin,
Pascals, m/s², m³ — so pedagogy can compute `n = PV/RT` against the
substrate's own state without needing a parallel "real units" track.


## ⭐⭐ The line a biome may not cross

> *A biome may say what the outside **AIR** is doing. It may never say
> how well a **STRUCTURE** holds heat.*

Climate is a fact about the air; construction is a fact about the room.
The envelope build (2026-09-24) deleted `_defaultTemperature` from
`indoor/baseline` — the 294 K decree every interior in the realm
inherited, in January, at 4 a.m., with the door open — and replaced it
with a derivation from what each room is **built of**.

⚠ Put construction on a biome and **one row warms every room that
references it**, with no fire in any of them, cascading to everything
downstream. That is the dishonest-physics failure in its purest form,
and it is the one that CASCADES. `lint:envelope` holds the line with two
clauses: (c) no `Biome` row authors `enclosure`, `thicknessM` or any
envelope key; (d) no row under the `/stuff/idea/biome/indoor/` admin
subtree authors `_defaultTemperature` — *"indoor" is a folder that means
an enclosure, and an enclosure's temperature is a structure's.*

⭐ What a biome legitimately still says: `underground/upper-workings`
authors 285 K, because a working **is** that temperature the year round.
That is the air of the mine. It gets no weather deviation, because it is
rock rather than sky.

⚠ Note that `indoor/cafeteria-atrium` is a **`SkyExposedBiome` under the
`indoor/` folder** — an atrium has a glass roof — which is why every
check here reads the **class**, never the path.

## Two trees, separated by job

The biome substrate uses **two independent structures** that meet at
the `Biome` template:

| Tree | Role | Where it lives | Mechanism |
|---|---|---|---|
| **Admin tree** | Ownership / write-access scoping ("biome team") | `FolderZone` templates at `/stuff/idea/biome/`, `/stuff/idea/biome/outdoor/`, `/stuff/idea/biome/indoor/`, etc. | templatePath organization |
| **Inheritance tree** | Atmospheric defaults inherited from parent biomes | `Biome` leaf templates with `_extendsBiomePath` refs | explicit Pattern-A ref (independent of path) |

Biomes are **leaves** in the admin tree — they're not folders; they
don't extend `Zone`. They're reference data, like `Material` and
`Species`. The inheritance tree they form is independent of where
they happen to live in the templatePath: a biome at one path can
extend a parent at any other path via its `_extendsBiomePath` ref.

Concretely, the shipped roster is deliberately slim — a handful of
demonstrative templates, parallel to how `Material` and `Species`
ship a representative sample rather than a content roster. Content
teams flesh out their own tree from here.

```
admin tree (FolderZones — ownership/write-access):

/stuff/idea/biome/                        FolderZone     ← biome team root
  universe.yaml                    Biome leaf     ← inheritance root
  outdoor/                         FolderZone     ← outdoor sub-team
    baseline.yaml                  SkyExposedBiome
    meadow.yaml                    SkyExposedBiome
  indoor/                          FolderZone     ← indoor sub-team
    baseline.yaml                  Biome
    cafeteria.yaml                 Biome
    cafeteria-atrium.yaml          SkyExposedBiome ← scenario C
```

> ⚠⚠ **The root is a `boot:` entry of `base-library`** — nothing else
> clones it. `BiomeApi.getRootBiome()` is a synchronous read of a
> resident singleton, and until 2026-09-17 no pack booted it: every
> field resolve that fell through to step 6, and the water pack's
> `Conduit` (which takes gravity from it), threw *"root universe biome
> is not loaded"* in every fresh world. The chain's own tests stand in
> "the root universe biome that boot would seed" — and boot did not.

```
inheritance tree (Biome._extendsBiomePath — independent of paths):

   universe
   ├── outdoor/baseline (SkyExposedBiome)
   │   └── outdoor/meadow            ← 3-deep chain
   └── indoor/baseline          ← ⭐ authors NO temperature since the
   │                              envelope build; see below
       └── indoor/cafeteria
           └── indoor/cafeteria-atrium ← scenario C: sibling-with-
                                         extends-ref (path-decoupled
                                         inheritance + SkyExposed
                                         override on the child)
```

Each shipped template earns its place demonstrating a substrate
property: universe (chain terminal), tier baselines (extends-chain
+ SkyExposed/plain split), the meadow leaf (3-deep chain), and the
cafeteria + cafeteria-atrium pair (scenario C — path-decoupled
inheritance).

**Why two trees:** the admin tree answers "who owns this template,
who can write to it." The inheritance tree answers "what defaults
does this biome inherit." Those are different concerns — coupling
them via path was the original design's mistake. A future "alien
planet" biome can sit at any admin path and explicitly extend
`/stuff/idea/biome/universe` (or any other biome); the path doesn't
constrain its inheritance.

## Organizing the inheritance tree — the spine

The `_extendsBiomePath` tree is the *mechanism*; this is the *principle* for
**where a biome sits** in it. **Order by atmospheric dominance** — each level
down is the next-biggest determiner of what a biome exists to carry (temp /
humidity / pressure / light / medium). Coarse → fine:

1. **medium** — air / water / vacuum (biggest lever: breathability, pressure,
   density)
2. **sky exposure** — outdoor / indoor / subterranean (weather, light, thermal
   stability)
3. **climate** — temperate / arid / tropical / polar (the temp + humidity
   envelope)
4. **surface / cover** — forested / grassland / sandy / **paved** / **urban** …
   (local humidity, thermal mass, the ground)

Placement rule: **coarser = bigger lever.** A node's home is found by asking
"how much does this move the actual numbers vs. its neighbors." Depth =
specificity, and that's fine — a deep biome like `outdoor/temperate/urban` is
**reusable** (any temperate city street).

**Cross-cutting attributes are cousins, not parent/child.** "urban" (paved +
built density + heat-island) and "paved" (bare hard surface — thermal mass,
runoff) both sit at the surface level on *different* branches: a rural highway
is paved-not-urban, a city park urban-not-paved.

**Reusable atmospheric *type* vs. place *identity* — the load-bearing line.** A
biome is a reusable environmental type; it is **never a specific place.**
"Terminus deferred-city street" is *not* a biome — `outdoor/temperate/urban`
is, and **Terminus is the `SpatialZone`** that uses it and adds the
place-specific overrides + mood. Single-inheritance means a leaf carries the
**dominant** atmospheric stack; leftover bits that are more *identity* than
*atmosphere* live on the **zone** as overrides (the spatial zone sits in the
resolve chain above the biome — see *The override chain*). Two
genuinely-atmospheric axes with no dominant (coastal-urban) → a combined leaf,
or let the zone carry one; rare, per-case.

## `Biome` class

A leaf `extends Idea`. Nine persistent fields:

- `name: string` (e.g., `'universe'`, `'temperate-baseline'`,
  `'quad'`)
- `_extendsBiomePath: string | null` — an identity ref to the parent
  biome; `null` on the root. `getExtendsBiome()` /
  `setExtendsBiome(value)` resolve via
  `StuffApi.findByTemplatePath` (HMR-safe); `getExtendsBiomePath()`
  exposes the raw string for the chain walker.
- `_defaultTemperature: Quantity<'K'> | null`
- `_defaultPressure: Quantity<'Pa'> | null`
- `_defaultHumidity: Quantity<'%'> | null`
- `_defaultGravity: Quantity<'m/s²'> | null`
- `_defaultAtmosphere: string | null`
- `_defaultAmbientSoundLevel: Quantity<'dB'> | null` — universe-root
  biome's value seeds the sync ambient floor on `SoundModality`'s
  depth-0 walk (2026-06 perception build). The full async biome-chain
  resolver for ambient sound is deferred; root-only is sufficient for
  v1 because content-room scenes either author explicit sources or
  inherit the same baseline everywhere.
- `_ambientSoundMml: string | null` — narrative prose (not
  propagating signal).
- `_ambientSmellMml: string | null` — narrative prose (not
  propagating signal). Per-room ambient smell ships via
  `SmellSourceMixin` emitters in the room rather than a typed biome
  field.

The thermal build added a `_defaultWind` field (with `_wind` /
`_detailWinds` on `AtmosphericMixin`) — a mechanical clone of the
humidity spine, mandatory on the universe-root seed, static-authored
for now and read by the body's wind-chill transform (weather will
drive it dynamically later). See [thermal.md](./thermal.md).

`null` on a Quantity / string default means "fall through to my
extends parent." A leaf biome carrying only `_ambientSoundMml`
inherits all four Quantity defaults from the chain.

`Biome` does NOT compose `SingletonMixin` in v1 — leaves room for
future procedural / time-of-day variance per clone.

The four Quantity-typed fields round-trip through
`QuantityMarshaller`s; setters are strict-on-unit and throw
TypeError on mismatch.

## `AtmosphericMixin`

Composed onto **both** `Location` and `Vessel` base classes. Pure
containers (Box, Backpack, treasure chest) do NOT compose it and
are atmospherically transparent (skipped by the chain walk).

Eleven persistent fields per host (sparse storage):

```
_biomePath                                   # identity biome ref
_temperature, _pressure, _humidity,          # five room/vessel-scope
_gravity, _atmosphere                        # bulk overrides
_detailTemperatures, _detailPressures,       # five per-detail maps
_detailHumidities, _detailGravities,         # (Record<string, V>)
_detailAtmospheres
```

`null` slots persist as absent; empty maps mean "no per-detail
overrides." A Vessel composing the mixin but setting nothing costs
five `null` fields + five empty objects and otherwise reads
identically to a non-composing pure container.

### ⚠⚠ `getBiome()` is a REGISTRY read, and the roster must be warmed

`getBiome()` resolves the authored `_biomePath` through
`BiomeApi.findByPath` → `StuffApi.findByTemplatePath` — an **identity ref
resolved on read**, which answers only with instances that are *already
live*. A room whose row cites a biome nobody stood up therefore answers
`null` **forever**, and every consequence of that room's biome goes quietly
inert: `isSkyExposed` returns its documented false-when-nothing-resolves, the
outward chain walk never gets past the room, and the ground build's on-grade
derivation hands an interior default to open country.

⭐ **`BiomeCatalogue` (`platform/idea/BiomeCatalogue.ts`) is what warms it** —
the `MaterialCatalogue` shape, self-warming at `postRegister`, eager through
the platform pack's `boot:` manifest (role `sync-read`). The roster is
**derived**: every root's `idea/biome/` subtree filtered to rows whose `class`
extends `Biome`, so a realm pack shipping
`/world/<place>/idea/biome/cavern` is warmed with nothing to edit — never an
allowlist of roots.

⚠ **Twice now.** `base-library/pack.yaml` still carries a `boot:` line for
`/stuff/idea/biome/universe` alone, with the note that it *"was never cloned
by anything … so `analyze power <thing>` threw 'root universe biome is not
loaded' in every fresh world, found by the grain-chain wire flow."* That fixed
the ROOT and left every other biome cold — which the ground build's drive
found by reading *"It is oak, laid as boards"* in a wood, a concrete apron and
a mine adit. The root's line stays: `getRootBiome()` **throws** when it is
cold, and belt and braces for the one row whose absence is fatal rather than
silent costs one line. ⭐ The general lesson is the one
lint-family.md's census-then-ratchet section keeps making:
**an enumerated boot list is a list somebody has to remember to extend.**

⚠ A room that cites **no** `_biomePath` at all is a different thing and is not
this mechanism's problem — nothing resolves because nothing was named. Two of
Rejection's rooms are in that state (the pithead yard and the adit) and read
the interior default as a result; `pnpm lint:ground --report` counts them.

### Method surface

```ts
interface Atmospheric {
  getBiome(): Biome | null;   // ⚠ registry read — see above
  setBiome(value: Biome | null): void;

  getTemperature(detailKey?: string): Promise<Quantity<'K'>>;
  setTemperature(value: Quantity<'K'> | null, detailKey?: string): void;
  // pressure / humidity / gravity follow the same shape.
  getAtmosphere(detailKey?: string): Promise<string>;
  setAtmosphere(value: string | null, detailKey?: string): void;
}
```

Getters are async because chain step 5 (spatial zone via
`Zone.lookupField`) is async. The getters are thin delegators to
`BiomeApi.resolve*For(this, detailKey)`.

Setters are sync. `set*(null)` clears the bulk override or deletes
the detail entry (per-detail key form). Each Quantity setter
asserts the runtime unit matches and throws TypeError on mismatch.

## The override chain

For any `(scope, detailKey?)` pair where `scope` is the innermost
`Stuff & Container`:

1. Walk innermost-container-outward through containment ancestors.
   For each ancestor that composes `AtmosphericMixin`:
   - **(a)** Exact detail override at `detailKey` on this ancestor
     — innermost scope only.
   - **(b)** Prefix-inherited detail override on this ancestor
     (longest-prefix-first walk — `hearth.embers` checks
     `hearth.embers` then `hearth`) — innermost scope only.
   - **(c)** Room-scope (bulk) override on this ancestor.
   - **(d)** Biome default with `_extendsBiomePath` walk on this
     ancestor's biome (if it has one): walks the explicit ref
     chain, consulting each biome for the field; first non-null
     value wins. Cycle-guarded (visited set + depth cap of 32).

2. **Spatial zone** — outermost Location's
   `getZone()?.lookupField<T>('atmosphere.<field>')`. Zones don't
   compose `AtmosphericMixin`; they participate via
   `Zone.lookupField`'s generic field-inheritance walk.

3. **Universe terminal** — `BiomeApi.getRootBiome()` returns the
   cached `/stuff/idea/biome/universe` template. Each `_defaultX` field
   on it is mandatory; the resolver throws a boot-invariant error
   if a field is unset at the root.

First override at any layer terminates the walk. Pure-container
ancestors (Box, Backpack, …) are skipped entirely.

⭐ **Pressure has a fourth step, and it resolves a circularity** (water
build). `measure altitude` used to compute `(P_sea − P_local)/(ρ·g)` from
a pressure an author had typed — an instrument reading back the number
that was invented to make it read. Now the walk runs to completion as
above, and **only if it fell all the way through to the root biome** does
`pressureFromElevation` replace that answer: it takes
`ZoneApi.elevationFor(scope)` and returns `P = P_sea − ρ·g·h`, tagged
`source: 'elevation'`, so the barometer reads a **cause**. An authored
pressure at *any* layer still wins — and the test for "nothing was
authored" is `sourcePath === ROOT_BIOME_PATH`, *not* the source tag,
because an outdoor room's biome ancestry reaches the universe row and
reports `biome-ancestor`. Three cases derive nothing and keep the root
value: no elevation in the chain, an elevation of exactly **0** (sea
level *is* the reference, so deriving would be a no-op that only costs a
walk), and a medium with no tabulated density (a vacuum has no barometric
anything). The weather deviation still rides on top of whichever
base won, which is also why a barometric altimeter is fooled by weather
in real life. See [watershed.md](./watershed.md).

### Worked traces

```
# Empty room with no biome — universe default.
resolveTemperatureFor(room) → 295 K   (source: 'universe')

# Room with biome /stuff/idea/biome/outdoor/temperate/quad (which extends
# temperate-baseline at 285 K).
resolveTemperatureFor(room) → 285 K   (source: 'biome-ancestor',
                                       sourcePath: '/stuff/idea/biome/outdoor/temperate/baseline')

# Room with biome AND room.setTemperature(Q(310, 'K')).
resolveTemperatureFor(room) → 310 K   (source: 'room')

# Same room + room.setTemperature(Q(800, 'K'), 'hearth').
resolveTemperatureFor(room, 'hearth') → 800 K        (source: 'detail')
resolveTemperatureFor(room, 'hearth.embers') → 800 K (source: 'detail-prefix')
resolveTemperatureFor(room, 'elsewhere') → 310 K     (source: 'room')

# Vessel inside the room with vessel.setAtmosphere('vacuum').
resolveAtmosphereFor(vessel) → 'vacuum'              (source: 'room')
resolveTemperatureFor(vessel) → 310 K                (source: 'room' on the OUTER room)
```

### Vessel cases (sparse storage falls out)

- **Porous** — vessel composes the mixin but sets no overrides.
  Every field is `null`; the chain walks straight through to the
  outer Location.
- **Sealed** — vessel overrides one or more fields. Those fields
  terminate at the vessel; un-overridden fields walk outward.
- **Partial sealing** — a bell jar that overrides atmosphere only.
  Atmosphere reads stop at the jar; temperature reads walk to the
  enclosing Location.
- **Nested** — Vessel B inside Vessel A. B's `null` fields walk to
  A; A's `null` fields walk to the outer Location.
- **Transparent** — a Box (pure container) inside a Location. The
  walk skips Box entirely; the Location's overrides win.

### Detail-key locality

Detail keys apply only at the innermost scope. A vessel-scope query
with `detailKey: 'hearth'` does NOT carry the detail key onto outer
ancestors — the vessel's own `_detailTemperatures` map is consulted
first, then the walk proceeds with the detail key dropped. Querying
the outer Location directly with `detailKey: 'hearth'` does see the
Location's hearth override.

## Atmosphere medium tags

v1 ships **three** tags from a private const map in `BiomeApi`:

| tag      | density at standard conditions |
|----------|--------------------------------|
| `air`    | 1.225 kg/m³                    |
| `water`  | 1000  kg/m³                    |
| `vacuum` | 0     kg/m³                    |

A parallel thermal-conductivity column ships with the thermal build:
`BiomeApi.conductivityOf(tag): Quantity<'W/(m·K)'>` reads
`ATMOSPHERE_CONDUCTIVITIES` (air ≈ 0.026, water ≈ 0.6, vacuum ≈ 1e-4 —
a tiny-nonzero floor so insulated vessels cool slowly, not never), the
dominant `R` term in the Thermal capability's τ = R·C. See
[thermal.md](./thermal.md).

`BiomeApi.densityOf(tag)` reads the map; throws on unknown tag.

A parallel **`breathable` column** (`ATMOSPHERE_BREATHABLE`: `air` true,
`water`/`vacuum` false) is read via **`BiomeApi.breathableOf(tag)`** — the
sibling of `densityOf`, throwing on unknown tags — the known-medium gate
the [respiration](./respiration.md) driver's medium trigger consults (an
air-breather drowns where it reads false). A third **`contaminant`
column** (`ATMOSPHERE_CONTAMINANT`, all `null` in v1) is **laid unread**:
the breathable≠safe / inhaled-toxin seam, gated on metabolism's
toxin-burden — no reader ships, no engine consults it.

`AtmosphericMixin.setAtmosphere(value)` accepts any string silently
— validation is the read-side concern (the density lookup or the
verb consumer's typed surface). If content authoring grows past the
three tags, the map grows by one line. If author-extensible
atmospheres become a real need, promote to an `Atmosphere extends
Idea` templated singleton parallel to `Material` at that point —
strictly additive.

No `AtmosphereDef` bundle, no `registerAtmosphere` extension
point — premature abstraction for the three v1 consumers.

## `SkyExposedMixin`

Capability seam for biomes whose Locations look out on the open
sky. `SkyExposedBiome extends SkyExposedMixin(Biome)` is the
concrete subclass outdoor biome leaves extend (parallel to
`RadioactiveMaterial`).

`isSkyExposed(): boolean` — the predicate is the entire substrate
in v1. Future `getCelestialBodies()` lands with its consuming
subsystem. The planned `getWeather()` seam **shipped not as a mixin
method** but as the `BiomeLogic` deviation seam (the
[weather](./weather.md) build): `resolveQuantityFor` folds
`WeatherApi.deviationFor` into the four weather-deviated fields
(temperature / humidity / wind / pressure) **for SkyExposed scopes
only**, zero-when-absent. `isSkyExposed` is the gate that seam
consults — weather is felt through biome's existing reads, not a new
biome method.

`BiomeApi.isSkyExposed(scope)` walks the containment chain outward
looking for the nearest atmospheric ancestor with a biome ref, then
narrows the biome via `MixinApi.isSkyExposed(biome)`. Returns
`false` when no biome resolves anywhere in the chain.

The atrium-in-cafeteria scenario authors a sibling biome
`/stuff/idea/biome/indoor/social/cafeteria-atrium` that extends
`SkyExposedBiome` and `_extendsBiomePath`-refs the cafeteria —
inheriting all of the cafeteria's profile while adding the
sky-exposed trait. The biome chain inherits shared defaults; the
sibling overrides only the trait.

## `BiomeApi`

Static surface:

```ts
findByPath(path: string): Biome | null
densityOf(tag: string): Quantity<'kg/m³'>        // throws on unknown tag
conductivityOf(tag: string): Quantity<'W/(m·K)'> // throws on unknown tag (thermal build)
getRootBiome(): Biome                             // cached; HMR invalidates
invalidateRootBiomeCache(): void

resolveTemperatureFor(scope, detailKey?): Promise<Quantity<'K'>>
resolvePressureFor(scope, detailKey?): Promise<Quantity<'Pa'>>
resolveHumidityFor(scope, detailKey?): Promise<Quantity<'%'>>
resolveGravityFor(scope, detailKey?): Promise<Quantity<'m/s²'>>
resolveAtmosphereFor(scope, detailKey?): Promise<string>
resolveWindFor(scope, detailKey?): Promise<Quantity<'m/s'>>  // thermal build

traceResolveTemperatureFor(scope, detailKey?): Promise<AtmosphericTrace<…>>
// + sibling trace variants (incl. traceResolveWindFor), and
// traceResolveAll which returns a typed bag for the `analyze
// atmosphere` verb.

isSkyExposed(scope): boolean
restampThermalContentsOf(room): void   // weather build (D-F)
```

`resolveWindFor` / `traceResolveWindFor` walk the same override chain
as the other resolvers; wind feeds the body's wind-chill transform.
See [thermal.md](./thermal.md).

The four **non-trace** Quantity resolvers
(`resolve{Temperature,Humidity,Wind,Pressure}For`) fold the active
weather deviation for SkyExposed scopes (the trace variants do not —
they report biome-chain provenance; weather is a separate additive).
`restampThermalContentsOf(room)` is the gated fan-out wrapper the
weather segment-boundary coupling calls to refresh thermal's cached
ambient. See [weather.md](./weather.md).

`AtmosphericTrace<V>` carries `{ value, source, sourcePath,
ancestorChain }` — provenance for verb rendering and tests.
`source` is one of `'detail' | 'detail-prefix' | 'room' | 'biome' |
'biome-ancestor' | 'zone' | 'universe'`.

No `findByPathOrThrow` / `allBiomes` convenience helpers in v1 —
add when a real consumer needs them.

## Derived geometry on `Location`

`Location` declares two abstract methods that concrete subclasses
override per their topology:

- `getVolume(): Quantity<'m³'> | null`
- `getCeilingHeight(): Quantity<'m'> | null`

### `CartesianLocation` — cube cell

`CartesianZone.cellSize` graduates from informational to
**load-bearing** with a default of **3.0 m** linear (a typical
room). Each cell is a cube:

| derived            | formula           | default (cellSize = 3) |
|--------------------|-------------------|------------------------|
| `getVolume()`      | `cellSize³`       | 27 m³                  |
| `getCeilingHeight()` | `cellSize`      | 3 m                    |
| `getSizeScale()`   | `cellSize²`       | 9 m² (light scale)     |

The light substrate's receiving-surface area divisor is now
**derived** from the linear cellSize by squaring it — rather than
authored separately as m². A 5 m × 5 m room is `cellSize: 5`; the
light walk sees a 25 m² scale. Tests pinning specific LightBand
values may need to set `cellSize: 1` for the prior 1 m² calibration.

### `SphericalLocation` — sphere + inscribed cube

A sphere of radius `r` reserves its full volume but presents an
inscribed cube as the usable floor-to-ceiling interior:

| derived              | formula              | radius 2 m       |
|----------------------|----------------------|------------------|
| `getVolume()`        | `(4/3)·π·r³`         | ≈ 33.51 m³       |
| `getCeilingHeight()` | `2r/√3` (cube side)  | ≈ 2.309 m        |

The asymmetry is intentional: volume answers "how much gas does
the room hold?" — the sphere; ceiling answers "how much vertical
headroom is there?" — the cube inscribed in the sphere.

### Reading the derived geometry

Both methods live on `AtmosphericMixin` (composed onto Location
AND Vessel), so any atmospheric scope exposes them directly — no
Api wrapper:

```ts
if (MixinApi.isAtmospheric(scope)) {
  const volume  = scope.getVolume();        // Quantity<'m³'> | null
  const ceiling = scope.getCeilingHeight(); // Quantity<'m'> | null
}
```

The mixin returns `null` from its default implementations; concrete
Location subclasses override per their topology. A future Vessel
subclass with a real interior volume (submarine, pressurized
module) overrides the same way.

## Instruments + verbs

> ⚠⚠ **This section was rewritten by the instrumentation build
> (2026-09-24). Its instruments are no longer this subsystem's.** They
> were six `Thing` **classes** under `/stuff/thing/instrument/`, each
> contributing `measure.yaml` and each owning a `measure <x>` subcommand.
> There are no such classes now: an instrument is a **row over
> `/platform/thing/ToolItem`** declaring a capability, and the channel it
> serves is a `Reading` row. See
> [instrumentation.md](./instrumentation.md).

What is still **this** subsystem's is the physics the channels read:

| channel | reads |
|---|---|
| `temperature` | `resolveTemperatureFor` |
| `pressure` | `resolvePressureFor` |
| `humidity` | `resolveHumidityFor` |
| `gravity` | `resolveGravityFor` |
| `atmosphere` | `resolveAtmosphereFor` |
| `altitude` | barometric delta vs the root biome |

⭐ The four that share one rung body (`temperature`, `pressure`,
`humidity`, `gravity`) do so through `lib/instrument/BiomeReading.ts`,
which names its MML channel by *returning* it — a fact
`MeasureChannel.totality` has to know about, and did not until it
reported a live channel as emitted by nobody.

⚠ The verb is **not** conferred by carrying the instrument any more.
`analyze`/`measure` ride `Avatar.commandContributions.self` and always
exist; what an instrument buys is the **ceiling**, and with none in reach
the refusal NAMES the instrument rather than the verb vanishing.

`altitude` derives `(P_sea − P_local) / (ρ · g)` from
`BiomeApi.getRootBiome().getDefaultPressure()` as the sea-level
reference and refuses in vacuum (ρ = 0 → no medium to define altitude).

⚠ `analyze atmosphere`'s developer half moved: *where did the engine get
that value from* is a question about the simulation, not about the
world, so it is **`trace atmosphere`** now — a free `system` verb beside
`affordances` and `errors`. The `traceResolve*` family is unchanged.

Instruments query at `actor.getContainer()` — a character inside a
sealed vessel reads the vessel's atmosphere; a character in a
porous vessel reads through to the outer Location via the chain.

`Altimeter` derives `(P_sea − P_local) / (ρ · g)` from
`BiomeApi.getRootBiome().getDefaultPressure()` as the sea-level
reference and refuses in vacuum (ρ = 0 → no medium to define
altitude).

`analyze atmosphere [detail]` runs without an instrument. It dumps
the full resolved state — biome path, spatial zone, per-field value
with provenance, derived volume / ceiling / density, per-detail
overrides — using the `traceResolve*` family. Cheap; doubles as a
developer debug tool.

Rendering convention: canonical units rendered via `q.formatMml()`
plus friendly tag in parentheses — `<quantity unit="K"
value="295">295 K</quantity> (warm)`. The pedagogical-seam toggle
for "friendly-only" vs "canonical-only" rendering is deferred to a
cross-cutting setting alongside sound's.


> ⭐ **A biome's readings carry a reveal level.** Temperature, pressure, humidity, wind, gravity and ambient sound level — the numbers an instrument takes. The ambient sound and smell stay open: that IS the description of arriving. `spoiler: 1` with
> `spoilerName: 0`, so a reader sees which properties exist with the
> values collapsed — **one click, not a lock**, and a reader who set
> `wiki.spoilerAppetite` higher never sees a collapse at all. The rule
> the sweep settled on: *collapse what the WORLD measures, never what
> the PLAYER operates.* Levels are declared on the field and therefore
> apply wherever the value surfaces. See
> [wiki.md](./wiki.md) § the reveal model.

---

## Cross-references

- [docs/subsystems/quantities.md](./quantities.md) — `Quantity<U>`
  + `QuantityMarshaller`; biome adds tag tables for K (thermal
  scale), Pa, %, m/s² and marshallers for Pa / % / m/s² / m / m³.
- [docs/subsystems/spatial.md](./spatial.md) — Location hierarchy +
  the `CartesianZone.cellSize` graduation.
- [docs/subsystems/zone.md](./zone.md) — `Zone.lookupField` for
  chain step 5; biome admin folders use `FolderZone`.
- [docs/subsystems/race.md](./race.md) — Material's `getMaterial`
  prefix-walk shape that `AtmosphericMixin`'s detail walk mirrors
  exactly; Material/Species are the precedents for the leaf-Idea
  + explicit-ref-inheritance pattern Biome now follows; the
  cross-link for a future per-species breathing gate.
- [docs/subsystems/light.md](./light.md) — `VisionModality`'s chain-walk
  shape; the receiving-surface area divisor is now derived from
  `cellSize²`.
- [docs/ref-shapes.md](../ref-shapes.md) — the identity ref for the
  `_biomePath` and `_extendsBiomePath` refs.
- [docs/subsystems/shell-environment.md](./shell-environment.md) —
  the universe defaults are NOT settings; the chain's terminal
  step reads from the root biome at `/stuff/idea/biome/universe`.

## History

Three substantive design shifts during the biome substrate build
(MR `panterasbox/saxonberg!26`):

1. **`Biome extends Idea`, not `Zone`.** The initial requirements
   doc had `Biome extends Zone` (parallel to `Clade extends Zone`)
   with templatePath-walking inheritance. MR review surfaced that
   this stretched Zone's meaning beyond its original "admin /
   ownership scope" intent. The refactor moved Biome to a leaf
   Idea with explicit `_extendsBiomePath` parent refs, and
   introduced `FolderZone` templates under `/stuff/idea/biome/` for the
   biome team's admin tree. Inheritance is now decoupled from
   templatePath organization. (Commits `2cc46c2` → `44ada01`.)
2. **`getVolume` / `getCeilingHeight` live on `AtmosphericMixin`,
   not `Location`.** Initially declared on Location with an
   `api/location.ts` wrapper; MR review surfaced that vessels can
   have meaningful interior volume too, and the wrapper Api was a
   misnomer (its parameters took `Stuff & Container`, not
   Location). The refactor moved the methods to AtmosphericMixin
   so both Location and Vessel hosts expose them directly; the
   wrapper Api was deleted. (Commit `a7166f6`.)
3. **Slim demonstrative roster, not 39 content commitments.** The
   plan called for 39 biome leaves across outdoor/temperate,
   underground, and five indoor sub-tiers. MR review pushed back
   on baking that much content into the substrate seed roster.
   The slim-down dropped 35 leaves + their tier folders, leaving
   8 demonstrative templates (universe + outdoor/indoor tier
   baselines + a couple of leaves + the cafeteria-atrium scenario-
   C showcase) — parallel to Material's 10-leaf and Species's
   8-template demonstrative rosters. (Commit `3650011`.)

The `_extendsBiomePath` ref-walk + `FolderZone` separation is the
shape that survives. Future biome content authoring extends from
`/stuff/idea/biome/universe` (or any other biome) via the explicit ref;
the path tree organizes ownership, not inheritance.

---

## ⭐ The air as a value object — `airFor` / `airSegmentsFor` (extraction W1)

`BiomeApi` gained two **sync** reads over three `syncChainWalk`s plus
`WeatherApi.deviatedFieldFor`:

```ts
airFor(scope): Evaporation
airSegmentsFor(scope, fromS, toS): AirSegment[]   // { air, durationS, rainMmPerH }
```

`lib/material/Evaporation.ts` is the value object and **the one place that says
how fast water leaves matter** — a vapour deficit × a wind term × a doubling per
10 K, capped at the boil (373 K), plus `rewets(moisture)`. It holds no statics,
because `lint:lib-statics` is a ratchet at 337/337.

⭐ **A decision worth knowing:** `airSegmentsFor` derives each segment's rain by
integrating that segment's own window through the **shipped**
`WeatherApi.precipitationBetween` rather than reading the rate table, so the
operator dials stay in one place and no new `WeatherApi` surface was added. And
it **never returns an empty list** — a caller summing segments must not
silently credit nothing.

⚠ Sync is the constraint that shaped it: the callers are reconcile-on-read
getters (a ham's water state, a pan's concentration) and a getter cannot await.

### `weatherLocality()` on `AtmosphericMixin`

The locality memo for the weather field, as `weatherLocality()` +
`resolveWeatherLocality()` with a private path/resolved/promise trio. ⚠ **Not
persisted, and no tri-state** — nothing integrates a backlog off it, so an
unresolved read misses the deviation once and then heals. That is cheaper than
correct here, and saying so is the point.
