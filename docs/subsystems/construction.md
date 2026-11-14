# Construction — what is over and around a place

> Shipped by the rubber build (2026-10). Code: `lib/spatial/Enclosed.ts`
> (the vocabulary), `lib/spatial/Exposure.ts` (the answer),
> `lib/biome/Atmospheric.ts` (`construction()`, `exposure()`),
> `lib/structure/Structure.ts` (`storeys`). Gates: `lint:envelope`,
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

⭐ **A window onto the sky is enclosure** — glass as a layer here; a
window onto another place is a way's boundary, [boundary.md § Window](./boundary.md#window)'s.

Author a real material and a thickness; the physics is the material row's.
There is no U-value, no "insulated", no "roofed" — `lint:envelope` clause
(e) keeps the spec closed. An awning is `enclosure: none` + a canvas roof;
a walled yard is walls + `roof: none`.

⭐ **The getters return the RAW authored form** — a spec, a list, `'none'`,
or `null` when the row says nothing. The repair build's building upkeep
seeds its layer state from them: do not reshape them without telling it.

⭐ **The rung ladder** is row → class hook → universe. `enclosureDefaults()`
answers the universe default (the root biome row's `_enclosureMaterial`, granite 0.3 m — content since load-on-use round two, MR !575);
`roofDefaults()` answers `null` on a `Location` — **undeclared** — and a
class that knows its own construction declares it: a ROOM (`Room`, `Hall`
and their kept twins — roofed in its walls, `location.md` § The room rung),
an `ExitableVessel`
(its own material all round, or `none` for an open boat), a `Hive`, a
`ColdStore`, a `SealedCellar` (a metre of granite over it too).

## A structure is a set of places — and made of materials, as an Idea is

A `Structure` row says which places are its members (an `extent` prefix)
and which stand on which (`storeys`). ⭐ **It IS made of materials** (the
owner, paint build 2026-10-31) — but as an Idea, not a Thing: what it is
made of is a BILL, the materials it takes to erect and later to maintain
(the construction slate's). What it is NOT is a container of walls: the
walls and roofs are the places' own, and `lint:envelope` refuses a
Structure row that authors an envelope. Its OUTSIDE is its own — the coat
and the weather on it (`FacedMixin`) and the fabric under it (a `Durable`
condition) ([painting.md](./painting.md)).

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

## The landing — where a structure meets the weather

A structure is not a Thing or a Location, so a player engages it from a
place that touches it. ⭐ **A landing** is *a place that does not hold its
own air (`exposure().holdsAir()` false) with an exit into a member that is
enclosed or is the structure's entrance — or with the structure lying
alongside it* (a ship at a quay: the exit answers `alongsideNow()` —
`undefined` on an ordinary exit, whose far side is its path; the ship or
`null` on a way aboard, which never falls back to its authored berth, so a
sailed ship lands nowhere. The way aboard is the ship's own `BerthExit`
(harbors), installed when she makes fast and taken up when she casts off:
it answers her while she stands and lies at that landing, else `null`). Derived, never authored (`StructureBase.isLandedAt`; `Place.landedStructuresNow()`
from the room's side). A closet holds air; a neighbor's parlor is another
structure's member; a sailed ship's berth has nothing alongside; a member
courtyard open to the sky is a landing of its own building. The landed
structures enter the room's `here` pool, so `look cottage`, `measure
paint cottage` and `paint cottage` bind from the lane with no new MQL seed.

⭐ **A structure is not always a building** (paint D1, split in review):

| class | what it is |
|---|---|
| `lib/structure/Structure` | *these places are one thing* — membership, entrance, height, storeys, the landing, whether it is sold (`isTitleSold`) |
| `lib/structure/Building` (only inherited) | a structure with an OUTSIDE: a face painted from its landing, a `Durable` condition the weather wears once the coat is gone |
| `platform/idea/Building` | what a building row names (a `Singleton` per row); persists once sold |
| `HoldingBuilding` (`residence`) | a holding's building, minted per building row its programme names (`buildings:`, main first, on the programme row beside the plan's `extent` and `entrance`); its rooms are the plan's under that row's `extent` — authored, never grouped by how rooms connect |
| `platform/idea/Structure` | a bare structure row — a thing with no outside; today only the peak of Waponi Woo, until the mountain slate |
| `Ship` (`platform/idea/Ship`, `Durable(Faced(lib/structure/Structure))`) | a structure that moves; her hull takes coats wherever she is, and her seams are a `Durable` fabric under them (harbors) — worn, she leaks. A ship of a kind's members are her warren's, never a path (`Structure.memberExtent()` `@hook` answers `''`; every smack shares her kind's room rows). See [ship.md](./ship.md) |

⛔ **There is no blueprint row** (harbors review, 2026-11-03). A
`Structure` is only ever a thing standing in the world; the plan a warren
copies (an `extent` of room rows, an `entrance`) lives on its OWNER — a
holding's programme row (a warren, not a structure), or a ship kind's
own `Ship` row, whose `extent` is her plan rather than rooms of her own —
and is read by `lib/location/Floorplan.ts`. See
[holding.md § Plans](./holding.md).

A holding's keyed rooms have no path of their own, so a place asks its
WARREN which building it is in first (`Place.structureNow` →
`Warren.structureOf`, which the holding answers), then the location graph by
its identity path — what membership, the landing and `exposure()` read.

## The one rule, and the `Exposure` it computes

> **A face is exterior unless the space across it is an enclosed place of
> the same structure** — one with walls or a roof of its own.

`AtmosphericMixin.exposure()` applies it once per place into an
`Exposure` value object:

| field / method | answers |
|---|---|
| `top` | `open` (no roof) · `roof` · `ceiling` (an enclosed member stands `on:` this storey) |
| `walls[]` | four faces; `exterior: false` for a face onto an enclosed member (it carries the neighbor and the exit) |
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
neighbors change, re-checked at most once a real second (the envelope gate
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
  neighbor's LAST temperature: **shut ⇒ conduction** through both places'
  walls in series (symmetric), **open ⇒ flow** at the open-opening constant
  × min(V). One hop, recursion-free. `feel` names the sun and the warm room
  next door.

## Undeclared places — the counted transition

A place whose roof is undeclared reads the **legacy biome discriminator**
(`BiomeApi.isSkyExposed`): a sky biome ⇒ open (no walls unless authored),
anything else ⇒ its walls' material on five faces. ⭐ **The default is
open**: outdoor rows are never debt. The debt is the interiors, counted by
`lint:biome-sky` (`INTERIORS_FROM_THE_BIOME`, a falling list — 97 after the
rubber build, 37 after the room rung: a `Room`/`Hall` declares its roof as
its walls, so every interior on the rung paid its line). At zero both `Location` defaults become `none`, the legacy
branch goes, and `SkyExposedBiome` retires (biome-normalization-slate).

## Deferred

A shared wall between members with no exit; air mixing between rooms as a
mechanism; sun direction (one face weight becomes four); radiation to a
clear night sky (the exterior top face's third heat path); pane objects on
a construction face; a building's bill of goods (upkeep → the repair
build's W7; erection → structures-slate).
