# Construction — what is over and around a place

> Shipped by the rubber build (2026-10). Code: `lib/spatial/Enclosed.ts`
> (the vocabulary), `lib/spatial/Exposure.ts` (the answer),
> `lib/biome/Atmospheric.ts` (`construction()`, `exposure()`),
> `platform/idea/Structure.ts` (`storeys`). Gates: `lint:envelope`,
> `lint:biome-sky`.

## The one fact a place decides alone

A place says what bounds it. **`enclosure:`** is its walls and **`roof:`**
is what is over it. Each part is one layer, a list of layers outermost
first, or the literal `none`:

```yaml
enclosure:                              # the walls — a sawdust-filled double wall
  - { material: /stuff/idea/material/wood/pine, thicknessM: 0.025 }
  - { material: /stuff/idea/material/organic/sawdust, thicknessM: 0.3 }
  - { material: /stuff/idea/material/wood/pine, thicknessM: 0.025 }
roof: { material: /stuff/idea/material/glass/glass, thicknessM: 0.004 }
```

Author a real material and a thickness; the physics is the material row's.
There is no U-value, no "insulated", no "roofed" — `lint:envelope` clause
(e) keeps the spec closed. An awning is `enclosure: none` + a canvas roof;
a walled yard is walls + `roof: none`.

⭐ **The getters return the RAW authored form** — a spec, a list, `'none'`,
or `null` when the row says nothing. The repair build's building upkeep
seeds its layer state from them: do not reshape them without telling it.

⭐ **The rung ladder** is row → class hook → universe. `enclosureDefaults()`
answers the universe default (`enclosure.defaultMaterial`, granite 0.3 m);
`roofDefaults()` answers `null` on a `Location` — **undeclared** — and a
class that knows its own construction declares it: an `ExitableVessel`
(its own material all round, or `none` for an open boat), a `Hive`, a
`ColdStore`, a `SealedCellar` (a metre of granite over it too).

## A structure is a set of places, and made of nothing

A `Structure` row says which places are its members (an `extent` prefix)
and which stand on which (`storeys`), and **never** what anything is made
of — `lint:envelope` refuses a Structure row that authors a material.

```yaml
storeys:
  - { key: ground,  extent: …/location/ground }
  - { key: works,   extent: …/location/ground/works }   # longest prefix wins
  - { key: loft,    extent: …/location/loft,    on: works }
  - { key: terrace, extent: …/location/terrace, on: loft }
```

No `storeys` is one implicit storey; a storey with no `on` stands on the
ground; whether a storey is enclosed is read off its members' own walls
and roof (never authored on it). A lone hut is complete with no structure;
a structure with no member places (a landmark) still stands on the horizon.

## The one rule, and the `Exposure` it computes

> **A face is exterior unless the space across it is an enclosed place of
> the same structure** — one with walls or a roof of its own.

`AtmosphericMixin.exposure()` applies it once per place into an
`Exposure` value object:

| field / method | answers |
|---|---|
| `top` | `open` (no roof) · `roof` · `ceiling` (an enclosed member stands `on:` this storey) |
| `walls[]` | four faces; `exterior: false` for a face onto an enclosed member (it carries the neighbour and the exit) |
| `openSides` | doorless exits to a place that is not enclosed — an open-fronted barn's missing wall (declared places only) |
| `floorIsRoofOf` | a roof terrace: the place below, whose roof is this floor |
| `rainReaches()` | `top === 'open'` |
| `skyLightFraction()` / `skyVisible()` | 1 open, 0 under a ceiling, else the roof's transmittance + ¼ per transmitting wall |
| `holdsAir()` | top not open and walls not none — the place has air of its own |
| `windEnters()` | no walls, an open side, or an exterior opening standing open |
| `isOutside()` | open above and nothing around (the wayside survey) |
| `exteriorAreaM2()` | the envelope's loss area — exterior walls + the roof |
| `faceTransmittance(face)` | ⭐ the ONE place a face's light transmittance is computed |

⚠ A stated abstraction: the engine has no room geometry, so *which wall is
shared* is answered by *who is next door through an exit*. Openings are
exits; there is no per-wall vocabulary.

⚠ The cache: rebuilt when the place's structure or its resident
neighbours change, re-checked at most once a real second (the envelope gate
asks many times a command); door state is read live; an undeclared place
keys on its biome's sky bit (a biome resolves after its room).

## Who reads it

- **air & heat** — `envelopeApplies`, `airChangesPerHour`,
  `addAtmosphereContent` read `holdsAir()`; `isThreshold(far)` reads the far
  side's; `envelopeExposedAreaM2` reads `exteriorAreaM2()` (a place in no
  structure is five faces, as before). The weather's deviation reaches a
  place's air iff it holds none of its own (`BiomeLogic.airIsOutside`).
- **rain** — `computeResolved` takes `{air, rain}` (under an awning the air
  is the street's and the rain is not); the soil stamp, the floor's snow,
  `airSegmentsFor`'s rain and the storm fan-out read `rainReaches()`/`top`.
- **light** — `AmbientLit.isSkyLit()` / `skyLightFraction()` (the authored
  `sky`/`glow`/`none` rungs keep winning); `VisionModality` multiplies the
  sky term by the fraction. Light through matter is **Beer–Lambert**:
  `Material.lightAbsorption` (1/m; absent = opaque), a layer passes
  `e^(−α·t)`, a face the product over its layers. Glass α 8 (4 mm passes
  0.97); water 0.5; horn 200.
- **view** — horizons read `skyVisible()`; the wayside survey `isOutside()`.
- **ground** — a storey standing on another has no ground beneath it; every
  other floor keeps its shipped derivation.
- **heat (solar)** — a transparent exterior face admits
  `I_noon · skyFactor(t) · weatherDim · Σ T·A·w` (`light.solar.
  noonIrradianceWm2` 700, `sideWeight` 0.25), a `RoomHeatExchange` `fixedW`
  term, integrated in ≤ 1-game-hour pieces with the sun at each midpoint.
- **heat (between rooms)** — each interior face is a conductance to the
  neighbour's LAST temperature: **shut ⇒ conduction** through both places'
  walls in series (symmetric), **open ⇒ flow** at the open-opening constant
  × min(V). One hop, recursion-free. `feel` names the sun and the warm room
  next door.

## Undeclared places — the counted transition

A place whose roof is undeclared reads the **legacy biome discriminator**
(`BiomeApi.isSkyExposed`): a sky biome ⇒ open (no walls unless authored),
anything else ⇒ its walls' material on five faces. ⭐ **The default is
open**: outdoor rows are never debt. The debt is the interiors, counted by
`lint:biome-sky` (`INTERIORS_FROM_THE_BIOME`, a falling list — 97 after the
rubber build). At zero both `Location` defaults become `none`, the legacy
branch goes, and `SkyExposedBiome` retires (biome-normalization-slate).

## Deferred

A shared wall between members with no exit; air mixing between rooms as a
mechanism; sun direction (one face weight becomes four); radiation to a
clear night sky (the exterior top face's third heat path); pane objects on
a construction face; a building's bill of goods (upkeep → the repair
build's W7; erection → structures-slate).
