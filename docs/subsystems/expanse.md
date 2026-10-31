# Expanse — a region you cross by setting a course

The maritime build's subsystem: a sea (and in principle a desert, an ice
sheet, a void) whose places are reached by **setting a course**, not by
walking an exit. ⭐ A ship on a RIVER is positioned on the river's line,
not the chart, and her course is a planned passage — see
[river.md](./river.md); so is the water beside a ship stopped at sea (the
`SeaWarren`). It is the second and third spatial registers beside the
room grid:

| register | what a position is | neighbours |
|---|---|---|
| the **grid** | a room with cardinal exits | the exits |
| a **passage** | a fraction along one dimension (a channel over a bar) | its two ends |
| a **field** | a coordinate | none — you steer |

⭐⭐ **Two layers, and nothing is ever inside the frame.** An expanse holds
**nodes** (`ExpanseNode` — places and passages, each a point of a
spherical graph relative to its expanse) and **bands** (`Band` — named
edges and regions with a width that confer cost and character). Both are Ideas. Behind a node may sit ordinary rooms — a
landing, a pool, a lighthouse's foot — authored exactly as any other
place. Nothing about the sea enters the location graph.

## The three tiers

| tier | where | what it adds |
|---|---|---|
| the **frame** | `lib/expanse/Expanse.ts` (kernel, abstract) | nodes, bands, composition, the craft registry, sight, traffic |
| the **medium** | `/system/water/idea/WaterExpanse` (water pack) | the column (depth, bottom), derived sea state, the water state fish read |
| the **realm's sea** | `/stuff/idea/WaterExpanse/<key>` (a row) | the Greywater: its nodes and bands beneath it |

The frame is a `SpatialZone` (it has an address, landmarks, stock) and
composes **no** Location half — `LocationZoneMixin` (`lib/zone/LocationZone.ts`)
is what holds rooms, and only `CartesianZone` / `SphericalZone` compose it.

## Position: the graph is spherical, the chart is lat/lon

⭐⭐ Two systems, and they never describe different things.

| layer | system | who reads it |
|---|---|---|
| **the graph** — nodes and bands | ⭐ **spherical**, relative to the expanse: a node is a focus `[rho, theta, phi]` + `radius` (`SphericalCoordinatesMixin`, the shape and units a `SphericalLocation` uses in its `SphericalZone` — meters; `theta` the bearing from the origin, `phi` the inclination, `90` on the surface) | authors |
| **the chart** — a craft's position, course, reckoning, sight, fix, the horizon's bearings | latitude and longitude (`GeoPosition`), plane sailing | sailors |
| **the bridge** | the expanse's authored `origin` (where the graph's centre lies on the chart) and `GeoPosition.atFocus(focus)` — the surface run `rho·sin(phi)` along `theta`, the depth `−rho·cos(phi)` | one place |

A node's chart position is **derived, never authored**
(`ExpanseNode.getExpansePosition()` = origin + focus), so moving the
origin moves the whole sea together. The node's **radius** is its size
and so *how close is there*: a course for it arrives inside the radius
(`getArrivalNm()`, `isAt(pos)`, `Expanse.nodeAt(pos)`), and once arrived
being there is an identity again. A node that authors no radius falls
back to `expanse.arrivalNm`. A node finds its expanse as its frame
(stamped by the compile) or its zone; it authors no `expanse:`.

`GeoPosition` (`lib/expanse/GeoPosition.ts`) is latitude, longitude and
`depthM` (always `0` until the underwater build); the arithmetic is
plane sailing (`cos(lat)` departure), which is what a navigator with a
traverse table does. Persisted as its plain record.

`PositionedMixin` (kernel) carries a CRAFT's chart position —
`expansePosition` and `expanse` (the row path). Composed by `Ship` and
the transport pack's `Boat` — never by `ExitableVessel` or `Mobile`,
⛔ never by a plain `Structure` (a building has no position of its own;
a light is where its rock's node is), and not by `ExpanseNode`, whose
position is the graph's. Two height hooks
— `sightHeightFor(observer)` and `getTargetHeightM()` — are what the
horizon reads.

⭐ A person's position is never their own: `ExpanseApi.craftAt(placePath)`
answers the craft a place IS (a launched boat) or is IN (the Ship
whose extent covers the room). Callers walk their own containment
(`getRootContainer()`) and pass the root's path — the Api takes paths
only (`lint:object-verbs`).

## Bands: cost and character, never connectivity

`BandExtent` is a closed two-word vocabulary: a **corridor** (two points
and a width — a rectangle on the local plane) or a **belt** (the box
between two corners — joins nothing). ⭐ Both are **authored on the
graph**, as foci relative to the origin (`widthM` in meters), and a
corridor that names `endpoints` and no `from`/`to` is **drawn between
those nodes** — an edge can never disagree with what it joins, and
moving a node moves it. `Band.getExtent()` resolves the authored shape
onto the chart (lat/lon), which is what the arithmetic runs on; a
failed resolve (an endpoint not yet compiled) is not cached. Inside or
not is the one question;
`crossings(a, b)` is an exact Liang–Barsky clip, so a boundary crossing
is a parameter along the track, orderable and deterministic.

⛔ Removing every band from a sea leaves it fully crossable. A band sets
you (`setKn` along `direction`), slows you (`cost.wayFactor`), leans the
wind (`lean: { directionDeg, strengthMps }` — its own shape, not
`ClimateLean`, which has no direction), wears gear (`gearHardness`), and
places things (`hazards`, `stock`). `endpoints` are a planning
affordance; a `confined` corridor is what a **linear** node rides, and
from such a node only its two ends are a course.

**Composition** (`Expanse.fieldAt`): a field value comes from the
NARROWEST band containing the position (the zone walk's innermost rule);
placed things union. ⚠ So a wide belt never wins anywhere a corridor
runs through it — keep a belt as narrow as the weather it must make
felt.

## The voyage — held by the craft

⭐⭐ **One craft, either frame** (river review, W7 — `lib/expanse/Frame.ts`).
A craft is positioned in a `Frame`: the sea's chart (an `Expanse`) or a
river's line (`LineFrameMixin`, [river.md](./river.md)). The craft never
branches on which: it asks its frame to plan a course
(`planCourse` — a bearing at sea, a worked passage on a river, or the
frame's refusal), where it puts her at a time (`craftPositionAt`), and what
a watch brought (`watchCraft` — what changed, how far she came, the stop).
`Voyaging` is the frame-agnostic craft: one `steer(request)`, `whereAtS`,
`placeAt(pos[, framePath])` (a frame change at a river's mouth is a
placement), `liveFrame`, and the craft's own `hullNow` · `burnFor` ·
`onStop` on either frame (a steamer burns at sea too). The navigator's plot
is the one chart-only surface: on a line you always know where you are
(`Frame.reckons`).

⚠ **A position is in one frame at a time, and the later write wins**
(`PositionedMixin`'s hydration setters): a chart position takes a craft off
any line and a line position takes her off the chart. Restore applies the
row and then the snapshot, so a ship whose row puts her on the Kestrel and
whose snapshot has her at sea comes back at sea — the setter used to throw
there, and the *Ostrander* restarted after going out over the bar did not
restore at all (river browser walk). `placeOnLine` keeps its guard for code
that moves a craft.

`VoyagingMixin` (kernel; `Ship`): ⭐⭐ the voyage's whole state
is **three persisted fields** — the stored `expansePosition` IS the fix,
plus `course` and `courseSetAtS`. `getExpansePosition()` answers the
DERIVED position while a course is set:

    positionNow = fix + course × (now − courseSetAt) + Σ set × time-in-band

integrated piecewise along the undisplaced track, split at every
boundary. ⭐ A course set for a node **stops where the track first comes
inside its radius** (`Expanse.craftPositionAt` over the raw `driftAtS`; sampled at
half the radius, bisected, memoised per course) — between watch beats
she lies off the node instead of sailing through it and snapping back at
the beat (the live browser walk, 2026-10-14). `rebase` is the one
writer; `steer` / `anchorHere` / `placeAt` are the acts. The drift and the
arrival search live on the `Expanse` (the chart's `Frame`), the watch's
chart half — boundaries, arrival, neglect — in `Expanse.watchCraft`.

`Voyage` (`lib/expanse/Voyage.ts`) is a **sustained engagement whose
actor is the craft** — nobody's `cancel` reaches it, logging out does
not end it, a craft under way with nobody aboard still advances. One
emission per watch (`expanse.watchGameHours`, default 4):

1. **the water says what CHANGED** — at every band boundary crossed, in
   order, the field just before against just after
   (`Expanse.changesAcross`, a `@hook`: the frame reports the SET — *a
   current takes her, setting north-east*, *the current lets her go*;
   `WaterExpanse` adds the sea getting up or going down and the water
   paling over shallow ground), then the sky over the deck against the
   last watch's (*Fog closes in.*). Never a position, never authored; a
   crossing that changes nothing perceptible says nothing;
2. **arrival** — a course set for a node arrives when she is inside the
   node's radius (`isAt`, the clamp's own metric) or the track passed
   within it (else `expanse.arrivalNm`); a bare bearing never does. ⛔
   `course <node>` while already inside it is refused (*"You are at
   Gannet Rock already."*);
3. **neglect** — the deck's `Durable` gear wears by
   `gearHardness × roughness × expanse.neglectWearPerWatch`, a quarter of
   that when the watch is manned. Nothing new persists.

⛔⛔ **The beat sees nothing and fixes nothing** (revision 1). Noticing a
sail is the lookout's job (`watch horizon`, the `keeps-watch` brain) and
fixing the position is the navigator's (`measure bearing`); a beat that
announced either made the seat decorative. It names no hazard either:
seeing the race ahead is somebody's job, and the water says what the race
does to her once she is in it.

`Persistable.onRestored()` re-arms it after a restart. ⚠ It does NOT await
the expanse's compile — the compile stands the ships up, so awaiting it
from inside a ship's restore deadlocks.

## The plot — the other position

`Plot` is the reckoning: the last fix carried along the courses steered,
knowing nothing of the set. `course` and `locate` report THIS, labelled
as a reckoning, with an uncertainty (`expanse.reckoningGrowthNmPerHour`)
worded by the reader's `navigation` band — never refused. A fix
collapses it, and every fix is an ACT: a bearing taken on a fixed thing
(`measure bearing` — to within `range × tan(bracket)`), a sounding over
ground with a depth of its own, arriving somewhere; a sun sight
(`measure latitude`) mends the latitude only. ⛔ Nothing fixes you by
being seen.

## What the frame registers

⭐⭐ **A Ship is a Structure whose position is a variable** (maritime
review, 2026-10-13). `Structure` is *these rooms are one building, seen
from outside* — membership, entrance, the look, `landmark`, `heightM` —
and nothing else; `Ship extends Structure` adds the position, the voyage
(Positioned, Voyaging, Engaged, Persistable), the deck and masthead
heights, the house, and the aboard hooks. *Is it a ship?* is `instanceof
Ship`, never a guard on a field. (Every Structure once composed the
whole voyage and told ships from buildings by `deckHeightM === null`.)

⭐⭐ **Nothing here is booted and nothing is found by searching**
(maritime review, 2026-10-14, which removed a boot-warmed
`StructureCatalogue` and `lint:structures-booted`). A structure is
ATTACHED to the extent its own `extent` names, and a ship to the sea its
`expanse` names — derived in the location graph's record, never written
twice ([location-graph.md § Attachments](./location-graph.md); the
maritime build's zone-side `structures:` list retired in the scan
sweep's review). A room joining its zone stands up what is attached to
it (kicked, never awaited inside the load: a ship standing up reaches
back for its sea, and a sea awaiting its ships would wait on itself).
*Which structure is this room in* is `NavigationApi.structureAt(path)`
— the innermost attached extent (`structureAtNow` for a synchronous
reader). The sea loads what is attached to it:
- **its ships** — the realm's own (the Hesper), counted in `craft()` at
  anchor or not, so a ship is alongside its landing before it ever sails.
  A ship a player builds is loaded by its crew being aboard;
- **its landmarks** — a BUILDING standing at one of its nodes (the node's
  landing is in the building's extent): the light on Gannet Rock. ⭐ It has
  no position of its own; it is where its rock's node is, so the
  position is authored once. In view from a craft's eye, never a sail,
  and the thing a bearing fixes by.

A sighting candidate (`Sighted`) carries its position and target height
from the expanse — a ship's own fix, a landmark's node — and
`Expanse.sightables()` is the synchronous list the horizon reads.

## Sight and traffic

`SightingChannel` = `visual | signal`. Visual range is
`1.17 (√h_obs + √h_tgt)` nautical miles, heights in feet; signal is
`expanse.signalRangeNm` regardless of height. Traffic is a SEEDED field
(`Expanse.trafficAt`): FNV over the expanse, the 0.1° cell and the game
hour against the narrowest band's `traffic` weight — the same for every
observer, unfarmable, nothing drawn.

## The water tier

`WaterExpanse` / `WaterBand` add `depthM`, `bottom`, `fetchKm`, `stock`.
⭐ **Sea state is derived and no row authors it**: wind direction from the
narrowest band's lean (else the prevailing wind), strength from the
weather field plus the lean, fetch and depth from the band — `Hs ≈
0.0016·U·√(F/g)`, capped at a developed sea, **steepened** where the
water is shallower than eight wave heights. Rendered in words.

A deck is a **shore of the sea under it** (maritime revision 3): the
deck row is a `/platform/location/Shore` whose `sea` feature cites the
expanse, and `WaterExpanse` — a `WaterSourceMixin` implementer — derives
the cell from the craft's position (a beach's from its nearest landing,
`peekShorePositionFor`) as `"<expanse path>@<lat>,<lon>"` quantized to
0.1°. The fishery reads that cell from its bands' stock — `fish`, `look
sea`, and a bucket over the side (salt water) all work from a deck.
Nothing is minted when a craft stops. (Was `Gunwale extends Shore`, a
prop; retired.)

## Knowledge: charts and pilots

`ChartedMixin` — *a source whose knowledge is claims*. A `Chart` read
writes `charted` claims; a `Pilot` paid (`pilot`) writes `told` claims
signed `toldBy`. Entries are authored as the source states them, so a
wrong chart is wrong in one row and never reconciled. See
[location-graph.md § Six channels](./location-graph.md).

## Verbs

| verb | afforded by | does |
|---|---|---|
| `course` | `Helm` (peers), `Boat` (inventory — granted inward to whoever sits in it) | reads the plot; lays a course by bearing or node |
| `anchor` | same | stops where she is |
| `hail` | same | signals a contact in signal range |
| `launch` / `recover` | `Boat` (inventory — you are in it) | over the side / back aboard |
| `pilot` | `Pilot` (peers) | buy told claims |
| `measure depth` / `measure latitude` | reading rows (`sounding` / `sighting`) | the sounding; the noon sight |
| `measure bearing` | reading row (`bearing`; the hand-bearing compass, or the eye at ±10°) | the bearing of a thing in view; a fix on a fixed one |
| `scan horizon` | `query` (alias `scan`; a bare `horizon` names a set) | what is in view beyond the room |
| `watch horizon` | `watch` (the stream verb's horizon form) | keep a watch: told only what changes |
| `swim` | `ShoreMixin` (inventory) — every place beside or in a water; the gate decides the way | a small water is a place: a `Pool` IS the water, swum in and swum INTO |
| `go aboard` / `go ashore` | `AboardExit` / `AshoreExit` | the far side recomputed every traverse |

⛔ No `fix` verb: the four fixes are four acts that already have verbs.

## Exits

`AboardExit` (on a node's landing) resolves to the entrance of whatever
SHIP lies at that node now; `AshoreExit` (on a ship's entrance) to the
landing of the node she lies at. Both refuse in words otherwise. Their
lookups (which node is this landing, which craft is this deck's) run
inside the async `canTraverse` at the moment of asking, cached after
the first — the one gate every crossing goes through (see
[locomotion.md](./locomotion.md)).

## Perception at range (revision 1)

⭐⭐ **Pull, never push.** Nothing on the horizon is ever announced to
anybody: somebody looks (`scan horizon`) or keeps a watch (`watch
horizon`, `HorizonWatch` — an engagement on the watcher's attention that
says only what came into or dropped out of view). Push comes only from
emitters that announce themselves (a bell, a gun, a signal).

- **What is in view** is `PerceptionApi.horizonFor(viewer)`, and it is
  ⭐⭐ **SYNCHRONOUS** — it is the MQL `horizon` seed's pool and the
  resolver cannot wait, so every MQL caller gets the truth with one call.
  It reads only warm state, and ⭐⭐ **only through the scope that owns
  each answer — never a registry-wide read** (maritime review,
  2026-10-13): *which craft am I aboard* is the room's own zone's
  `structureOf`; *what can I see at sea* is that craft's expanse's
  (`sightables()`); *what stands in a named zone* is that zone's
  `getStructures()`. ⭐ **What a place can see loads with the place**
  (`LocationZone.lookOutFrom`, 2026-10-14): when a zone gets its first
  room, the zones and seas its `horizon` names are stood up, and with
  them the structures they name — so a scan reads them resident without
  anything at boot. It cannot cascade: a zone stood up only because
  somebody can SEE it has no room, so it never looks out itself. Also read: the resident zones' fields
  (`Zone.peekField`, the sync sibling of `lookupField`), the compiled sea,
  and the cached sky (`WeatherApi.peekSkyType`). A sky nobody has read yet
  counts as clear once and starts its read; a sea nobody has compiled
  shows a height ashore nothing once and starts compiling. Afloat: the
  craft and landmarks on her sea from the viewer's own eye (`Ship.sightHeightFor` — the lookout seat is the
  masthead), nothing below decks. Ashore: the nearest `horizon`
  declaration (a room's, then the zone walk) names ZONES; a Structure in
  one is in view from a sky-exposed room when `(heightM + |Δelevation|) ×
  perception.horizonRatio ≥` the distance the zones' spans imply
  (`CartesianZone.getExtentM`). A zone naming an EXPANSE is read like a
  deck: from the nearest landing (`Expanse.peekShorePositionFor`) at the
  zone's elevation, through the one sync sighting core
  (`Expanse.sightingsAmong`, which `contactsFrom` also uses over the sea's
  own registry) — the headland raises the light, the strand below it does
  not.
- **A Structure is Visible** (+ Detailed + Perceptible — `Species`'s and
  `Material`'s precedent): described once, the ordinary way; its inside is
  its member rooms. `landmark: true` FEATURES it in the summary; a room's
  `featured` features others. Whether it can be seen is derived.
- ⛔ **The engine appends nothing to a room's prose.** A room that mentions
  the tower does so in its own words, for its own reason (the crossing's
  `tower` detail tells the time).
- **Weather shortens sight**: visual range × the observer's sky (`fog
  0.1 · storm 0.3 · snow 0.4 · rain 0.6`, `expanse.visibility.<type>`); a
  signal is heard and unaffected.
- **Fog is a weather lean**: `Band.weatherLean` multiplies over the
  Locality's lean wherever a scope is aboard a craft inside the band
  (`ExpanseApi.leanAtSync`, warm state only — cold leans nothing). See
  [weather.md](./weather.md). No band authors prose.

## ⭐ The planet is a setting (sugar D12, 2026-10)

The world's radius is ONE authored value: `_planetRadiusM` on the root
biome (`universe.yaml`, 2 124 000 m, a third of Earth's), read through
`BiomeApi.planetRadiusM()` (Earth's 6 371 000 when absent). Two things
derive from it:
- **The chart:** `GeoPosition`'s nautical miles per degree is
  `2πR / 360 / 1852`, 20 on this world, and exactly 60 when no radius
  is authored.
- **The horizon:** `Expanse.sightRangeNm` scales its coefficient by
  `√(R / R_earth)`, 0.675 of Earth's.

Every authored node focus is in METRES and never moves; the degrees it
comes to do. Gravity is authored separately and stays 9.81: the
heavy-world postulate (the-world slate). The water pack's sea-state and
power reads and the pump's work now read `BiomeApi.standardGravity()`
instead of a 9.81 literal.

## Deferred seams

| seam | attach point | slate |
|---|---|---|
| hazards firing at a boundary | `Band.hazards[]` + the boundary report | navigable-water § 3b |
| the berth | `AboardExit.computeDestination` | navigable-water § 7a |
| the second depth value | `GeoPosition.depthM` | underwater |
| the tide, a water biome, latitude as a resolved value | `WaterExpanse`, the `open-water` stub | climate-and-water |
| vector wind | `Band.lean` becomes a deviation | weather |
| a boat adrift across a restart | the dinghy is a deck prop, not boot-listed | the vessel build |
| cargo in a ship's hold | the rooms are singleton, non-persistable | the vessel build |
| the logbook (*what proves the depletion*) | `chronicle` is append-only and exists | navigable-water § salvaged Q1 |
| the place↔bulk line (a bath, a cistern, a spring) | `Shore` (beside) and `Pool` (in) are its two drawn edges | navigable-water § salvaged Q2 |
| reversing flow, the tidal bore | — | navigable-water § 4 / salvaged Q3 |

## History

- **Maritime review — lazy, not booted (2026-10-14).** Maritime R4 had
  booted every Structure and sea and enforced it (`lint:structures-
  booted`), and kept a `StructureCatalogue` — a class scan of every
  structure row — to answer which structure a room is in and what stands
  in a zone. Both broke the house rule: *everything loads when a player's
  act forces it, by following authored references, never by search*.
  Zones name their structures (`structures:`), what a place can see loads
  with the place, the sea names its own ships and lights, a ship's voyage
  is reconciled when she loads (one watch at load), and `journey` asks
  the seas the traveller can see instead of scanning every node row.
- **Maritime live browser walk (2026-10-14, `3978f8935`…`a9528fc07`).**
  The derived position stops at the target's radius and the beat records
  it on the first watch; `course` to the node you are at is refused;
  swimming is at the surface (respiration — a person breathes the air
  over the water); and a query set carries each match's own `via`, so
  `query all sea` on a deck names the sea, not the deck.
- **Maritime review — a pool IS the water (2026-10-13, `b6d9f5211`).**
  D27's `StillWater` (a fixed swimmable thing propped in the pool and on
  the strand) retired for `platform/location/Pool` and a strand that is
  a `Shore`; the locomotion gate reads an exit's far side. See
  [location.md](./location.md), [locomotion.md](./locomotion.md).
- **Maritime review — a Ship is a Structure (2026-10-13, `df5f24d0d`).**
  *"One field is the whole difference"* reversed: `Ship extends
  Structure` carries the voyage; a building is a plain `Structure`, and
  the light on a rock is where its rock's node is. Scoped reads replaced
  every registry-wide one (`d3daaa29a`).
- **Maritime review — the graph is spherical (2026-10-13, before
  merge).** The build had made lat/lon the ONLY system: every node
  authored an absolute `expansePosition`, nodes had no size
  (requirements: *"a node carries no radius"*), and a corridor restated
  its endpoint nodes' coordinates by hand, with nothing to keep them
  agreeing. That lost the planning ruling of 2026-10-10 — *the same
  spherical coordinates as locations, so you can put nodes anywhere;
  you plot in lat/lon, and one converts to the other*. Now nodes are
  `SphericalCoordinatesMixin` foci with a radius relative to the
  expanse, the expanse authors one `origin`, corridors are drawn from
  their endpoints, belts are authored relative too, and arrival is the
  radius. Every chart-side reader was unchanged.

- **Maritime revision 1 — perception at range (2026-10, before merge).**
  The room hook (`RoomContributorMixin`), the `Vantage` object, every
  `outsideDescription`, the beat's automatic sightings and fixes and its
  hazard warning were REMOVED: the engine was doing the lookout's and the
  navigator's jobs (lens 3b). In their place: the `horizon` scope, `look
  horizon`, `watch horizon` + the hired lookout's brain, `measure
  bearing`, the derived crossing delta, and fog as a weather lean.
- **Maritime revisions 2′ and 3 (2026-10-11/12).** `look` takes one
  thing and `query` (aliases `scan`, `inventory --ledger`'s rendering)
  lists a set (`810387483`); the shore as a place and drawing water
  (`a0d5e65cb`…`e9a83bdb4`) — see [location.md](./location.md) and
  [fishing.md](./fishing.md).
- **Maritime build (2026-10, MR !358, `0a393f154`…`b6d9f5211`).** The
  frame, the water tier, the voyage, the Structure, the two channels,
  the Greywater and the Hesper. The `SpatialZone` split into the region
  and `LocationZoneMixin`. The plan and requirements retired at the
  sweep; git history keeps both, with the drive record.
