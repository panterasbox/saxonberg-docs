# Light Subsystem

How the world's light is modelled, propagates, and is perceived.
Saxonberg is a text MUD, not a renderer — light here is an abstract
quantity that flows between rooms through `Boundary` substrate
(windows, doors, future portals), gets attenuated by exits, and is
perceived per-viewer through a `Sensor`-aware query layer that
threads through the cross-cutting perception pattern.

**Build state:** the 2026-06 perception substrate retired
`LightApi`. The propagation walk relocated into
`VisionModality.signalAt`, which outside consumers call directly on the
modality singleton (`vision().signalAt(loc)`; the thin
`PerceptionApi.signalAt` forwarder was removed in the 2026-07 antipattern
sweep — a `Modality.signalAt` is one object's own contract, so callers
narrow to the modality and call it). Vision-modality
domain helpers (`bandFor`, `applyBandShift`, `compareBand`,
`REQUIRED_BAND_FOR_DETAIL`, the `ShadowQuality` type) live in
`Light.ts` next to the `LightBand` vocabulary. Per-viewer queries
(`perceivedBand`, `canSee`, `viewerVisionProfile`, `shadowsAt`) are
static methods on `VisionModality`. The Light value object,
`AmbientLitMixin`, `LightSourceMixin`, and the conduit interfaces
ship unchanged. See [senses.md](./senses.md) for the cross-modality
substrate and [augmentation.md](./augmentation.md) for the
augment-conferred ESP modalities.

The subsystem ships:

- A pure-value `Light` class with `intensity` + `color` + `sources`.
- `VisionModality.signalAt(loc)` (sync substrate contract) —
  propagation walk; static convenience helpers `VisionModality.lightAt`,
  `bandAt`, `perceivedBand`, `canSee`, `shadowsAt`,
  `viewerVisionProfile`. (Callers reach it directly on the modality
  singleton, `vision().signalAt(loc)`.)
- `AmbientLitMixin` and `LightSourceMixin` for inherent ambient and
  for emitter-side contributions.
- `Boundary`, `BoundaryAnchor`, and the `Conduit` interfaces — the
  generic abstraction for "anything that connects two `Adornable`
  containers and gates channels (light, sight, movement, sound,
  smell) between them."
- Concrete `Boundary` users: `Window` (new) and the retrofitted
  `Door` (now also a Boundary).
- `Adornable` / `Adornment` mixins, the host-side machinery for
  fixtures that surface in `getFixtures()` (parallel to
  `getContents()`).

Sibling docs cover related ground without overlap:

- [perception.md](./perception.md) — the cross-cutting viewer-aware
  query pattern. All Light-side viewer parameters honor it:
  `Stuff & Sensor`, always explicit, no command-giver inference.
- [spatial.md](./spatial.md) — `Door`, `Exit`, `Sealable`,
  `ExitableVessel`, `Mobile` — the existing infrastructure this
  builds on. The Door retrofit details that intersect movement live
  there.
- [persistence.md](./persistence.md) — the scalar-default rule
  Light value-objects honor (intensity + color stored as named
  scalar fields; getter reconstructs).
- [collections.md](./collections.md) — fixture surface vocabulary
  (`addX` / `removeX` / `hasX` / `getXs`) used by `Adornable`.

## The Cast

| Type | Kind | Role |
|---|---|---|
| `Light` | value object | Immutable `intensity` (`Quantity<'lux'>`) + `colorTemperature` (`Quantity<'K'> \| null`) + `sources`. `Light.ZERO`, `add`, `attenuate`, `withColorTemperature`, `Light.of`, `Light.from`. Not Stuff. |
| `LightBand` | string union | `'pitch-black'` / `'very-dim'` / `'dim'` / `'lit'` / `'bright'` / `'blinding'`. The granularity controllers and prose check against. |
| `AmbientLitMixin` | mixin | Inherent ambient light a Container exposes. Persistent: `ambientIntensity` (lumens), `ambientColorTemperature` (Kelvin). Runtime API surfaces `Quantity` values via `getAmbientFlux` / `getAmbientColorTemperature`. |
| `LightSourceMixin` | mixin | Marks a Stuff as emitting light. Persistent: `emittedIntensity` (lumens), `emittedColorTemperature` (Kelvin). Runtime: `getEmittedFlux` / `getEmittedColorTemperature`. Fires `onLightSourceChanged` Witness hook (contract in `lib/perception/LightSource.ts`) on the immediate environment when emission changes. |
| `LightSourceObserver` | TypeScript interface | The Witness hook contract that `LightSourceMixin` fires. Lives in `lib/perception/LightSource.ts` next to `LightSourceMixin`, so consumer Containers (the receiver side) implement it from the same module. |
| `VisionModality` | `Modality` singleton | `signalAt(loc)` (the walk), `lightAt(loc)`, `bandAt(loc)`, `shadowsAt(loc)`, `perceivedBand(viewer, loc)`, `canSee(viewer, target, detail?)`, `viewerVisionProfile(viewer)`, `singleton()`. Outside callers reach these directly on the singleton (`vision().signalAt(loc)`). |
| `bandFor(lux)` | helper in `Light.ts` | Map a lux numeric to a `LightBand` via the registered tag-table. Sits next to `LIGHT_BANDS`. |
| `Adornable` | mixin | Container-side surface for non-portable attached Stuff (sconces, anchors). Composed onto `Location` and `Vessel`. `addFixture` / `removeFixture` / `getFixtures` plus typed walks `getFixtureBoundaries()` / `getFixtureLightSources()`. |
| `Adornment` | mixin | Host-side back-reference (`adornedTo`) and not-portable invariant. Concrete users: `BoundaryAnchor`, future fixtures. |
| `Boundary` | concrete `Thing` subclass | The two-anchor abstraction for cross-room channels. Just `extends Thing` — `Visible` / `Perceptible` come baked into Thing's default composition. Subclasses (`Window`, `Door`) compose `Sealable` for shutter / closed-door state. |
| `BoundaryAnchor` | concrete `Thing` subclass | `Adornment` Thing — the per-side proxy in each host's `getFixtures()`. Two anchors per Boundary. |
| `Conduit` | TypeScript interface | Channel-shape — `LightConduit`, `LineOfSight`, `MovementConduit`, `SoundConduit`, `SmellConduit`. Boundary subclasses implement (a subset of) these. |
| `Window` | concrete `Boundary` subclass | `SealableMixin(Boundary)`. Implements `LightConduit` + `LineOfSight` + `SmellConduit` + `SoundConduit`. Configurable `baseTransmissivity`, optional one-way overrides, optional `colorTint`. Shutters via `Sealable.open` (predicate `isOpen()`). |
| `Door` (retrofitted) | concrete `Boundary` subclass | `SealableMixin(Boundary)`. Implements all five conduits — `LightConduit`, `LineOfSight`, `MovementConduit`, `SoundConduit`, `SmellConduit`, all gated on `isOpen()`. Closed Door now blocks light, not just movement. |
| `BoundaryApi` | static API | `attachExistingBoundary({ boundary, hostA, hostB })`, `create({ factory, hostA, hostB })`, `destruct(boundary)`. |

## Class Hierarchy

```
Stuff
  ├── Idea
  ├── Thing                          (Visible + Perceptible + Tangible + Containable)
  │     ├── Boundary                 (just `extends Thing`)
  │     │     ├── Window             (Sealable + LightConduit + LineOfSight + SoundConduit + SmellConduit)
  │     │     └── Door (retrofit)    (Sealable + LightConduit + LineOfSight + MovementConduit + SoundConduit + SmellConduit)
  │     └── BoundaryAnchor           (Adornment)
  ├── Location                       (Adornable + Container, was Container only)
  └── Vessel                         (Adornable + Container + Containable, was Container + Containable)
```

The retrofit moves `Door`'s class hierarchy from
`VisibleMixin(PerceptibleMixin(SealableMixin(Thing)))` to
`SealableMixin(Boundary)` — Boundary already provides Visible +
Perceptible via Thing's default composition. The `attachedTo:
Set<Exit>` field, the `attachExit` / `detachExit` accessors, the
`Door.detach()` method, `Door.getKeywords()`, all existing `Door`
templates and call sites keep working unchanged.

## The Light Value Object

`Light` is immutable, factory-constructed, not a Stuff. It carries
real units throughout (see [quantities.md](./quantities.md) for
the substrate):

- `intensity: Quantity<'lux'>` — illuminance at the receiving
  surface. The propagation walk's lumen contributions are divided
  by the receiving Container's `getSizeScale()` (m²) before being
  wrapped here.
- `colorTemperature: Quantity<'K'> | null` — color temperature,
  atmospheric only. Multi-source mixing uses the **flux-weighted
  average**. The naming reserves bare `color` for a future
  abstraction layer above color temperature; concrete Kelvin
  values always carry "temperature" in the name.
- `sources: readonly LightSourceRef[]` — capped list (3) of
  contributing sources for prose attribution. Each entry carries
  `{ stuffId, flux, colorTemperature }`. **Runtime-only** —
  populated during the propagation walk; never persisted.

Operations:

- `Light.of(intensity, colorTemperature?, source?)` — primary
  constructor. Accepts `number | Quantity<'lux'>` for intensity;
  tag string / `Quantity<'K'>` / null for color temperature (string
  resolves through KELVIN_TAGS via `Quantity.parse(s, 'K')`).
  `Light.of(0)` returns the canonical `Light.ZERO`.
- `Light.from(value)` — coercion utility for externally-sourced
  data shapes (test fixtures, JSON over the wire). NOT used in the
  persistence path.
- `add(other)` — sum two Lights. Intensities (lux) add via
  `Quantity.add`; sources merge, cap-3 by flux; color temperature
  is the flux-weighted average across the merged source list.
- `attenuate(factor)` — multiply intensity by a 0..1 factor (a
  Boundary's transmissivity, or `EXIT_TAU` for cross-exit
  propagation). `factor ≤ 0 → Light.ZERO`; `factor ≥ 1 → this`.
- `withColorTemperature(value)` — copy with the color temperature
  overridden.

`LightBand` derivation lives in `bandFor(luxValue)`. The threshold
table (lux):

| Lux value          | Band         |
|--------------------|--------------|
| `< 1`              | pitch-black  |
| `>= 1`, `< 5`      | very-dim     |
| `>= 5`, `< 20`     | dim          |
| `>= 20`, `< 60`    | lit          |
| `>= 60`, `< 200`   | bright       |
| `>= 200`           | blinding     |

These thresholds are authored in `mud/config/quantity-tags.yaml`
under `lux/default` and load at boot via `QuantityApi.loadTagTables`.
`light.intensity.tag()` and `bandAt(loc)` agree by construction —
`bandFor` is a thin typed adapter that calls `Quantity.tag()` and
narrows the result to the `LightBand` union (membership-checked
against the `LIGHT_BANDS` `as const` tuple in
`lib/perception/Light.ts`, which is the single in-code source for
the band vocabulary).

Band shift / compare arithmetic in `VisionModality` (used by
`perceivedBand`'s species-vision adjustment, `canSee`'s required-
band threshold) delegates to the generic
`Quantity.shiftTag('lux', band, n)` / `Quantity.compareTag('lux',
a, b)` helpers — so a future thermal-K scale or sound-dB scale
gets the same banded-arithmetic substrate without each channel
re-implementing it. See
[../subsystems/quantities.md § Banded-arithmetic helpers](./quantities.md).

### Calibration — MUD-game scale, not photometric scale

The band thresholds are deliberately compressed for fantasy ambient.
Real-world lux runs from ~0.01 (moonlight) through ~500 (office
lighting) to ~120,000 (direct sunlight); ours top out at ~200. This
keeps fantasy-flux authoring (a candle ~12 lumens, a torch ~50, a
magic lantern ~200) reading as expected without authors having to
reach for hundreds of lumens to bump out of the dim bands.

Authoring guidance:

- Storage units are real (lumens for emission and ambient, m² for
  receiving area, Kelvin for color temperature).
- Thresholds are MUD-tuned. Tuning is content-driven; revisit when
  world content stresses the table.

### `getSizeScale()` per location kind

`getSizeScale()` returns the **effective receiving-surface area in
m²**. The propagation walk divides accumulated lumens by this scalar
to produce lux.

- `CartesianLocation.getSizeScale()` — reads its zone's `cellSize`
  (in **linear meters** as of the biome substrate, default `3.0`)
  and **squares it** to produce the m² receiving-surface area. A
  `cellSize: 3` zone reports a 9 m² scale. The override on
  `CartesianLocation.getZone()` narrows the return type to
  `CartesianZone | null` by invariant.
- `SphericalLocation.getSizeScale()` — returns its own radius (in
  m²; v1 uses the radius scalar directly rather than computing a
  spherical-shell area, deferring exact-physics fidelity until
  content needs it).

`cellSize` is interpreted as **linear meters** ([biome.md](./biome.md)
graduated it from informational to load-bearing). Tests that pre-
date the biome substrate may pin `cellSize: 1` to preserve the
prior 1 m² calibration.

### Authoring calibration (typical defaults)

`cellSize` is linear meters; the light scale is the square. Default
is `3.0` m (a typical 9 m² room).

| Room shape                | Linear `cellSize` | Light scale (m²) |
|---------------------------|-------------------|-------------------|
| Tight alcove / closet     | 1                 | 1                 |
| Small room (3m × 3m)      | 3                 | 9                 |
| Standard room (5m × 5m)   | 5                 | 25                |
| Hall (10m × 6m)           | ~8 (approx)       | ~64               |
| Outdoor plaza             | 10                | 100               |

| Light source                     | Suggested flux (lumens) |
|----------------------------------|--------------------------|
| Single candle                    | 10                       |
| Oil lamp / lantern               | 50                       |
| Modern bulb (room light)         | 800                      |
| Magic lantern (fantasy bright)   | 200                      |
| Direct sunlight (ambient slot)   | 8000                     |

Sample math:
- Two candles in a tight alcove (cellSize 1): 20 / 1 = 20 lux → `lit`.
- One candle in a standard room (cellSize 25): 10 / 25 = 0.4 lux → `pitch-black`.
- A modern bulb in a standard room: 800 / 25 = 32 lux → `lit`.
- Direct sunlight on a plaza: 8000 / 100 = 80 lux → `bright`.

## Propagation: `VisionModality.signalAt`

The single recursive walk. `VisionModality.signalAt(loc)` is the
substrate entry point — the `Modality.signalAt` override callers reach
directly on the vision singleton (`vision().signalAt(loc)`).
Internally it accumulates **flux** (lumens) plus per-source
attribution via `walkFluxAt`, then divides by the receiving
Container's `getSizeScale()` (m²) and wraps the result as a Light
with `Quantity<'lux'>` intensity. The static `VisionModality.lightAt(loc)`
is a thin convenience read — it resolves the vision singleton and calls
`signalAt` — so callers with a `loc` in hand get a `Light` directly.

### ⭐⭐⭐ An opening cannot make you brighter than what is through it

Legs (d) and (e) — light arriving from **another scope**, through a
boundary or a doorless exit — are **capped together at the brightest
neighbour's illuminance**. A scope's own light (ambient, contents,
fixtures, what you carry) sums normally.

*Three windows onto a 45-lux afternoon give you an afternoon, not three
of them.*

⚠⚠ **Why.** `EXIT_TAU` is `1.0` — *no extra dimming on exit traversal* —
so the leg added each neighbour's ENTIRE flux and divided by the
**receiver's** area, and a room in a chain of bright rooms came out
brighter than every room lighting it. At midday
`delight-road/crossroads`, which authors **600 lumens over 400 m²** (1.5
lux, and `lint:light-sources` calls it *"deliberate gloom"*), read
`blinding`. It was harmless until the envelope build, because before W0
no outdoor row carried 24 000 lumens — **a consumer written when the
input was small**, the same shape as two other findings in that build.

⭐ Found by a **browser walk at solar noon** (2026-09-25). The wire drive
could not see it: the harness boots at `t = 0` and `t = 0` is always
midnight, so a build about the sun had its daylight half untested live.

### ⚠⚠ Light is RESIDENT-dependent, and we say so plainly

A neighbour contributes light only while it is **loaded**. Rooms lazy-load
and the cold tail self-evicts ([residency.md](./residency.md)), so the
lux where you stand can change because something one hop away was paged
in or reaped — with nothing in the fiction having moved.

**This is a property of the architecture, not a bug, and it is not worked
around.** Two honest consequences, stated rather than hidden:

- a room can get brighter when a neighbour loads, and dimmer when one is
  evicted;
- the cap above is computed over the neighbours that are **resident at
  that moment**, so it moves with them.

⭐ It is survivable precisely because the effect is bounded: the cap means
a neighbour can only ever bring you *up to* its own illuminance, never
past it, so paging cannot produce a reading that no room in the
neighbourhood could justify. A player who notices will read it as the
light changing as the world settles around them, which is close enough to
true.

### ⭐ `blinding` describes, it does not withhold

Only `pitch-black` and `very-dim` withhold a room's description
(`LIGHT_BANDS_TOO_DARK_TO_DESCRIBE`). `blinding` has a **phrase and no
withholding**: *"The glare here is hard to look into"* rides along and
the room still describes itself.

The rule behind it: **staring at the sun is blinding; stacking ten
thousand torches in a room breaks the simulation but must not blind
you.** Too-bright-to-see is an old MUD flourish worth keeping as
*flavour*, and it must never take the room and its exits away — which is
what would turn a degenerate build into a player who cannot leave.

⚠ With the cap above, ordinary daylight cannot reach `blinding` at all,
so the band is reserved for something that genuinely declares itself.

### ⭐⭐ Two questions, and only one of them is `signalAt`

> **`signalAt(loc)` — what is the light doing now.**
> **`peakSignalAt(loc)` — how bright does this place GET.**

They were one question until the envelope build, because a scope's
ambient was an authored constant. A sky-lit scope now swings from
`pitch-black` to `bright` and back every game day, so a consumer that
asks *is this a good spot* rather than *what can I see* must say which
it means.

`Modality.peakSignalAt` defaults to `signalAt` — correct for every
modality whose field has no day in it — and **vision overrides it**,
walking with `CelestialApi.skyFactorDailyPeak()` (the brightest the sky
gets today, memoized per game day) in place of `skyFactorNow()`. ⭐ Only
the **sky leg** moves: a lamp reads the same either way, because a lamp
does not have a day.

⚠ **All of perception uses `signalAt`, and must** — a player sees what
is there now. The one consumer of the peak read today is
`GrowingMixin.sampleLux` ([husbandry.md](./husbandry.md)): a plant
profile's `luxHappyAt` is a claim about a PLACE, and every plant row in
the tree was authored when a scope's lux was a constant. Sampling the
instant instead meant **a lily on a sunny windowsill starved of light
because its owner watered it in the evening.**

⚠ The peak is found by scanning the day and then refining across the
winning sample's neighbours. Without the refinement the grid straddles
solar noon and under-reports by ~0.2% — and a peak that a `signalAt`
reading can exceed is not a peak.

```
walkFluxAt(loc, depth, visited) -> { flux, sources }:
  if depth > MAX_HOPS: return empty                   // depth budget
  if visited.has(loc.stuffId): return empty            // cycle guard
  visited.add(loc.stuffId)
  acc = empty

  if MixinApi.isAmbientLit(loc):                       // (a) ambient
    add(acc, loc.getAmbientFlux().rawValue(),
        loc.getAmbientColorTemperature()?.rawValue())

  for item in loc.getContents():                       // (b) contents-side emitters
    if MixinApi.isLightSource(item):
      add(acc, item.getEmittedFlux().rawValue(),
          item.getEmittedColorTemperature()?.rawValue())

  if MixinApi.isAdornable(loc):
    for fx in loc.getFixtureLightSources():            // (c) fixture-side emitters
      if MixinApi.isLightSource(fx):
        add(acc, fx.getEmittedFlux().rawValue(),
            fx.getEmittedColorTemperature()?.rawValue())

    for anchor in BoundaryAnchors among loc.getFixtures():
      boundary = anchor.getBoundary()                  // (d) cross-boundary propagation
      conduit  = boundary.getConduits().find(c => c.kind === 'light')
      tau = conduit.transmissivity(otherSide, anchor.side)
      if tau > 0:
        sub = walkFluxAt(otherHost, depth+1, visited)
        merge(acc, sub, tau)                            // flux scales by tau

  if MixinApi.isExitable(loc):
    for exit in loc.getObviousExits():                 // (e) cross-exit propagation
      if exit.getDoor(): continue                      //   (skip — Door's Boundary handles it)
      sub = walkFluxAt(exit.getDestination(), depth+1, visited)
      merge(acc, sub, EXIT_TAU)

  return acc

signalAt(loc):                                          // the Modality override
  acc = walkFluxAt(loc, 0, new Set<stuffId>())
  if acc empty: return Light.ZERO
  lux = acc.flux / loc.getSizeScale()                   // lumens / m²
  colorTemperature = fluxWeightedAverage(acc.sources)   // Quantity<'K'> | null
  return Light.from({ intensity: Quantity.of(lux, 'lux'),
                      colorTemperature, sources })

lightAt(loc):                                           // static convenience read
  return vision().signalAt(loc)                         // the vision singleton

bandAt(loc):
  return bandFor(lightAt(loc).intensity.rawValue())
```

Constants:

- `MAX_HOPS = 2` — the propagation walk truncates two containers
  from the query origin (one boundary hop + one further-room hop).
  Bounds the walk and matches the v1 "open door / window one room
  over" requirement.
- `EXIT_TAU = 1.0` — exits don't attenuate by themselves in v1. A
  future Door subclass that varies on traversal mode could drop
  this; for now exits leak fully.

The walk is **fully lazy**. No caches, no event-driven invalidation
in v1. If profiling later shows the walk is hot, a caching layer
hangs off the existing `onLightSourceChanged` Witness hook and a
future `onBoundaryStateChanged`.

### Why doored exits skip the cross-exit branch

Every `Door` is a `Boundary` after the retrofit, and every
`addBidirectionalExit({ door })` installs a per-side
`BoundaryAnchor` on each room. The cross-boundary branch (d)
already routes light through the Door's `LightConduit`. If the
cross-exit branch (e) also propagated through doored exits, every
neighbor would count twice when the door is open.

The skip preserves single-counting: doored neighbors flow through
the boundary; doorless neighbors flow through the exit at
`EXIT_TAU`.

### Eviction defense

The walk crosses up to `MAX_HOPS` containers. Future eviction
policy will need to "pin rooms within MAX_HOPS of any active
Interactive" to keep the walk's reads valid. v1 has no eviction;
the walk simply skips exits whose destination is path-only and not
currently loaded — reads as ZERO rather than paging in.

## `LightSourceMixin` and `AmbientLitMixin`

Both follow the same shape: typed Quantity values exposed at the
runtime API, decomposed into named scalar fields for storage per
the persistence subsystem's scalar-default rule. The setter
accepts numeric / string / Quantity input for authoring ergonomics;
storage is always primitive scalars.

**`AmbientLitMixin`:**

### ⭐⭐ Ambient light is DERIVED; only the exceptions are authored

The envelope build (2026-09-24) inverted this mixin's contract. Before
it, a room's ambient light was **a number somebody typed**: 84 of 195
Location rows authored one, 111 did not, and neither group had a reason
— the university-avenue crossing, the market square, the terminal hall
and the general store's shop floor authored nothing and were therefore
**pitch black at noon**, while a dormitory authored 30 lumens that meant
*lit by day* and burned all night.

Now:

- **A scope open to the sky follows the sun**, the moon's phase and
  altitude, and the cloud — because its biome says it is open to the
  sky. No row declares it, so no row can disagree with its own biome.
- **A scope that is not open to the sky has no ambient at all**, and is
  dark unless something in it is lit or light spills in through an
  opening.
- **The noon flux is derived too**: `light.sky.noonLux` (80) times the
  scope's own light-receiving area, so an unauthored room is lit
  correctly **for its size**. An authored `ambientIntensity` is a
  *calibration override* — authorial control over a place brighter or
  gloomier than its size suggests — and the sky still moves it.

The three departures are the `AmbientSource` vocabulary, and
`lint:light-sources` keeps a curated list of every row that declares
one:

| value | what it means |
|---|---|
| `null` (the ordinary case) | **derived** — ask the biome chain |
| `'sky'` | daylight reaches an enclosed room through an opening. The row must name that opening (`ambientOpening`, a detail id), which is what makes *"daylight reaches this room"* a claim a player can walk up to and look at rather than another number |
| `'glow'` | an inherent, always-on, non-sky ambient — a luminous cave, the holodeck floor. Does **not** follow the sun |
| `'none'` | a sky-exposed scope that is nonetheless dark — the bottom of a shaft |

⭐ The read is `isSkyLit()`, and the **weather's cloud dim now follows
it** rather than following sky *exposure*: a skylight is dimmed by cloud
exactly as a yard is. ⚠ An enclosed skylit room asks for the *exposed*
weather sample explicitly, because `computeResolved` runs the procgen
sky field only for exposed scopes and would otherwise dim a shop window
by a biome baseline nobody is standing under.

### ⭐⭐ A carried light lights the room

The propagation walk's contents leg reads the **room's** contents, and a
carried lamp is in the **carrier's** — so until the envelope build a
player could light a lantern, stand in the pitch dark, and have the
street read exactly as black as before. It never mattered, because
nowhere was dark.

The walk now looks **one level** into a room occupant. That is the
mirror of a rule the perception gate already has — *what you HOLD you
see in the light of where you stand, not in the dark of your own
pocket* — and the light travels the other way for the same reason: you
are holding it up, in this room.

⚠ One level, and only through a person. A lamp sealed in a chest inside
a pack is not lighting anything, and a general recursion would make the
hot path walk the world.

### ⚠ A band shift cannot manufacture photons

`Light.applyBandShift` is index arithmetic on the lux tag table, so a
`bandShift: +1` species read `very-dim` in a sealed cellar **with no
light in it at all**. Never noticed, because until this build nowhere
was dark. `perceiveFor` now applies the shift only to a non-zero signal:
seeing further into the dark is not seeing in the absence of light.

- Persistent: `ambientIntensity: number` (lumens — the noon
  **calibration**, not the light itself),
  `ambientColorTemperature: number | null` (Kelvin),
  `ambientSource: AmbientSource | null`, `ambientOpening: string | null`.
- Runtime:
  - `getAmbientFlux(): Quantity<'lumen'>`
  - `setAmbientFlux(Quantity<'lumen'> | number | string)` — string
    routes through `Quantity.parse(s, 'lumen')` so LUMEN_TAGS
    (`'glow'`, `'lamp'`, `'bright'`, …) round-trip via authoring.
  - `getAmbientColorTemperature(): Quantity<'K'> | null`
  - `setAmbientColorTemperature(Quantity<'K'> | string | number | null)`
    — string routes through `Quantity.parse(s, 'K')` so KELVIN_TAGS
    (`'warm'`, `'neutral'`, `'cool'`, …) round-trip.

**`LightSourceMixin`:**

- Persistent: `emittedIntensity: number` (lumens),
  `emittedColorTemperature: number | null` (Kelvin).
- Runtime: `getEmittedFlux` / `setEmittedFlux` /
  `getEmittedColorTemperature` / `setEmittedColorTemperature` —
  same shapes as ambient.
- Witness hook: `setEmittedFlux` and `setEmittedColorTemperature`
  fire `onLightSourceChanged(source, oldFlux, newFlux,
  oldColorTemperature, newColorTemperature)` on the immediate
  environment when the stored value actually changes. The hook
  contract is `LightSourceObserver` in `lib/perception/LightSource.ts`
  (alongside `LightSourceMixin`, so consumer Containers implement the
  receiver from the same module). v1 fans out to the IMMEDIATE environment
  only — no walk-up to outer Containers. A future caching layer
  may widen this radius.

To "extinguish" a source: `source.setEmittedFlux(0)`. To "ignite":
`source.setEmittedFlux(50)` (or any positive lumen value). There is
no `Switchable` composition, no fuel state, no `light X` verb in
v1 — that's content-authoring layered on top of this physics surface.

## Boundary Substrate

A `Boundary` connects two `Adornable` containers and gates channels
(light, sight, movement, sound, smell) between them. The runtime
triple is **boundary + anchorA + anchorB** — the Boundary itself
plus a `BoundaryAnchor` Thing in each host's `getFixtures()`.

### `Adornable` and `Adornment`

`Adornable` (composed on `Location` and `Vessel`) gives any
Container a `fixtures: Set<Stuff & Adornment>` collection parallel
to `contents`:

- `addFixture(f)` / `removeFixture(f)` / `hasFixture(f)` / `getFixtures()` — canonical collection vocabulary.
- `getFixtureBoundaries()` / `getFixtureLightSources()` — typed walks for the propagation walk and MQL queries.

`Adornment` (composed by `BoundaryAnchor`, future sconces) carries
the host back-reference (`adornedTo`) and the not-portable
invariant: `ContainmentApi.move` pre-flight rejects any attempt to
move an attached Adornment as inventory — the host must
`removeFixture` first (same shape `Door.detach()` uses).

### `BoundaryApi` install / teardown

```ts
BoundaryApi.attachExistingBoundary({ boundary, hostA, hostB });
// Constructs the two anchors, wires anchorA/anchorB on the boundary,
// calls hostA.addFixture(anchorA) and hostB.addFixture(anchorB).

BoundaryApi.create({ factory, hostA, hostB });
// Convenience: StuffApi.create(factory) + attachExistingBoundary in one call.

BoundaryApi.destruct(boundary);
// StuffApi.destruct(boundary) → boundary.onDestruct() walks anchors,
// removes from hosts, destructs anchors → boundary destroys.
```

Asymmetry guard: `attachExistingBoundary` rejects `hostA === hostB`
and rejects re-installing on a boundary that already has anchors
(would silently orphan the previous pair).

### Conduits

A `Conduit` is a TypeScript interface a Boundary subclass implements
(in v1, via `getConduits()` returning lightweight wrapper objects;
implementing two interfaces directly on one class collides on
`conduitKind`). Each conduit is per-channel, with directional
methods that take `BoundarySide` arguments so one-way channels can
encode it.

```ts
interface LightConduit    { conduitKind: 'light';
  transmissivity(from: BoundarySide, to: BoundarySide): number;
}
interface LineOfSight     { conduitKind: 'sight';
  canSeeThrough(from: BoundarySide, to: BoundarySide): boolean;
}
interface MovementConduit { conduitKind: 'movement';
  canPassThrough(from: BoundarySide, to: BoundarySide, mode: string): boolean;
}
interface SoundConduit    { conduitKind: 'sound';
  transmissivity(from: BoundarySide, to: BoundarySide): number;
}
interface SmellConduit    { conduitKind: 'smell';
  transmissivity(from: BoundarySide, to: BoundarySide): number;
}
```

Conduits MUST NOT cache — the boundary's own state (e.g. Sealable
`open` / `isOpen()`) participates in transmissivity and changes at
runtime.

### Naming: `BoundaryAnchor` vs `DoorBearing`

The codebase already had a `DoorBearingMixin` (a host-side mixin
giving `ExitableVessel` a single `door` field for synthesized
exits). `BoundaryAnchor` is a different concept: a `Thing`
subclass + Adornment that sits in `getFixtures()`. After the Door
retrofit, an `ExitableVessel` participates in BOTH — its
`DoorBearingMixin` field still drives the synthesized `'in'`/`'out'`
exits, AND a `(vessel, environment)` anchor pair migrates as the
vessel moves (see "Door retrofit" below). The two roles do not
overlap; do not unify them.

## `Window`

The first concrete Boundary user. Composition: `SealableMixin(Boundary)`.
It implements four conduits — `LightConduit`, `LineOfSight`,
`SmellConduit`, `SoundConduit` — all gated on its shutter state.

State (all persisted as scalars):

- `baseTransmissivity: number` (default 1.0) — symmetric pass-through factor when open.
- `aToBOverride: number | null` — one-way override for A→B.
- `bToAOverride: number | null` — one-way override for B→A.
- `colorTint: ColorTag | null` — stained glass.
- `open: boolean` (from `Sealable`, predicate `isOpen()`) — shutter state.

`transmissivity(from, to)`:

- Returns 0 if `!isOpen()` (shutters closed).
- Same-side reads (e.g., A→A — not used by the walk but sound
  semantically): `baseTransmissivity`.
- Otherwise: the relevant directional override if non-null, else
  `baseTransmissivity`.

`canSeeThrough(from, to)` — `transmissivity(from, to) > 0`.

The structured runtime API `getDirectionalOverrides()` /
`setDirectionalOverrides()` reconstructs / decomposes the
`{ aToB?, bToA? } | null` shape from the two stored scalars — same
pattern Light value objects use, applied to the per-side overrides.

Template authoring: Window is template-loadable like Door
(`class: '/lib/boundary/Window'`,
`hydratorClass: '/platform/idea/persistence/PersistentHydrator'`). Seed code
calls `BoundaryApi.attachExistingBoundary({ boundary: clonedWindow,
hostA: roomA, hostB: roomB })` to install on two rooms — mirrors
how `addBidirectionalExit({ door })` wires a templated Door.

## Door Retrofit

A `Door` is now a Boundary in addition to its prior identity.
Composition changed from `VisibleMixin(PerceptibleMixin(SealableMixin(Thing)))`
to `SealableMixin(Boundary)`. Boundary just `extends Thing`, and
Thing's default composition already bakes in Visible + Perceptible
+ Tangible + Containable.

Conduit registry: a Door advertises five conduits — `LightConduit`,
`LineOfSight`, `MovementConduit`, `SoundConduit`, `SmellConduit` —
all gated on `isOpen()`.

```ts
public override getConduits(): readonly Conduit[] {
  return [
    lightConduitFor(this),
    lineOfSightFor(this),
    movementConduitFor(this),
    smellConduitFor(this),
    soundConduitFor(this),
  ];
}

public transmissivity(_from, _to): number {
  return this.isOpen() ? 1 : 0;
}
public canSeeThrough(_from, _to): boolean { return this.isOpen(); }
public canPassThrough(_from, _to, _mode): boolean { return this.isOpen(); }
```

`Exit.canTraverse` is intentionally NOT modified — it already
consults `door.isOpen()` directly, which returns the same
answer as the new `MovementConduit.canPassThrough`. Routing
`canTraverse` through the conduit is deferred to a future Door
subclass that varies on traversal mode (`'squeeze'`, `'climb'`).

What stays the same:

- `Door.attachedTo: Set<Exit>` and the `attachExit` / `detachExit` /
  `hasAttached` / `getAttachedExits` accessors.
- `Door.getKeywords()` override (unions PerceptibleMixin keywords
  with shortDescription tokens).
- All existing `Door` templates (`class: '/lib/boundary/Door'`).
- All existing `addBidirectionalExit({ door })` call sites.

What changes observably: a closed Door now blocks light propagation
between its two rooms. The Door also appears as a fixture on each
side via its anchors — MQL fixture walks dedupe by `stuffId` so the
existing `getExitDoors()` path keeps `via.exit` attribution.

`Door.detach()` was extended to clear BOTH the existing
`attachedTo` Exit references AND the Boundary anchor pair (via
`super.detach()`). `Door.onDestruct` chains through Boundary's
which destructs the anchors.

### `ExitableVessel` migration

A vessel-door is a Boundary on the runtime triple `(vessel,
environment)`. As the vessel moves, the boundary follows.

- `ExitableVessel.onMoved(_from, to)` — destructs the existing
  anchor pair (via `Boundary._detachAndDestructAnchors()`, the
  subclass-bypass seam that doesn't touch Door's `attachedTo`),
  then re-installs the pair on `(vessel, to)`.
- `ExitableVessel.setDoor(door)` — destructs the old door's anchor
  pair (if any), updates the door reference, installs the new
  door's pair on `(vessel, currentEnv)`.

So a wardrobe with an open Door reads room ambient inside; a
closed Door reads ZERO inside; moving the wardrobe to a different
room migrates the boundary so the inside reads the new room's
ambient.

## Per-Viewer Perception

Viewer-aware queries follow the [perception.md](./perception.md)
pattern: viewer is `Stuff & Sensor`, always passed explicitly,
never inferred from execution context.

```ts
VisionModality.perceivedBand(viewer, loc): LightBand
VisionModality.canSee(viewer, target, detail?): boolean
VisionModality.viewerVisionProfile(viewer): VisionProfile
VisionModality.shadowsAt(loc): ShadowQuality
```

`perceivedBand` pipeline:

1. Compute raw `bandAt(loc)`.
2. Apply species vision profile via
   `viewerVisionProfile(viewer).bandShift` (v1: identity for every
   viewer; the Organism subsystem populates this later).
3. Apply the optional `viewer.perceivedBandModifier?(raw, loc)`
   seam — invoked via `ShadowApi.getShadows(viewer, ...)` so a
   Shadow on the viewer can intercept (BlindfoldShadow,
   NightVisionShadow, DarknessShadow). The seam method is declared
   only by Shadows, NOT by `Sensor` itself — Sensor implementors
   don't add it.

`canSee` pipeline:

1. Resolve target's environment. Non-Containable targets (Ideas,
   Zones) read true — they're queried by reference, not by
   location.
2. Compute `perceivedBand(viewer, env)` and compare against the
   detail-level threshold:

   | Detail | Required band |
   |--------|---------------|
   | `'shape'`  | `very-dim`    |
   | `'figure'` | `dim`         |
   | `'detail'` | `lit`         |
   | `'fine'`   | `bright`      |

3. Apply optional `canSeeOverride?(target, detail, raw)` Shadow
   seam — XRayShadow style overrides go here.

`shadowsAt(loc)` is a non-viewer-specific concealment surface for
future Hidden / Stealth mechanics — maps the band to one of
`absolute / deep / partial / faint / none`.

The `viewerVisionProfile` returns a constant `human-shaped` profile
in v1; the Organism subsystem will populate it from species data
later.

## MQL Fixture Exposure

`Adornable.getFixtureLightSources()` and `getFixtureBoundaries()`
parallel `ExitableMixin.getExitDoors()`. The MQL `here` scope walks
fixtures (deduped by `stuffId`) so `look candle`, `look window`,
`open window`, `close window` resolve naturally — same code path
as inventory items.

Door duplication: the fixture walk surfaces a Door once via
exit-doors and once via fixture-boundaries. The resolver dedupes
by `(stuff, via)`; the exit path wins, preserving `via.exit`
attribution.

Fixtures are NOT auto-listed in `look` room descriptions in v1.
Authors who want them mentioned write the long description
accordingly. Future enhancement is one helper in `LookController`.

## Persistence

Light value-objects do NOT round-trip as plain objects in MongoDB.
Per the [persistence.md](./persistence.md) scalar-default rule:

- `AmbientLitMixin` persists `ambientIntensity` (lumens) and
  `ambientColorTemperature` (Kelvin) as two scalar fields. The
  runtime API surfaces `Quantity<'lumen'>` and `Quantity<'K'> | null`
  via `getAmbientFlux` / `getAmbientColorTemperature` — string-tag
  authoring (`'warm'`, `'glow'`) coerces through the setter via
  `Quantity.parse(s, U)`.
- `LightSourceMixin` persists `emittedIntensity` +
  `emittedColorTemperature` with the same shape.
- `Window` persists `baseTransmissivity` + `aToBOverride` +
  `bToAOverride` + `colorTint` + (from Sealable) `open`. The
  pre-flatten `directionalOverrides: { aToB?, bToA? }` object is
  reconstructed at the runtime API layer only.
- `Door` persists nothing new beyond what `Sealable` already had
  (`open`).
- `Boundary` itself adds nothing persistent — runtime anchors are
  rebuilt at load time by seed code calling
  `BoundaryApi.attachExistingBoundary`.
- `BoundaryAnchor` carries `side` (`'A'` / `'B'`) but not the
  boundary back-reference (Stuff cross-references go via custom
  `persistenceHandler` shape; v1 anchor wiring is hydrate-only).

Each setter on a flat scalar field validates its primitive shape
independently — non-finite or negative intensity throws TypeError
at hydrate time; non-string color tag throws; out-of-range
transmissivity throws RangeError. The `PersistentHydrator`'s
bracket-assign is dumb — coercion lives at the setter, validation
fires early.

`Light` itself is a runtime-only value object; the propagation walk
constructs it from the stored scalars and never persists the
`Light` instance directly. See
[quantities.md § Persistence](./quantities.md#persistence) for the
broader Quantity persistence story.

## ⭐⭐ The sky, and the town's lamps

### The sky is a sync memo

`CelestialApi.skyIlluminanceFactor(profile, lat, t, opts?)` is pure
geometry — plain numbers in and out, beside the altitude formulas it is
checked against — and `skyFactorNow()` is a per-game-minute memo over it
on the `CelestialLogic` singleton. **Synchronous, because the light walk
is.** Three factors compose and none knows the other:

```
flux = the scope's own noon flux  ×  skyFactorNow()  ×  weatherDimFactor
```

The curve is a sun term, a moon term and a starlight floor. ⚠ The sun
term is `HORIZON_DIFFUSE + (1 − HORIZON_DIFFUSE)·sin α` above the
horizon and `HORIZON_DIFFUSE · 10^(α/decade)` below it, **and the two
branches meet**: written as a bare `sin α` above, the factor steps from
0 *up* to 0.1 as the sun sets and a street gets brighter at sunset. The
moon's phase term is **squared** (`((1 − cos 2πp)/2)²`), which is the
real thing — a half moon is about a tenth of a full one, not a half.

⚠⚠ **One sky for one world.** `skyFactorNow()` reads `EARTH_LIKE` at
`CAMPUS_LATITUDE` and asks no location, so Terminus and Rejection share
a sun. `CelestialLogic.profileFor` **throws by name** if a zone ever
authors a second celestial profile, and `lint:light-sources` clause (f)
refuses the row — per-zone profiles are a named deferred seam rather
than a silent half-truth.

### The town's lamps are a PROPERTY of the street

`PublicLightingMixin` on **`platform/location/Street`** — the singleton
public way, which is the only kind of place a town lights. A street
declares that the service runs here (`flux`, a `detail` id, and a `seniority`); whether it
is burning is **derived** — funded, after dusk, therefore lit. The lamps
themselves are prose: a dynamic detail saying *burning*, *standing cold*
or *out; it is daylight*. **Nothing is minted**, and
`check-light-sources` clause (e) refuses any class anywhere named for a
street lamp.

⭐ The test that decides object-or-property is *is it the target of a
verb?* The clock tower is not an object; the floor **is** one, because
`dig` has to bind it. Nobody binds a street lamp, and one identical
fuelled object per street would be forty-one fuel reserves reconciling
to produce the same number.

⚠ The escape hatch is forestry's four-representations pattern: if a
later build wants **one** lamp smashed or climbed, that lamp becomes a
prop at **that** spot and every other street keeps the property.

⭐ The generalization: **a light is an object where somebody acts on it,
and a property where the town runs it.** Indoors is where objects earn
their place — a tavern's lamp and a hearth are things you ignite, feed
and run out of.

## Out of scope (v1)

- ~~Time-of-day / world clock / outdoor ambient computation.~~ **Shipped
  in the envelope build** — see above.
- ~~Fire mechanics — `Combustible`, `Lightable`, `Burning` all
  deferred.~~ Shipped (the combustion build); a fuelled light is
  `platform/thing/Lamp`, a `FurnaceMixin` over a `LightSource`.
- `Switchable` and other generic state mixins.
- ~~Light-source archetypes (no canonical `Candle` / `Lamp`).~~ `Lamp`
  and `Hearth` both ship.
- Schedule integration — except `civic:lighting`, the nightly settle.
- Sound conduit / `Audible` mixin / sound propagation.
- Eager cache invalidation; v1 is fully lazy.
- Abstract color tints layered over color temperature (the
  `ColorTag` type alias is reserved for this future surface;
  `Window.colorTint` is the only current consumer).
- Mechanical penalties for fine actions in dim/blinding light
  (controller's call, not Light's concern).

## Cross-References

- [quantities.md](./quantities.md) — `Quantity<U>` substrate;
  the lux / lumen / Kelvin tag tables authored in
  `mud/config/quantity-tags.yaml` and loaded via
  `QuantityApi.loadTagTables`.
- [perception.md](./perception.md) — viewer-aware-query pattern.
- [spatial.md](./spatial.md) — Door, Exit, Sealable, Vessel,
  Adornable composition on Location/Vessel.
- [persistence.md](./persistence.md) — scalar-default rule,
  Marshaller framework.
- [collections.md](./collections.md) — fixture surface vocabulary.
- [docs/antipatterns.md](../antipatterns.md) — pre-asserted casts,
  redundant casts, scalar-default rule, instanceof vs. virtual
  vs. cast-by-invariant.
