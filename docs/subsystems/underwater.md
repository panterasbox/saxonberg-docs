# Underwater — water as a place you can be IN, and diving

Water as a **state of any place**, the **column** of rooms under a deep
water down to its bed, the crossings between them that cost real time,
what the water does as a **medium** (no flame, no voice, dim and blue
light, everything wet), **buoyancy** and the load in water, **drowning by
cost** and the rescue, the disconnected body **held at the surface**, the
`trade-diving` pack (the line, the bell, the hard-hat suit on a crewed
pump, the slate, the spear, the air bags, the harbor jobs), the **tide**
and what lives by the water, and magic below the surface.

This doc is the source of truth for:

- kernel — `lib/location/WaterPlace.ts` (`WaterPlaceMixin`),
  `lib/location/WaterColumn.ts` (the `WaterColumnMixin` seam),
  the water half of `lib/location/Shore.ts` (`wayIntoWater`, the depth in
  words, `onWaterRead`), `lib/biome/Medium.ts` (the medium vocabulary),
  `BiomeApi.mediumFor`, the in-water burden in
  `lib/encumbrance/LoadBearing.ts`, the water branch of
  `LocomotionLogic.engageHost` and `overYourHead`, `Exit.crossingRealS` /
  `Exit.wiresItsOwnInverse`, the timed crossing in
  `platform/idea/cmd/movement/LocomotionControllerBase.ts`,
  `DescendController`, the water branches of `SwimController` and
  `LieController`, `lib/spatial/{Pullable,Tossable}.ts`, `BuoyMixin`;
- the water pack — `packages/content/water/src/location/Band.ts`,
  `idea/ColumnWarren.ts`, `idea/exits/ColumnExit.ts`, `lib/AirPocket.ts`,
  `lib/Tide.ts`, the tide halves of `WaterExpanse` and
  `WatercourseCatalogue`, and the band rows under
  `content/system/water/location/band/`;
- `packages/content/trade-diving/` entire.

The breath itself — the store, the urge, cold shock, the ears, the bends,
the tread — is [respiration.md](./respiration.md)'s; this doc says what
the water asks of it.

**Read first:** [respiration.md](./respiration.md) (the body under the
water), [watershed.md](./watershed.md) (the reach and the read a water
place cites), [expanse.md](./expanse.md) (the sea),
[fishing.md](./fishing.md) (the fishery a spear draws from, `pull` on a
pot), [location.md](./location.md) (the Warren the column is),
[ground.md](./ground.md) (`noDefaultFloor` and the bed's floor),
[spatial.md](./spatial.md) (the bell's containment opening),
[exertion.md](./exertion.md) (the tread is exertion). The design:
`docs/slates/builds/underwater-slate.md`; the river after it:
`docs/slates/builds/river-slate.md`.

---

## ⭐⭐ Water is a state, not a kind of room

A place says **which water it is and how deep its floor lies below that
water's level**, and everything else is derived:

```yaml
water: { of: kestrel:confluence, depthM: 6 }
```

`of` is a water citation exactly as `waters:` values are (a reach ref or a
sea's path); `depthM` is the floor below the water's datum — negative is
normally dry (the tidepool's `−0.5`). Optional: `feature` (the `waters`
key the place's own water registers under — Hollow Mere keeps `mere`, so
`fill bucket from mere` still binds; default `water`), and `rimM` (a
basin's rim; below it the basin keeps its own water — see the tide).

`setWater` registers the water in the place's own `waters`, so `fill`,
`drink`, `wash` and `look water` find it with no `details:` feature: **the
place IS the feature.** The host composes `WaterPlaceMixin` over
`ShoreMixin` (the source, the memoised read) and `SwimmableMixin` (the
enablement). `Pool` survives as the kernel instanceable twin; the
retired kinds are gone (`PoolBottom`, the cistern's `_atmosphere: water`).

Derived on read, nothing stored:

| read | what |
|---|---|
| `levelNow()` | the water's level above its datum, m — the read's `levelM`, 0 when unknowable (the tide moves it) |
| `depthNow()` | `depthM + levelNow()`, floored at `rimM` when the basin is cut off |
| `stateNow()` | `dry` (≤ 0) · `wading` (≤ `water.column.wadeM`, 1.5) · `swimming` · `frozen` (the read says frozen) |
| `bandsNow()` | the column under it now (below) |

An uncited `Pool` reads `swimming` (legacy deep water). ⚠ The wading depth
is one dial for every body; the plan's per-stature factor was not built.

⭐ **Who is in the water is per occupant, not per room.** The place's
floor (the water's surface `WaterSurface`, or an ice sheet) either holds
an occupant up or it does not, so a body on the ice and a body in the
hole beside it share a room and not a medium. `WaterPlace.mediumFor(occupant)`
answers `air` for a body the floor supports and otherwise
`{tag: 'water', immersed: true, atSurface: true, temperatureK,
oxygenMgL}` — in the surface place your head is out. `BiomeApi.mediumFor(stuff)`
is the one medium read: it walks out from `stuff` to a liquid-holding
vessel, a holder with its own air (a bell), or a water place, and the
async atmosphere tag derives from it so the two cannot disagree.

**Frozen.** A water whose read is frozen is not swum (`canEngageAxis`);
the floor decides who it bears. The cited water's frozen state is asked
only AT the surface — a band under the ice is still water to swim in.

**The ground.** A place with no floor answers the posture verbs through
the kernel `Place.groundRefusal(named?)` / `noGroundLine()` hook: `sit`,
`lie`, `kneel` and bare `stand` ask it before *"you don't see any…"*. The
lit and dark bands author `noDefaultFloor: true`, and `Band` overrides the
line: *"There is nothing under you but water."* The bed authors no floor,
so it mints the default ground and is on grade.

**Walking in.** `LocomotionLogic.overYourHead`: a ground mode into a water
place whose `stateNow()` is `swimming` is refused — *"You would be in over
your head there; swim, or go down to it."*

## ⭐⭐ The column — bands budded under a deep water

A water deeper than you can stand has a column of rooms under it.
Being together is sharing a room, so depth is rooms, never a hidden
number every channel would have to filter.

**`bandsNow()`** from the depth `d`, the wading depth and the water's
clarity (`clarityM` on the read: the depth at which 1 % of surface light
is left — a `Watercourse` or `WaterExpanse` row may author it; default 5):

- `[]` when `d ≤ wade` (you stand);
- `lit` when `d > wade + water.column.minBandM` (2);
- `dark` when `d > clarity + minBand`, then ⭐ **the dark repeats**:
  `dark-2`, `dark-3`… about one per `water.column.darkBandM` (10 m, an
  atmosphere) of the water between the light's end and the bed, capped at
  `water.column.maxDarkBands` (5). Past the cap the bands thicken — a
  1,000 m trench is still seven rooms, with long swims between them;
- always `bed`.

The basin below the Wharfside quay (6 m, the Kestrel) is
lit · dark · bed; Hollow Mere (5 m, clarity 8) lit · bed; the cistern
(3 m) bed only. `bandDepthM(band)` is the middle of each band (lit:
halfway between the wading depth and the light's end; each dark band an
equal share of the dark; the bed at `d`) — the depth the breath, the ears
and the gas read.

**The seam.** The kernel knows a water is deep; it does not know how a
column is built. `WaterColumnMixin` is the capability (`ensureColumnExits`,
`bandOf`), the water pack's `ColumnWarren` singleton
(`/system/water/idea/ColumnWarren`, `TemplatePaths.waterColumn`) implements
it, and `WaterPlace` finds it by path and narrows by shape — so `sink`,
`takeUnder`, `holdAtSurface` and the way down live on the kernel mixin and
never import the pack.

**`ColumnWarren`** = `WaterColumnMixin(SingletonMixin(PersistableMixin(RoomWarren)))`,
one for the realm (a column needs no per-venue policy; the band rows are
the system's, so a second water needs zero rows):

- **Budded on first traversal**, keyed `<surface>#<band>` (the surface's
  identity path), serialized, `restoreOrSeed` from its snapshot, wired as
  it buds. Every dark band clones the one `dark` row. Coordinates are the
  surface's, z decreasing.
- **Wiring.** Each band gets `up` (the band above, or the surface) and
  `down` (the band below) as `ColumnExit`s; the surface gets its `down`
  from `ensureColumnExits`. ⭐ That runs in `WaterPlace.onContainableAdded`
  for any agent — walked in, swum in, or **restored there after a
  restart** (load on use: the use is being here).
- **Neighbors.** A surface row may author rooms beside one band:
  `column: { neighbors: [{ band, direction, destination, opposite? }] }`,
  wired both ways (exit kind `/system/water/idea/exits/underwater`) when
  the band buds. The intake mouth beside the basin's dark band is the one
  shipped — *the bespoke case is authored rooms beside the column*.
- **Keeping.** A band that empties of people is captured to its snapshot
  (what sank stays sunk across a restart); one its surface no longer has
  (the water went down) is captured and reaped.
- ⚠ **Sideways** between two neighboring waters' columns was not built —
  nothing that ships has two water places side by side (river-slate).

**`Band`** (`/system/water/location/Band`) =
`WaterPlaceMixin(SwimmableMixin(ShoreMixin(PersistableMixin(WarrenMemberMixin(Location)))))`,
stamped at bud with `surface` and `band` and its surface's water. It
authors nothing of its own water — every read is the surface's. Its
medium is always **immersed, never at the surface, at the band's depth —
standing on the bed too**, with `visibilityM` (the clarity) and the
surface's oxygen. `stateNow()` is always `swimming`; `bandsNow()` is `[]`.

**`ColumnExit`** (a `DeferredDestinationExit`, `media: [water]`):

- ⭐ **`targetNow()`** — the way resolves against the column **as it is
  now** on every crossing (up = the band above, down = the band below).
  A way fixed at bud skipped a band that exists now (a bed budded before
  the river's read landed went straight to the lit band).
- ⭐ **`wiresItsOwnInverse()` → true.** Every column's band shares one
  row, so the by-path exit-topology check found another water's band and
  blocked dark → bed. An exit that wires both ends says so on the kernel
  `Exit` hook and is never verified by path.
- **Under ice the only way up is the hole**: a way up into a surface whose
  floor is ice with no open water refuses — *"The ice is a ceiling over
  you; the only way up is the hole."*
- **Weight refuses the way up** (`WaterPlace.canRise`, below); *"There is
  no deeper water that way now"* when the tide took the band away. All
  three are checked before the mode gate.

## ⭐⭐ Crossings take real seconds

Depth is space you have to cross to get to air. The kernel `Exit` hook
**`crossingRealS(mover)`** is 0 for a step; `ColumnExit` answers *the
meters between the two depths over the body's vertical swim speed*
(`Respiration.verticalSwimSeconds`: untrained 0.4 … expert 1.2 m/s by
swimming band, ±0.5 m/s per 100 N of net weight or lift — ballast
speeds the way down and slows the way up — clamped to
`[0.15, 2.5 × base]`).

The movement base's one crossing asks it. A timed crossing:

1. pre-checks the **whole** traversal gate (`LocomotionApi.canTraverseExit`
   — body, posture, load, exit) so a way that would refuse at the end
   refuses now, before the breath is spent;
2. runs a `ManualBuildStep` on the **hands** slot for that many real
   seconds (scaled to game time by the world rate) and says *"You swim
   down."*;
3. lands in `arriveAfter`: nothing if the mover blacked out on the way;
   otherwise `actor.traverse` again (the world moved while you swam), its
   refusal said in the crossing's words. `stop` leaves you where you were.
   A completion runs outside any command, so it catches and logs — an
   unhandled rejection there takes the server down.

So **`descend`, `dive` (in the water) and `swim down` are one act**, and
none skips the time. `WaterPlace.wayDown(actor)` / `wayUp()` pick the way
and refuse in words (*"There is no open water here to go down into"* off
the ice; *"The water is no deeper here"*; *"You are already at the
surface"* — the basin's own `up` is the quay wall). Going into the water
from a shore is instant.

**The verbs.** `descend` and `ascend` [`surface`] are afforded by
`WaterPlaceMixin` (`inventory` bucket) — only a body IN a water has a
down; a dry quay has none. Beside a water, **`swim`** [`jump`, `dive`] is
the way in: with no way named, `Shore.wayIntoWater()` picks the exit into
the water (and owns the *"iced over"* refusal); in the water `dive` is
down. Out of the water is `climb` by something climbable — the quay steps
and the cistern's rungs are `Ladder` props; the wall beside them refuses
(`WaterPlace.canExit` admits swimming and climbing). A shore's water
reading says how deep it is off the edge, in words, never a number
(*just over your head · deep · very deep · wade · mud*).

A column exit's authored `messageOut`/`messageIn` is what OTHERS see; the
mover reads the ordinary second-person line.

## The medium — what water does to everything in it

`lib/biome/Medium.ts` is a **kernel vocabulary**, not a row kind: `MEDIA`
keyed by tag (`air`, `water`, `vacuum`, `smoke`, `blackdamp`, `stinkdamp`,
`carbon-dioxide`), one record of columns each seam reads
(`densityKgM3`, `thermalConductivity`, `breathable`, `contaminant`,
`supportsCombustion`, `carriesVoice`, `soundHopTau`,
`soundCarriesDirection`, `lightTransmittancePerM`, `drag`). Every reader is
synchronous on a hot path and in unit suites with no content store, where
a catalogue reads null cold — so the medium is physics, like the
materials `Channel` vocabulary; a new medium is one entry.
`BiomeApi.densityOf / conductivityOf / breathableOf / contaminantOf` read
it; `BiomeApi.mediumProps(tag)` is the record.

| what | under the water |
|---|---|
| **voice** | `Vocal.canVocalize()` reads the body plan's `voice.media` (default `['air']`) against `mediumFor`: `say`/`whisper`/`shout` come out as *"a gout of bubbles, and nobody hears them"*. The controllers render the returned `VocalResult`. `cast` asks it too (*"You cannot speak the words here."*). The implant (`dm`, emotes) is the aether and works as anywhere. |
| **fire** | `FireLogic` ignition refuses where the medium does not burn (`no-air`); `Burner.completeness()` is 0 under water; the water place **douses** whatever arrives in it and everything it carries. |
| **sound** | a band's hop is the medium's `soundHopTau` (0.3, further than air) and arrives from no direction — `ring` a signal bell and every diver in the column hears it, placing it nowhere. |
| **light** | `Band.getAmbientFlux()` is the surface's light now × `0.01^(depth / clarity)`; a night dive is black without your own light. Red goes first: carried as the band's color temperature (`6500 + 1500 × depth` K, capped). ⚠ A per-channel fold through the vision walk was not built. |
| **wet** | everything arriving immersed is `wet(1)` (an `Attired` body `immerse()`s), recursively through what it carries — except inside an air pocket. |
| **vessels** | an open, empty vessel arriving fills from the place's water. |
| **eating** | `Metabolic.canIngest()` refuses an immersed air-breather (*"You would take in water"*). |
| **wash** | works in the water you are in, and says so (*"in the water round you"*). |
| **posture** | mid-water has no ground (above). Floating is `lie` (or the word `float`) on a water surface that does not bear you — `Respiration.float()` decides, refusing a non-swimmer in words. |
| **sight in a fight** | `MediumRead.visibilityM` is how the combat engine reads the range cap. |
| **weapons** | `CombatLogic` passes the medium's `drag`: swung forms are slowed hard, thrusts much less. A `Launcher` refuses unless its row says `worksUnderwater`. |
| **senses** | `BodyPlan.portMedia(port)` / `getModalitiesIn(medium)` — a sense organ works only in its media (smell and taste are air-only by default; a shark's row says water). |

Each refusal is in the words of the thing that failed.

## ⭐ Buoyancy, weight and load in the water

`LoadBearing.getDisplacedVolumeM3(depthM)` sums the body (mass over its
body density), **the air in its lungs squeezed by the water over it**
(`getLungAirL() / (1 + depth/10)` — exhale and you sink, inhale and you
float), and every carried and worn thing by its material.
`getNetBuoyancyN(ρ, depth) = g(ρV − m)`: positive floats.

⭐ **The burden in the water.** `getBorneBurden(ρ)` / `getLoadRatio(ρ)`
take an optional water density: each carried thing counts at
`mass × (1 − ρ_water/ρ_thing)` — lead still pulls, a bag of air counts
negative and helps carry the rest, the total clamps at 0. The encumbrance
veto passes the water's density for a water mode.

⭐⭐ **Weight refuses only the way UP — never going in or going down.**
`LocomotionLogic.engageHost` lets every water-mode direction but `up`
through the load veto, because weight is what takes you down and what it
takes from you is rising (a diver in a belt with a stone was refused
`descend`; a suited diver could not get in at all). The way up is refused
twice, by the same physics: the veto, and the column's own
**`WaterPlace.canRise(body)`** — *"You are too heavy to swim up — drop
something."* when net lift plus a swimmer's 40 N kick is below zero.
Never a skill refusal. `drop belt` (or the ingot) always works:
`DropController` already releases worn gear through its slots. A body
somebody has grappled cannot rise (*"You are held"*) — holding someone
under is the breath model, not a damage roll.

**What goes where.** `WaterPlace.sinks(thing)`: a still, tangible,
unfixed thing denser than water — unless a **`BuoyMixin`** thing in the
same water `buoys()` it (an air bag lashed to it). In open water (no floor
that bears even an empty load, or no floor at all) a sinking arrival goes
to the bed (`sink`, deferred a tick); on ice it waits for the thaw
(`sinkWhatSinks`). In a band, a floater goes up to the surface place and a
neutral body stays. A water you can stand in has no column and keeps the
thing on its own floor.

## ⭐⭐ Drowning by cost, never a check

Swimming is a cost. `Swimmable.difficulty` is gone in effect
(`setDifficulty(non-null)` throws *"a cost, not a check"*); a body swims
if its plan lists `swim`. At the surface, unsupported, every respiration
beat is a **tread** (`RespirationTread`, a body-slot engagement): `exert`
at 150 W × technique (untrained 3.0 … expert 0.8) × buoyancy (0.5 when
you float; up to 4 when you are heavy) × (1 + the drowning bodies clinging
to you). The swimming Discipline is credited per beat.

- **Free:** floating (`lie` on the water, at least novice and not too
  heavy), holding something that floats you (net lift ≥ 30 N, scaled by
  body size — a frog floats on a small lift), being held up on a line,
  wading.
- **Spent** (`breathBand() === 'spent'`): the **drowning response**. The
  swimmer is told — *"You can't keep your head up…"* — and those around
  see *"upright in the water, head tipped back… not calling, not
  waving"*, **once per response**, not on every beat a tired swimmer dips
  back into it. The voice is refused (*"You cannot get the breath to call
  out"*). The body climbs the nearest other swimmer at the surface
  (`beClungTo`: the rescuer treads for two — *reach, throw, don't go* is
  derived, not taught).
- **After 60 s** spent, `WaterPlace.takeUnder(body)` moves it into the
  first band, silently to others; the swimmer's own line is *"The water
  closes over your head."* Then the breath model takes over.

## ⭐ The disconnected body is held at the surface

A player who drops connection in the water is floated up and **held
face-up at the surface** until they come back: `Avatar.onLinkdead()` →
`WaterPlace.holdAtSurface(body)` (moved to the surface place,
`Respiration.holdUp()` — breathing, no tread). The hold is a breathing
state on respiration, not an Avatar field. `Avatar.enter()` releases it.
The dying clock, if it was running, runs on. ⭐ A flood does the same:
`WaterPlace.reconcileOccupants()` (on entry, and on every fresh read of
the water via the `Shore.onWaterRead` hook) holds every linkdead body in
a water whose state became `swimming` and tells the place *"The water is
coming in."* A logged-out body is not in the world to hold. Animals and
NPCs keep the physics. The general disconnection rule is
physiology-slate's.

## Rescue

- **A line.** `throw <line> to <person>` puts the free end in their hands
  and holds a body in the water up while it is held. `pull line` brings
  whoever is on the end up **one band per pull** — hand over hand
  (0.6 m/s, slower with a heavy load, plus 5 s for the steps at the top),
  as a timed step on the puller's hands — and the last pull drags them up
  the steps and out, conscious or not. A pull that brings them up a band
  says there is more line still in the water. A person on the surface is
  held up. Carrying a person is slated
  (`docs/slates/builds/carrying-a-person-slate.md`); the water's rescue is
  the line.
- ⭐⭐ **Air rescues a breath death.** Back in air with blood oxygen above
  the floor, a body whose dying cause is the breath's own (asphyxiation,
  strangulation — `Vitals.getDyingCause()`) is **stabilized** — rescued,
  not healed: the `aspiration` condition follows (cough → fever → drowning
  in bed, by game-hour stages, cured by medicine). See respiration.md.
- **A death is told.** An environmental death says *"<you> goes still"*
  to the room (`ConditionLogic.divideBody`), and the shade's first line
  says you are dead, where your body lies, and that `passage` takes you
  back. `passage` refuses a shade in the water (*"Get out first"*) — a
  drowned diver re-embodied on the bed drowned again — and sets each
  reserve to a level (40 %) rather than deducting, relieving the collapse,
  starvation and dehydration those levels clear.

## ⭐⭐ The diving trade — `/trade/diving`

The `trade-diving` pack (root `/trade/diving`): practised and quittable;
it owns no water. A second dive shop is rows.

**Verb protocols are protocol mixins** (review): what a verb narrows its
target on is a thin mixin with a refusing default each host overrides,
never a bare interface — it can be `requires:`'d, has `MixinApi.isX`, and
shows in Studio. Kernel `PullableMixin` (`pullBy(puller, sense)` →
`PullOutcome`, which may carry `takes: { realS, finish }` for a timed
pull) and `TossableMixin`; kernel `BuoyMixin`; the fishing pack's
`TakesFishMixin`. See [mixins.md](./mixins.md).

**One verb per act** (each a review decision):

| act | verb | not |
|---|---|---|
| bring a line, a pot or the bell in | `pull` (no `haul` alias — haul, hauler, haulage are logistics' words, and the kernel `HaulableMixin` is a cart you hitch) | `haul` |
| toss a line's end to somebody | `throw <thing> to <person>` (over `Tossable`; anything else is told to throw it AT someone or hand it over) | `heave` (now one of `throw`'s words) |
| take a fish with a spear | `fish`, with the spear in hand or `fish using spear` (the spear is `TakesFish`; a `Rod` casts) | `spear` |
| write on a slate | the kernel `write <words> on <thing>` over `MarkedMixin.writeOn` (`writingRoom` on the row; held or fixed; appended) | `jot` — and the shell's authoring verb became **`edit`** |
| float | `lie` on the water | `float` as its own verb |
| clear your ears | nothing — passive, like breathing | `equalize` |

Trade verbs: `tug` (afforded by the line's end to whoever holds it),
`ring` (a signal bell), `lash` (a line's end or an air bag to a load).
Reused: `wear`/`remove`, `pump`, `drop`, `get`, `read`, `buy`, `consign`,
`appoint`/`apply`/`clock`, `job`/`fulfill`, `analyze water`, `cast`/`zap`,
`climb`.

**The gear** (`content/trade/diving/thing/`):

- **Diving stone** and **weight belt** (a worn garment) — ballast down.
- **Lifeline** (`Lifeline`, `kind: line`, 20 m) — the fixed end is in the
  holder's hands; making it fast mints a **`LineEnd`** into the hands of
  the person on it (holding it is being on the line, dropping it is
  letting go). Who is on a line is runtime — a line does not outlive a
  dive. A line knows people **as its holder does** (perceived keywords,
  `describeFor`), never by a true name nobody told them. A line brings at
  most 80 kg (in water, less air lift) off the bottom; heavier *"wants air
  under it."*
- **`tug`** — signals up the line to whoever holds the other end, any
  medium: one tug *all is well*, two *haul me up*, three *more line*
  (tugs within 20 **real** seconds are one signal — at 20 game-seconds two tugs had to land within 1.7 real seconds). The end of a hose answers
  to `hose`.
- **Signal bell** (`ring`) — a hearing event at 95 dB from the striker;
  in water every diver hears it, from nowhere they can place.
- **Slate** — `MarkedMixin(Good)`, `writingRoom: 240`; scratched, not
  inked, so the water does not wash it out.
- **Fish spear** — a `Weapon` (pointed: the form that still hurts below)
  and `TakesFish`: under the surface it **draws from the reach's fishery**
  (the most plentiful species above 15 % of capacity) and mints the
  fishing pack's fish; a fished-out water has nothing to spear. The
  fishmonger stays a consignment shelf: the sale is `consign`.
- **Air bag** and **air barrel** (`AirBag` = `BuoyMixin(BulkableMixin(Good))`)
  — `lash` to a load: lift is `liters × 200 / (1 + d/10) − bag mass`
  per bag (a vessel's liters are compressed ×200); when the bags lift more
  than the load's mass, the pair goes up to the surface together. While
  lashed, the bag `buoys` the load, so the surface's "a dense arrival
  sinks" rule leaves it. The lost anchor (300 kg) wants three at the bed.
  Carried into a bell and opened, its air freshens the pocket.
- **Tide almanac** (`TideAlmanac`, `MarkedMixin`) — its page is the sea's
  own tide computed when read: the next twelve turns, by the calendar.
- **Dive tables** (a `Chart`) — the stops by band and bottom time; the
  stop is dwelling in a band.

**The bell.** `Bell` = `AirPocketMixin(ExitableVessel)` (the water pack's
`lib/AirPocket`): an authored `airM3` of air at the surface, squeezed by
Boyle to `airM3 / (1 + d/10)` with the water rising inside
(`waterlineFraction`); every breather fouls it with CO₂ at a resting rate
and a lamp fouls it more, on the pocket's own 30-second clock while
anyone is inside; the shipped displacement floor is what finally stops
them breathing. The **`Hoist`** (`PullableMixin(Tool)`, on the quay) moves
its bell one band per `pull bell down|up`, whoever is inside riding with
it. ⭐ **The containment opening:** exitables may not cross zones by
containment, and a water's bands are the column's zone while the bell is
its quay's — so a vessel may enter a water band **whose surface place is
in the vessel's own zone** (spatial.md). The stamp stays honest. ⚠ Bell
occupants do not load gas (no bends from a bell).

**The hard-hat suit.** The **helmet** is an `AirTank` row worn on the head
— the suit and the shipped air tank share one supply path; the **dress**
is a two-slot garment (listed once in what you wear); the lead **boots**
(36 kg) make you negative, so you walk the bed and cannot swim up. The
**hose** is a `Lifeline` with `kind: hose` (25 m): its free end is the
helmet's end, and the same `pull` brings the diver up. The **`AirPump`**
on the quay speaks the kernel `Pumpable` protocol, so the verb is `pump`:
each spell (20 s at the handles) fills the helmet on the end of a hose held
there. The helmet holds **2 units** — about a minute at the surface, forty
seconds at the basin's bed, at the tank tap — so the diver lives on the
pump: pump stops → the helmet drains (faster at depth) → the supply crisis
→ two tugs → the tender pulls. Gas loads only
from a worn supply at depth: the **bends** on surfacing,
**recompression** by going back down (respiration.md).

**The crew** (Morrow's dive shop at Wharfside, `call: rota`): the
`divemaster` sells and hires; the **tender** (`tends` brain) holds a line,
pulls on two tugs, and pulls a diver who has gone limp after 20 real
seconds of silence; the **pump hand** (`cranks` brain) works the pump whenever a
diver is on its hose. Both use the verbs a player types and are not
presence-gated — a diver breathes whether or not anyone watches the quay.
The counter stocks the stone, belt, line, gear, tables, almanac and a
wand of glowlight.

**The harbor jobs.** The Wharfside pilot holds the harbor-works seat and
its **`surveys`** brain keeps two jobs on the board with the same `job post`
a player types: *deliver the lost anchor to the bank* while it lies in a
water place, and *recover a body* — `job post recover <identity> to
<bank>`, a delivery of THAT instance by its own identity (a corpse has no
chattel). It reads the basin's bed only when that band is resident —
load on use, never a search. ⛔ There is no inspection job: it is a
contract-and-report interaction, not water's, and needs its own design.

## The tide and what lives by the water

**`Tide`** (`/system/water/lib/Tide`, a value object) — semidiurnal, two
highs and two lows a lunar day, the caller handing it the moon's phase:
each high water ~50 minutes later than the day before; springs at new and
full moon, neaps at the quarters (±30 %); `streamFactor` peaks at mid-tide
and is 0 at the turn. A `WaterExpanse` authors `tidal: { rangeM, lagH }`
(the Greywater 3 m); a river's sea-level reaches take the realm's dials
(`water.tide.rangeM` 3, `water.tide.lagH`), and the stream is
`streamFactor × water.tide.peakStreamMps` (1.2). ⭐ A tidal read is
**re-tided every 10 game-minutes** between full reads (`WaterSource.retide`,
the `Shore` memo) and fires `onWaterRead` — no timer anywhere. The shore
reads the tide in words on `look`; `analyze water` gives the figures.

**The estuary** — the channel (the brine source the mouth cites, 3 m),
the **tidepool** (`depthM −0.5`, `rimM 0.4`): below the rim it is cut off
(`isCutOff`), keeps its own water, and its oxygen falls from the moment it
is first SEEN cut off by `water.tidepool.o2DecayPerHour` × a Q10 of 2 —
and the **strandline** (`depthM −1.3`, `tide: { collectsFrom: [the
channel] }`): on entry and on each fresh read it takes every floating thing
that has lain in a collecting water since before the last high water.
Arrival times are runtime — a restart forgets which tide a plank came in
on, and it waits one more.

**Life by the water** — breathing is a **rate**: `BodyPlan.breathing:
[{medium, apparatus, intake}]` with per-stage overrides
(`Species.breathingByStage`), `getBreathableMedia()` derived (intake ≥ 1);
gills in water take the water's oxygen against the stage's habitat
(`habitatByStage`), so a trout in foul water suffocates; the bullfrog
dives and surfaces on its own breath (the kernel `surfaces` brain);
dissolved oxygen is derived on the read. Detail: respiration.md and
fishing.md.

## Magic below the surface

The two held workings ride the shipped `SustainedEffect` arm:
**`breathe-water`** (`realizes: confer-medium` → `Respiration.setConferredBreathing`,
900 s) and **`keep-warm`** (`offset-ambient` → `ThermalRegulation.setWorkingOffsetK`,
+12 K). Each pays a **standing bill** from the caster's mana at the charged
shell's standby draw × attention; at zero the working lapses and you have
the breath you had to get up on. A spell needs a voice — `cast` refuses
under water — so you cast on the quay; a wand works below (the dive shop
stocks the wand of glowlight). Breathe-water loads no gas.

## Known gaps (from the final browser walk)

- **(a) Entering the lowered bell under water** (`swim bell`) was refused
  in the walk and is unverified.
- **(b) Owned goods duplicate** when their owner re-materializes within
  one server uptime: estate entries keep place `inventory` for goods moved
  into transient places (corpses, the riverbed, the quay). A pre-existing
  chattel/estate defect, not water's ([furnishing.md](./furnishing.md),
  [chattel.md](./chattel.md)).
- **(c) Step-7 burst eardrums cannot occur in the world.** Every diveable
  water ships ≤ 6 m, and passive clearing (`EARS_CLEAR_MPS` 0.5, plus the
  clearing that accrues during the timed crossing itself) outpaces any
  descent there.
- **(d) Hyperventilation**: three quick inhales still give one warning;
  four silence it.
- **(e) Not walked:** the winter ice hole; dropped things sinking and
  floating; night and the glowlight; salt water; fights below; the bends
  (≈ 8 real minutes on the 6 m bed); the body-recovery job; a barrel on
  the strandline; an offline body on the low strand.
- **(f) Carrying a person** is slated
  (`docs/slates/builds/carrying-a-person-slate.md`).

- **(g) Requirements not met:** **narcosis** (a goal of the build; no row
  and no check exist); the **bends' onset** — they strike on surfacing,
  where the plan said twenty minutes after (the condition then advances in
  20-game-minute stages); **wading by stature** — `water.column.wadeM` is
  one depth for every body.

Also recorded, not built: sideways between neighboring columns; a
per-channel color fold for light under water; bell-occupant gas loading;
narcosis; *only `get` works in the drowning response*; an accountability
event for the drowning grab; thrown range capped in water; `disarm`
widened to worn gear (pulling off a helmet); the swim's effort raising the
breath burn (respiration.md).

## Files

- kernel: `lib/location/{WaterPlace,WaterColumn,Shore,WaterSource}.ts`,
  `lib/biome/Medium.ts`, `lib/encumbrance/LoadBearing.ts`,
  `lib/spatial/{Pullable,Tossable}.ts`, `lib/stuff/Place.ts`
  (`groundRefusal`), `lib/boundary/Exit.ts` (`crossingRealS`,
  `wiresItsOwnInverse`), `platform/idea/api/LocomotionLogic.ts`,
  `platform/idea/cmd/movement/{LocomotionControllerBase,DescendController,SwimController}.ts`,
  `platform/idea/cmd/posture/LieController.ts`,
  `platform/idea/cmd/inventory/{PullController,WriteOnController}.ts`;
  views `platform/cmd/movement/{descend,ascend,swim}.yaml`.
- water pack: `src/location/Band.ts`, `src/idea/ColumnWarren.ts`,
  `src/idea/exits/ColumnExit.ts`, `src/lib/{AirPocket,Tide}.ts`; rows
  `content/system/water/location/band/{lit,dark,bed}.yaml`,
  `content/system/water/idea/exits/{column,underwater}.yaml`.
- trade-diving: `src/thing/{Lifeline,LineEnd,Bell,Hoist,AirPump,AirBag,SignalBell,Slate,Spear,TideAlmanac}.ts`,
  `src/behavior/{tends,cranks,surveys}.ts`,
  `src/idea/cmd/diving/{Tug,Ring,Lash}Controller.ts`, rows under
  `content/trade/diving/`.
- venues: terminus `wharfside/water/` (basin, intake mouth, quay steps,
  bell and hoist, heaving line, lost anchor), `wharfside/diving/` (the
  shop, its outfit and crew), `estuary/{channel,tidepool,strandline}.yaml`;
  eternal-university's cistern; hollow-mere's pool.

## History

Built as `reqs/underwater`, MR !408 (2026-10-18 – 10-21): the plan
(decisions D1–D30, review rounds R1–R13) and the requirements were retired
at the sweep and their standing decisions live here and in
respiration.md. The build's commits are `origin/master...reqs/underwater`
at its merge.

## River build (2026-10-21)

A **moving** water place: a river's reach patch is a water place carried
along the river's line, its bed a fixed stop below it (no column under a
river — the Kestrel is too shallow for bands); the current and roughness ride
the medium read; the **sea patch** (`SeaPatch`) is the water beside a ship
stopped at sea. `WaterPlace.headOutAt` is the hook something closing over the
water (ice) answers. See [river.md](./river.md).
