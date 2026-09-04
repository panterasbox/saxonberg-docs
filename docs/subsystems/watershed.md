# Watershed

Built from the water requirements (D1–D27). The kernel takes the
physics; the `water` capability pack
takes the works.

The subsystem that makes water **get somewhere**. Water already had
physics everywhere and weather nowhere: it is bulk matter you can fill,
pour and drink, the body has a `hydration` reserve, soil holds moisture
in litres. What was missing was every connection between them.

One relation supplies all of it: **every place has a position on a
watershed.** Flow, rights, diversion, storage, pollution and hydro power
are readings of that single fact, and the organising primitive
underneath is **gravity** — `mgh`.
## ⭐ `SupplyState`'s second speaker

The six-word vocabulary (`dry` · `cut` · `frozen` · `fouled` · `off` ·
`overdrawn`) is in the KERNEL rather than in this pack, deliberately, and
the TPA reform is the reason that was right: **mana** is now the second
utility that fails in it. A Teleport Authority gate reports `dry` when
its cell is spent, `cut` when its line is severed, `off` when somebody
closed the stopcock, and `overdrawn` when the ride costs more than it can
cover.

Two packs that must agree on the same six strings are an entangled
namespace, which is one of the two things that genuinely belong in the
kernel — and had the vocabulary been owned by whichever utility shipped
first, arcana would now depend on the water pack for a word.

⚠ **Only the VOCABULARY is shared.** `ManaPoweredMixin` deliberately
does NOT implement `SupplyReporting`: `AnalyzeWaterController` reads that
interface structurally, so implementing it would make `analyze water
<terminal>` work by accident. The `supplyReport` shape stays water's.

See [fasttravel.md](./fasttravel.md) and
[magic-items.md](./magic-items.md).


## The spine

One edge, repeated at two scales: **precipitation integrates into a
place.** Once for soil (a bed fills from the sky) and once for a
watershed (a reach fills from its catchment).

## The precipitation integral (W1)

`WeatherApi.precipitationBetween(t0, t1, locality)` sums precipitation
over a half-open window and returns **two** figures plus a coverage
report:

| | |
|---|---|
| `liquid` | mm of water that reached the ground over the window |
| `frozen` | mm water-equivalent that fell as snow — banked, not run off |
| `coveredS` | game-seconds actually walked (`< t1 − t0` iff the cap bit) |

It is **exact**, not sampled. `weatherAt` is a pure function of
`(time, locality)` — no tick, no stored weather — so every six-hour
segment between two instants is computable now, and each contributes its
own rate times its own overlap with the window. Integrating the same
window twice gives the same answer; splitting it anywhere and summing
gives the same answer as integrating it whole. That is what lets a place
integrate an absence of arbitrary length on its first read back.

**One walk, two consumers** (plan § P2). Soil multiplies `liquid` by a
bed's land area for litres into the ground (W1); a watershed multiplies
it by a catchment's area for inflow to a reach (W4). Designing it around
one and generalising later is the trap the requirements exist to avoid,
so the walk knows about neither.

### The rate table has to be authored

`WeatherTypeProfile.precipitation` is a **descriptor** — `none` / `rain`
/ `snow`. It says *what* falls, never *how much*, and there is no honest
way to derive one from the other. So
`PRECIPITATION_RATES_MM_PER_HOUR` is the build's one new number, keyed
on the weather **type** rather than the descriptor — a storm and a
shower share the descriptor `rain` and differ by a factor of four.

The figures are **six-hour segment averages**, not instantaneous
intensities: our weather is piecewise-constant, so a `rain` segment
rains for six hours straight. Real moderate rain falls at ~2.5 mm/h but
never for six hours together, so the authored values are the averages
that reproduce a plausible annual total under the shipped transition
grammar. `Precipitation.test.ts` asserts that total as a **climate**
claim, so a dial change that would make the realm a desert or a
rainforest fails rather than passing quietly. Operators override through
`water.rate.{rain,storm,snow}`; the table is the fallback, so the kernel
rains correctly with the `water` pack absent.

### The cap

`WEATHER_DEFAULTS.PRECIPITATION_MAX_SEGMENTS` (120 segments = thirty
game-days) bounds one walk. It is part of the integral rather than a
later optimisation: "arbitrary absence" includes a place nobody has
touched since the epoch, and uncapped, the first read after a long
dormancy replays every segment since `t = 0`. The cap keeps the **tail**
of the window — you come back to recent weather, not ancient weather —
and thirty days is far longer than any bounded sink in the build takes
to saturate, so it costs no observable water.

### What it does not see

A **scope-tier** weather pin (one room overriding within an otherwise
modelled locality) is invisible to the integral, which is keyed by
locality. A **locality**-tier pin is honoured — every segment in the
walk is the pinned type, so a weeping valley rains through the year and
a pinned drought never rains at all. This is a deliberate boundary: a
per-scope walk would need a per-scope caller.

## Rain reaches soil (W1)

`CultivableMixin` gained the sky edge. `mm × m² = litres`, with the
millimetres from the integral and the square metres from the bed's
authored `landRequirementM2` — the footprint land use already charges it
for. **Drought becomes possible for the first time:** before this,
soil moisture only ever went down (drain) or up by hand (`water`).

⭐ A **pot catches nothing**, and that is correct. A pot draws zero land
because a houseplant is furniture rather than production — so it is
watered by hand, exactly as it ships. Ground that *is* production is
ground the sky can find.

Snow does not water the soil. The pack releases later, and elsewhere —
that is the watershed's integral, not the bed's.

### ⚠⚠ The unresolved ref, and why it has its own checkpoint

The covering `Locality` resolves **asynchronously** (an address walk);
the soil reconcile is **synchronous**. So there is a window — sometimes
across a restart — in which the ground genuinely does not know where it
is. This is the build's highest-risk item, and this codebase has been
bitten three times by a cache nothing warms reading null forever while
hand-constructed tests stayed green.

The discipline, in three parts:

1. **Cache the source's IDENTITY, derive its STATE.** The resolve stores
   the locality's *template path* and the sky-exposure flag; how much
   rain fell is derived live on every read.
2. **A checkpoint of its own.** `rainClockStamp` is separate from
   `soilClockStamp`. It opens at first touch — *before* the resolved
   check, so there is a backlog to back-fill — and then does **not
   advance** while the ref is unresolved. The first successful resolve
   therefore integrates the entire absence.
3. **Three states, not two.** `unresolved` reads `null` from
   `rainfallAbsorbedLitres()`. `resolved-to-sheltered` reads `0`. Those
   are different statements, and only the second one means "no rain".

The resolve is triggered on placement (`onMoved`, the shape
`ThermalMixin.restamp` established for `lastAmbientK`) and **kicked
lazily** from the reconcile whenever the ref is still unresolved, so a
bed restored from a snapshot into a room it never "moved" into heals
itself on the next read rather than staying blind forever. Concurrent
callers *coalesce* onto the in-flight promise — holding the promise
rather than a boolean is what makes `await restampWatershed()` mean the
ref is resolved when it returns.

`RainToSoil.test.ts` is written **unresolved-path first**, deliberately:
a test that hand-constructs the resolved value never exercises the path
that fails.

## Elevation (W2)

**`Zone.elevation`** — metres above sea level, inherited through the
ordinary `lookupField` ancestor walk, an authored value anywhere in the
chain winning over anything above it. `ZoneApi.elevationFor(scope)`
resolves it for a place.

### Why the zone, and not the biome

The zone already owns spatial geometry (`cellSize` drives volume,
light-scale and extent) and already does field inheritance. Biome owns
properties of the **air**; elevation is a property of the **ground**.

⭐ But the decisive reason is a **circularity**. `measure altitude`
computes `(P_sea − P_local) / (ρ·g)` from the biome chain's authored
`_pressure` — so altitude was back-computed from a number an author
typed, and putting elevation on the biome too would have given one
physical fact two sources of truth.

So **`_pressure` gained a derive-from-elevation fallback**:

```
P_local = P_sea − ρ · g · h
```

the linear hydrostatic form, chosen precisely because it is the
altimeter's own expression solved the other way. The instrument now
reads back the zone's height exactly. Pressure is the *consequence*, the
altimeter is honest, and `analyze atmosphere` reports the new
provenance `derived from elevation (<zone>)`.

**An authored value still wins.** The derivation fires only when the
chain walk fell all the way through to the **root universe biome** —
`sourcePath === /stuff/idea/biome/universe`, reached as `biome`,
`biome-ancestor` or the terminal `universe` step. That is exactly the
case where the 101 325 Pa in hand is the sea-level *reference* rather
than anything an author said about this place. A detail, room, biome,
biome ancestor or zone that names a pressure short-circuits first.

The weather deviation still rides on top, so a storm reads low over
whatever base the elevation step settled — which is also why a real
barometric altimeter is fooled by weather.

It returns to the reference untouched when there is no elevation, when
the elevation is zero (sea level *is* the reference), or when the medium
has no tabulated density — a vacuum has no barometric anything.

### ⚠ `coords.z` is not elevation

`z` is local and measured in zone **cells** — which floor of a building
you are on. A place's height is `zone elevation + z × cellSize`;
**hydrology reads the zone**, so a third-floor flat and the lobby are
the same point on the watershed, to the pascal. A stairwell is not a
waterfall. Terrain variation *within* a district comes from zoning
finer, which is what zones are for.

### Elevation is a COLD-PATH input (plan § P0)

`lookupField` is async, and nothing in this build reads elevation on a
hot path. It is *compiled* into two artifacts in contexts that are
already asynchronous — **reach ordinals** (at content load) and
**per-structure Δh constants** (at construction) — and every runtime
read is an integer compare or a scalar multiply. There is no
`lookupFieldSync`, no materialization stamp, and **no boot warming** —
the pattern that has failed in this codebase three times, where nothing
warms the roster and it reads null forever, silently.

## `Watercourse` — topology authored, direction derived (W3)

A watercourse is a data `Idea` in a catalogue, resolve-on-read — the
`Biome` / `Government` / `Corpo` / `Material` shape. The **class** is the
water pack's (`/system/water/idea/Watercourse`); the **rows** live in the
commons at `/stuff/idea/Watercourse/<key>`, exactly where `Locality` and
`Government` reference rows live and for the same reason: a river is a
fact about somebody's realm, and the realm's own pack has to be able to
edit it. A row under `/system/water` would be titled to the water group, and
world-seed could not touch the river it authored.

### A reach is not an object

A **reach** is a node identity on a course — `kestrel:confluence` — the
way a room already cites `_biomePath` and `_address`. Rights,
contamination and flow all key on it. A reach becomes a real object only
where content puts a **structure** on it: a dam, an intake, a weir.

Most of a watercourse runs through country nobody will ever stand in,
and Hinkley Lane settled the principle — *the unbuilt lots are prose,
not nine empty rooms.* It is also the wrong containment: a mill
**beside** the river is not **in** it, and a diversion right attaches to
a position that may never have a room at all.

⚠ Reaches are cited by **name**, ordered by **index**. The requirements
sketch a reach as `kestrel:4`; a positional citation would silently
re-point every right, intake and outfall the moment an author inserted a
node above it, so the durable half of the identity is the name and the
index stays an internal ordinal.

### Direction is derived; an author never writes an arrow

Nodes are declared source-first. Elevation is authored at **control
points** (source, falls, confluence, mouth) and **interpolated** linearly
between them, so an uphill reach is *unrepresentable* rather than caught
by a lint — there is no way to write one down. The source and the mouth
are control points by definition (there is nothing outside them to
interpolate from) and must both be authored.

Where elevation ties — a flat reach — the ordering falls back to the
authored node order, which is honest: a canal across a flat *is*
directed by how it was dug.

The parse refuses, naming names, when: a mouth is above its source; an
interior control point is above the one upstream of it; a source or
mouth authors no elevation; a course declares no basin or no nodes; two
courses claim one key; a branch names a reach that does not exist, or
one in another basin; or the drainage contains a loop. A failed parse
does not **stick** — the promise is dropped so the next read retries and
re-reports, rather than inheriting a rejection forever.

### One `branchesFrom`, two behaviours

A course names one parent node. **Which end of it attaches there is
derived from elevation**: whichever of the branch's own endpoints sits
closest to the junction's height is the junction. A branch attached by
its **last** node is a tributary joining; one attached by its **first**
node is a distributary leaving. One authored structure, both behaviours,
and no arrow anywhere.

### The compile: a reachability SET, not a graph walk

`WatercourseCatalogue` compiles, for each reach, the **set of reaches
downstream of it**. `compare(a, b)` is then one `Set.has` —
realm-wide, no walk, which is what the requirements ask for because
upstream/downstream is asked on hot paths (allocation, contamination,
navigability). A basin has tens of reaches, so the whole structure is a
few thousand strings.

A **set**, rather than the nested-set interval labels a tree would
allow, because *a delta is not a tree*: a distributary gives one reach
two downstream neighbours, and interval labels cannot express that. The
set is exact for any DAG and costs the same to read.

`compare` returns four answers, and the fourth is load-bearing:

| | |
|---|---|
| `upstream` / `downstream` | water at one reaches the other |
| `same` | one reach |
| `unrelated` | **different basins, sibling tributaries that only meet further down, or a citation naming no reach** |

⚠ "Not upstream" and "unrelated" are different. An allocation query that
conflated them would let a diversion in one valley curtail a right in
another.

### ⚠ Lazy, never warmed

Every public read is **async and self-loading**. This codebase has been
bitten three times by a reference roster that nothing warms reading
empty forever while hand-constructed tests stayed green, so there is
deliberately no "warmed vs cold" state to get wrong: the first caller
loads, everyone after hits the cache, and `invalidateCache()` drops it
for HMR. A `boot:` manifest entry would be an optimisation, never a
correctness requirement.

## A Locality declares its water (D21)

Two fields on `Locality`, and the first is the whole of what the
watershed asks of a place:

| field | |
|---|---|
| `_reach` | the reach citation this locality sits on and drains to, or `null` |
| `_catchmentKm2` | square kilometres draining to it, or `null` |

The address tree and the watershed are **two hierarchies, and their
misalignment is the point**. The address tree is *political* containment
— `terminus` → `city` → `campus`, with `_governmentKey` per locality.
The watershed is *hydrological* ordering — Rejection → Heart's Delight →
Terminus. Terminus governs its own streets and has no say over what
Rejection puts in the water. That is why a river authority is the one
institution that follows the second hierarchy while every other one
follows the first.

⚠ `null` is a normal state of the world, exactly as no government is. A
locality that declares no reach is off the watershed: it resolves no
relation with anybody, which is a different answer from "downstream of
everything". Three localities ship rootless today.

Catchment is **declared** per locality rather than derived per place:
deriving it would mean integrating an area over a world made of rooms,
most of which are indoors. The declaration is what turns the
precipitation integral into a river.

The kernel carries both as opaque strings and numbers — **it never
imports the pack**, and interpreting the citation is the pack's job.

## Flow, snowpack and navigability (W4)

### Flow is the integral's SECOND consumer

A bed multiplies the millimetres by its land area to get litres of soil
moisture; a reach multiplies the **same** millimetres by its catchment
area to get cubic metres of river. That is the whole spine of the build.

`WatercourseCatalogue.flowAt(reach, nowS, draws)` returns a
`FlowReading`: what is passing, what nature would deliver, how much of
it is melt, how much has been drawn, how much snow is still lying, and
whether a boat gets through.

Two terms make the natural flow:

1. **Runoff** — the mean liquid precipitation over the catchment's
   response window (30 game-days by default) × its area × a runoff
   coefficient. A *mean* rather than an instantaneous rate because a real
   catchment stores water: without the window a river would empty in a
   dry week and flood the moment it rained.
2. **Snowmelt** — water banked at altitude, released when the air there
   rises above freezing.

Mass conservation at watershed scale is an explicit non-goal: the sky
supplies and the sea absorbs, and the runoff coefficient is where the
rest of the water goes. Conservation applies to water in a **vessel**,
which is where a player can actually count it.

### Catchment: where the two hierarchies join

Each locality declares `_catchmentKm2` and `_reach`. The load folds that
area onto the reach and onto **every reach downstream of it**. A locality
that declares no reach contributes to nothing and is silently skipped —
being off the watershed is a normal state of the world, three localities
ship rootless, and it must never become an error.

The largest single contributor also becomes the reach's **climate
proxy**. Without it the realm would have one weather and Rejection's
valley could not be snowier than the city below it, which is the entire
premise of a headwaters town. A reach cannot resolve a covering locality
of its own — it is a position on a river, not a place you stand — so the
biggest contributor is the honest stand-in, and an unresolved one falls
back to the global field rather than to no weather at all.

### Flow is a TAKEABLE volume

`draws` is a ledger of withdrawals in m³/s keyed by reach. A draw at or
**upstream** of the reach being read is subtracted; one below it is not,
because that water already went past. Flow floors at zero — an
over-drawn reach runs dry, never negative.

The ledger is passed **in** rather than discovered, so the catalogue
never has to know conduits exist: `Conduit` supplies it (W5) and a test
supplies a literal.

### ⭐ Snowpack, and why altitude is the mechanism

`WeatherApi.segmentsBetween` gives the walk its season and type per
segment; the pack model is the **watershed's**, not weather's. For each
segment,

```
T_air = seasonalSeaLevelMean(season) + typeDeviation − lapseRate × elevation
```

snow accumulates below freezing and releases on a **degree-day** model
above it — the oldest and most robust snowmelt model there is, and the
right level of abstraction for a river you look at rather than forecast.

⚠ The seasonal sea-level temperature table is authored because **a reach
has no room to resolve a biome from**. The lapse rate then does the real
work: one number makes altitude the thing that banks snow, so the same
storm rains on the city and snows on the headwaters — and the headwaters
are where the ore is.

The result is a genuine hydrograph: a **spring rise** as the pack comes
off, and a **late-summer low**. That low is *why senior rights bind at
all* — without it, seniority never binds and the whole allocation layer
is decoration.

The snowpack walk looks back half a game year and is the most expensive
read in the build, so natural flow is **memoised per weather segment**.
Weather is piecewise-constant over six-hour segments, so the segment
index is a cache key whose invalidation is *by construction* rather than
enumerated.

### Navigability is derived, never authored

Flow **and** channel width, both. A torrent through a gorge is not
navigable, and neither is a wide trickle. Nobody writes down a navigable
stretch: a dry August closes one, and curtailing a junior right reopens
it. When the boat wave lands it reads a river that already knows where
it is navigable and knows that changes with the season.

## `Conduit` — the conveyance ladder (W5)

Getting water from a source to a place is **not a topology**. It is a
question terrain asks of every place, with three answers:

| mode | requires | costs |
|---|---|---|
| **haul** | nothing | labour and encumbrance — *ships already* |
| **gravity conduit** | the source above the destination | capital only |
| **pumped conduit** | power | capital **and** energy, forever |

The last two are the same object; which one you have is the sign of
`headM`, and nobody declares it.

⛔ **Nothing inside the delivered extent is modelled** — no pipe
segments, no street network, no `Street.ts`. *Coverage is legal,
connection is physical.* A conduit has two ends, and the review test for
anything proposed here is **does this add a node between an intake and a
delivery?** If yes, it is out of scope.

⚠ Three unrelated things in this codebase are called `Conduit`: the
kernel's sensory pass-through interfaces (`lib/boundary/Conduit`),
arcana's magical coupling item, and this. They are told apart by
namespace. Player-facing content gives its rows real names — *the city
intake*, *the Kestrel aqueduct*, *the Wharfside outfall* — so nobody in
the world meets the word twice.

### ⭐ A sewer is the same object reversed

`direction` is the only difference. **Supply** takes from a reach and
delivers to an extent; **disposal** takes from an extent and outfalls
into a reach. Storm drains likewise. The head expression is read the
other way and everything else — capacity, failure vocabulary, pump — is
shared, which is how *an outfall above an intake* becomes a fact about
terrain that nobody authored.

### Head is resolved once, at construction

`resolveHead()` is the P0 discipline in one method: elevation resolution
is an async ancestor walk and a conduit is asked for its state on hot
paths, so the walk happens where it is already asynchronous — a build
act — and every runtime read is a scalar comparison. `headM === null`
means **unknown**, and unknown is not "flat": both `isGravityFed()` and
`requiresPump()` answer `false`, and a caller that needs to tell the
difference asks for the head.

It uses `resolveEnclosingZoneForPath`, not the spatial variant: the
extent is a served area and very often a zone in its own right, and the
spatial variant deliberately returns `null` for a path that *is* a zone.

### The pump is hydro's equation, read backwards

`ρ · g · Δh · Q / η`. ρ and g are **reads, not constants** — gravity
ships as an authorable atmospheric trace and water's density as a
tabulated medium — so a world with different physics gets a different
pump bill without anybody editing the expression. ⭐ Water falling makes
power; water rising costs it. Water and power meet at one equation.

### Delivery: longest-prefix extent

The extent is a **template-path prefix**, resolved the way
`ParcelRegistry` resolves title. So *"am I on the main?"* is the same
question as *"who owns this?"*, asked of a different registry. ⚠ An
empty extent serves **nothing** — never everything, which is the
direction that would be catastrophic.

### The six-word failure vocabulary

`lib/supply/SupplyState.ts` — **kernel**, deliberately, even though the
water pack is its only speaker today: two packs that must agree on the
same six strings are an entangled namespace, and the alternative would
make a future power pack depend on the water pack for a word.

| | |
|---|---|
| `dry` | the **source** has nothing to give |
| `cut` | the **line** is physically broken |
| `frozen` | the line or the source is below freezing |
| `fouled` | what arrives is unfit — past what treatment removes |
| `off` | somebody **closed** it, and somebody can open it again |
| `overdrawn` | more is asked of it than its **capacity** carries |

Success is **not** in the list: a working supply reports `null`, so a
caller cannot treat "fine" as one more failure mode.

Only `cut` and `off` are **stored**. The rest are derived on read from
the river, the season and the demand — which is why a drought closes a
main without anybody running a job.

⭐ **Precedence** (`SUPPLY_STATE_PRECEDENCE`): `cut · off · dry · frozen
· fouled · overdrawn`. A supply can be in several at once, and the rule
is *the one furthest from being fixed by the person asking*. Reporting
`overdrawn` on a severed pipe would be true and useless.

### Excludability is why this is a business and a river is a law

A river is non-excludable and a conduit is not: you cannot keep someone
off a river, and you can close a valve on your aqueduct. So the river
gets **rights** and the conduit gets an **owner** — a consequence of
physics rather than a declaration.

## Storage and control (W6)

### `StorageNode` — a reservoir at a built or found elevation

A reservoir, a water tower and a cistern are the same object with
different numbers. What distinguishes a tower is that its elevation was
**built** rather than found — and that is a field, not a type.
`storageKind` is presentation only, and **no behaviour reads it**: a
tower behaves like a tower because its elevation is above the ground it
serves. The moment a rule branches on that field, elevation has stopped
being the mechanism.

Storage is not primarily for volume. It is for:

- **head** — a tank above you gives every tap below it pressure with no
  pump running, which is ⭐ **how a flat city gets a gravity conduit**,
  and Terminus is flat by construction;
- **buffer** — it decouples a steady supply from a peaky demand.

⭐ A tower is why water still runs during a power cut, for a few hours.
**The buffer size is the outage tolerance** — `outageToleranceS(draw)`
is that sentence as a method, and it is the number a town actually
argues about when deciding how big to build.

Filling costs `ρ·g·Δh·V / η` exactly when the source is **below** the
tank, and nothing when it is above — the whole economic argument for
gravity, and the third appearance of one equation. A full tank refuses
the surplus; a run-down store delivers what it has.

Roof-catchment harvesting falls out with no new machinery: catchment
area × precipitation, which is the soil edge's expression pointed at a
roof.

### ⚠ The level is the build's ONE piece of state

Everything else derives — flow from the weather, direction from
elevation, contamination from distance. A **level cannot**, because
outflow depends on what players drew and no function of time knows that.
So it is state plus a stamp on a `Persistable` host, and it is the one
place a restart could lose something. `Storage.test.ts` proves the round
trip through the spine itself: capture, lose the live value the way a
restart loses it, materialize, and the level comes back.

It is persistent but deliberately **not authorable**: a level is a fact
about the world, and a template that set one would be asserting how much
water is in a tank it has never seen.

### `ControlStructure` — the axis is CONTROLLED, not man-made

A dam, a headgate, a weir and a sluice are one thing: a structure on a
reach with a **setting** that decides how much water passes. That is
what actually distinguishes a canal from a creek and a reservoir from a
lake — not who dug it.

⭐ It is the most consequential object in the subsystem, for two
reasons: it converts flow **variability** into flow **reliability**,
which is the most consequential fact about water infrastructure in
history; and it makes the watershed **political**, because whoever holds
the dam holds everyone below. It also makes hydro dispatchable.

Two axes, one field. `passFraction` redistributes flow in **time** —
hold the freshet in May, let it down in August. `divertsTo`
redistributes it in **space** — send the held share into a canal. Both
are the same arithmetic; what differs is where the water ends up. A
canal is *a watercourse with a control at its head*.

What origin changes is **legal, not physical**: a natural course is a
commons allocated by law, a built work is sunk capital allocated by its
builder, and where the users collectively own the works that is an
irrigation district. None is a different object.

### Hydro: `ρ·g·Δh·Q·η`, and it rises and falls with flow

The paying direction of the same equation. Generation reads the
**arriving** flow rather than the passed share, because the water goes
through the machine whichever way it leaves — and a dam that generated
less when it diverted more would make the two axes of control interfere
for no physical reason.

`headM` is **fixed per structure, at construction** (P0), so the runtime
read is one multiply.

### Live withdrawals and outfalls: ONE scan, not a registry

`WatercourseCatalogue` walks the resident objects once and asks each of
them two questions — *do you take water* (the `Withdrawing` shape) and
*do you put anything in it* (`Discharging`). They are the same question
asked of the same objects, so they share one pass.

⚠ **Derive-on-read, not a registry.** A registry that objects joined at
`postRegister` would need an ordering, an eviction hook and a
re-registration on materialize, and every one of those is a way for the
roster to go quietly stale — a failure this codebase has paid for three
times. A scan cannot go stale.

⚠⚠ **And the scan is deliberately NOT memoised**, unlike natural flow.
Caching it looked free and was a correctness bug in waiting: a player
who shut a sluice would have watched the river stay dirty for up to six
game-hours, because the closed outfall was still sitting in a cache
keyed on the *weather*. The walk is a `typeof` per resident object and
costs microseconds; the expensive part is the snowpack integral it calls
per withdrawer, and that is already memoised per reach per segment.
**Cache the expensive derivation, never the enumeration.**

⚠ **Why not MQL**, which is normally how you search: MQL selects by
**mixin**, and a capability pack cannot ship one (its module categories
are branches, controllers and tests — no `lib/`). Its `class.X` filter
matches by class *name*, and three unrelated things in this codebase are
called `Conduit`. So a shape scan is the honest mechanism available to a
pack — and `check-world-scan` was extended to walk packs' `src/` so the
choice is a diff a reviewer sees rather than a hole in a gate.

A withdrawal is sized against the **natural** (undrawn) flow, because
sizing it against the already-drawn flow would be recursive and because
the honest rule is that a headgate is sized by what the river brings it.
A supply conduit draws its **capacity** rather than a demand: domestic
metering is an explicit non-goal, so there is no demand model to ask,
and an over-subscribed main in a dry August is exactly what gives the
rights layer something to bind against.

## Rights (W7)

A water right is **a volume per window plus a priority date**. Without a
volume it cannot be over-subscribed; without a date it cannot be senior.

### Two doctrines, one mechanism

| | how it exists | who has one |
|---|---|---|
| **prior appropriation** | an explicit, dated, transferable **record** | whoever filed — first in time, first in right |
| **riparian** | **derived**, with no record at all | whoever owns land on the bank, in equal share |

The substrate ships the **record** form because it is the superset;
riparian is a *derivation rule over the same shape*. A polity's doctrine
is therefore a **choice** (`allocate(..., { riparian: true })`), not a
second implementation, and both answer one query.

⭐ Riparian being record-free is not a shortcut, it is the doctrine: you
have a right because of where your land is, and nobody wrote anything
down. It also means the riparian path needs **no filing authority, no
gate and no verb** — which is why it is the one a player has on day one.

Every riparian right has priority date `0` — **simultaneous by
construction**. A drought shrinks every glass rather than emptying the
junior ones, which is exactly what "equal share" means.

### The parcel's reach citation

`ParcelRecord.reach` is the field riparian derivation reads. It lives on
the **parcel** rather than being derived from the address tree because
⚠ **the two do not correspond**: a room's content path and its
`_address` are independent by design — the market square sits at
`/world/terminus/market/square` and is addressed
`terminus/city/counting-houses/market-square`. A citation on the land is
the only honest link between a title and a river. `ParcelApi.citeReach`
sets it; a pack's `requires.title` claim can declare it.

### The `water-right` document kind

⚠ **A pack cannot declare a `DocumentKind`** — the kind's consumer is
code and the installer needs its go-live hook, so *editing that file is
a platform act*. The **kind** is the platform's; the **validated save**
that decides what a legitimate right looks like is the water pack's.
That is the whole of P1's split, in one feature.

Path-keyed at `/system/water/rights/<course>/<node>/<id>`, because seniority is
`list(prefix)` plus a sort — which wants the reach in the path rather
than in an index. `onVanish: 'keep'`, **not** `'delete'`: a right is a
record of something that happened, and no absent file should erase one.

### Allocation

Filed rights sort by priority date, oldest first, and are served in full
until the water runs out. ⭐ **The junior is the one that goes short** —
in a dry August the newest claim gets nothing while the oldest gets
everything, and that asymmetry is why a senior right is worth money. Ties
break on the id so an allocation never reorders between reads.

### The quota rides the RIGHT, not the source

A per-window counter against the holder's own record. ⚠ The refusal
**exposes no other holder's draw**, and that is a property of the
*shape* rather than of what a caller prints: `quotaRemainingM3(right,
drawn)` is pure over its two arguments and consults no register at all.
There is nothing for it to look up, therefore nothing to leak, and no
leaderboard can ever be built from it. *Aggregate, never report.*

### ⭐ Navigation is a claimant who is not a farmer

`navigation` is a use in the vocabulary carrying a **minimum flow**
rather than a volume. It takes **nothing** — it is a condition on what
survives everybody else — and a claim whose minimum is not met is
**stranded**. One entry in a vocabulary buys the classic water fight
(navigation versus irrigation — the Missouri, the Colorado) out of
quantities the build already computes, and it is why an upstream
diversion can strand a river and curtailing a junior right un-strands
it.

## Contamination and the counterplay ladder (W8)

### One substance, a LEVEL, and a KIND

A single contamination number is not enough, and the reason is the most
important fact about water pollution: **self-purification**. A river
cleans itself of sewage over a few miles and never cleans itself of
lead. One number would make the smelter and the outhouse the same
problem.

| kind | downstream |
|---|---|
| `organic` | **decays with distance** — the river recovers below the town |
| `persistent` | **does not decay** — the river never recovers below the smelter |
| `sediment` | settles, faster than sewage; raises turbidity while suspended |
| `nutrient` | does not decay; accumulates where residence time is long |

Survival is authored **per hop**, not per metre, because a reach is the
unit the whole subsystem reasons in and a per-metre rate would demand a
channel length nobody has authored. The numbers are dials; **the
ordering is the model**.

### It is a CONCENTRATION, so a dry month is a dirty month

`contaminationAt(reach, now)` sums every upstream outfall's load,
attenuated by distance according to its kind, and **divides by the
flow**. The same outfall fouls a summer trickle far worse than a spring
freshet — which is how the seasonal hydrograph reaches the health of the
water without anybody wiring the two together. A reach that has run dry
reports "filthy" rather than "infinite", which is both truer and the
only answer a caller can do arithmetic with.

### ⭐ The map is the argument

Whether an intake sits above or below an outfall is **already a fact
about the terrain**, derived from elevation and **authored by nobody**.
`contaminationAt` only reads that fact back. An intake above the outfall
is clean; the same intake a reach down is not; and moving it is free,
which is historically the first real answer anybody found. Shutting a
sewer's gate cleans the river below it with no rule saying so — the
outfall simply stops answering the discharge scan.

### The counterplay ladder

1. **Move your intake** upstream of the outfall — free, permanent, and
   the thing the map already tells you to do.
2. **Boil** — personal, per-use, costing fuel and time *every single
   time*, which is exactly why a town eventually pays for
3. **Treatment** — an attribute of a conduit: capital once, then
   nothing. `foulingOf` applies it to what arrives, so investing in
   treatment moves the `fouled` threshold rather than the river.

Only a **supply** can be `fouled`: a sewer carrying filth is a sewer
working.

### `boil` — the one new verb in the build, and it is an ACT

The extend-before-inventing check found a home for everything else;
`boil` had none, and it is `crafting` — it transforms matter with heat,
like the shipped `heat`. The **fire affords it** (`FurnaceMixin`'s
contributions), because you cannot boil without one and there is no
separate kettle to own.

⭐ **What boiling does is decided by what you boiled, never by the
controller.** Two consequences, neither privileged, and a target may
earn both:

| if the target is… | boiling… |
|---|---|
| a **build** (`Builds` — a cook pot) | latches the heat it reached and records the method `boiled`, which a recipe's reverse-match reads |
| a **vessel** (`Bulkable`) whose contents declare `purifiedByBoiling` | swaps the material for what that names |

⚠ **The first cut of this verb was a purifier, and that was a defect.**
Its arg required `BulkableMixin`, it recorded nothing, and its single
consequence was hardcoded — so a `CookPot`, which is
`ManualBuild + Tool + Durable` and *not* `Bulkable`, was refused
outright; no recipe could ever have required "boiled"; and a second
consequence would have meant a **kernel edit per trade**, which the pack
doctrine forbids. The cooking trade would have had to fight this verb
rather than extend it.

It now follows the shape the crafting branch already uses twice — `heat`
latches `noteHeat` on any `Builds` host, `stir` records a method, and
the **recipes** decide what those mean. `Technique` is an open
vocabulary by construction, so `boiled` costs no kernel list, and a
cooking pack gets boiling by authoring
`{ requiresHeatK: 373, method: boiled }` and changing nothing in the
kernel.

Purification is therefore a property of the **material**
(`Material.purifiedByBoiling`), not the definition of the verb. It is a
change of material rather than a mutation, because a `Material` is a
**shared reference Idea**: one row backs every litre of that stuff in
the world, so purifying by editing the material would clean every river
at once. The closed-material doctrine says the same from the other side
— *materials are a fixed set and blends derive*, so the way to say "this
water is different now" is to name a different material.

⭐ **The most important behaviour is still the one where boiling does
nothing.** A material that declares no counterpart just gets hot — the
command is not refused, because you really did boil it — so boiling a
lead-fouled river gives you hot lead-fouled river, and the player learns
the difference between organic and persistent contamination the way it
is actually learned. The threshold is the contents' **own** boiling
point, not a dial.

### The toxin route needs no new machinery

`Material.toxicity` and the metabolism clearance path already ship.
Fouled water is a material carrying a `ToxinTag`; drinking it doses
through the same route as everything else, and the water it boils into
carries none. The build adds one field and no mechanism.

## Verbs: the check ran, and the answer was mostly "no new verb"

Every act in this build is *operating a built mechanism*, which is the
`device` category's own definition. Running the extend-before-inventing
check:

- **`pump` already ships** — a bellows verb on a furnace. A pumped
  conduit is powered machinery, not a hand crank, so the build needs no
  pumping verb at all and `pump` is left alone.
- **`switch`** covers a conduit's valve: `Conduit` composes
  `SwitchableMixin` and that is the whole of its on/off surface.
- **`analyze`** gained a `water` subcommand rather than a new verb.
- **`boil`** is the one act with no plausible existing home (W8), and it
  is `crafting`, afforded by the fire.

### `analyze water`, and how the kernel reads a pack object

Bare, it reports the ground you stand on — the reach the covering
locality declares, what is passing, how much is snowmelt, how much snow
is still lying above you, whether a boat gets through. Pointed at a
waterworks, it reports that instead.

⚠ **The kernel does not import the water pack.** Both readings go over
shapes: a supply answers `supplyReport` (`SupplyReporting` in
`lib/supply/SupplyState.ts`), and the drainage catalogue is reached by
template path rather than by module — the `HoldingView` seam the
residences build established. A realm with no water pack installed gets
an honest "nothing here knows about water", never a crash. Ground that
drains nowhere says so, and that is a normal state of the world.

The arg is declared `requires: any` deliberately: a waterworks is
recognised by the shape it answers, not by a mixin, because there is no
kernel mixin here to name.

## The realm on the water (W9)

### Three basins, and the third one is the decision

| watercourse | basin | what it is |
|---|---|---|
| **the Kestrel** | `kestrel` | the home trunk: headwaters 1400 m → falls 500 → **confluence 30** → estuary 0 |
| **the Delight** | `kestrel` | the farming valley, joining at the confluence |
| **the Holloway** | `holloway` | the neighbouring drainage; the moor sits on it |
| **Cold Fell** | `cold-fell` | high, clean and **empty** — the aqueduct's source |

Read the Kestrel downwards and it is the realm's political economy in
five nodes. The ore and the snowpack are **both** at the headwaters,
which is why the mining town sits there and why what it puts in the
water is upstream of everybody. The farming valley joins at the
confluence and argues about volume. **Terminus is the confluence**, and
drinks last.

⭐⭐ **Cold Fell is over the ridge from Rejection, and that is the whole
decision.** A neutral third basin would let money simply exit the
politics. Routing the aqueduct past the town the city already fouled
means its attempt to escape **re-entangles it with the same neighbours
on worse terms** — the Owens Valley story. Capital becomes a move in the
game rather than an exit from it, and Rejection, Heart's Delight and
Hinkley are all still drinking the Kestrel.

⚠ **This wave authors no towns.** Rejection and Heart's Delight do not
exist as content and belong to their own locality builds; what lands
here is the water they will sit on, and the model asks them for one
declared field.

### Wild catchment, and why a node declares one

`WatercourseNode.catchmentKm2` is the **unclaimed** ground draining to a
reach — the fell nobody's locality covers. A locality declares the
ground it sits on (D21) and the two add. Without it, a realm with three
villages would have a river the size of a gutter and the seasonal
hydrograph the entire rights layer depends on would be unobservable. It
is a *declaration* for the same reason a locality's is: integrating an
area over a world made of rooms, most of them indoors, cannot be done
honestly.

### The numbers that are load-bearing

| | |
|---|---|
| `/world/terminus` zone | **35 m** — the city is FLAT by construction |
| `kestrel:confluence` | **30 m** — five below the bank, which is what makes a bank a bank |
| `/world/terminus/hinkley-hills` | **130 m** — a hundred above its own river |
| `cold-fell:cascade` | **1150 m** — 1115 m of head over the city |

`/world/terminus` is a new `FolderZone` carrying nothing but the city's
elevation: say it once, and Hinkley overrides it where it is false. That
is the inheritance walk doing its job.

### Wharfside — the city finally meets its river

The build authors **the water's edge**, not the district: the bank at
the confluence, the **intake** and the **outfall**. The dockers' hall,
the chophouse and the west bank belong to the city track, zoning and
freight.

⭐ Both works sit on the same reach a hundred paces apart, and the
intake is above the outfall — so the water the city draws is clean and
the water below the arch is not, **derived from terrain and authored by
nobody**. Standing at Wharfside and seeing both without turning your
head is what makes *the map is the argument* something a player can look
at rather than infer.

⚠ The city's intake **needs a pump, forever**: the river is at 30 m and
the city at 35, five metres the wrong way. That is not a flaw in the
content — it is the reason a tower and an aqueduct are both worth
building, and `analyze water the intake` says how many kilowatts.

### The Cold Fell aqueduct

1115 metres of head, **gravity-fed end to end**, costing nothing to run,
with a **weir and a turbine** on the last drop taking the energy out of
water that was going past anyway (`ρ·g·Δh·Q·η` — the third appearance of
one equation, in the direction that pays). It is **treated**, unlike the
Wharfside intake: capital once, then nothing.

⭐ It is the distinction the build turns on: **the body you form around
is not the source you drink from.** Your reach is geography you did not
choose; your conduit's intake is capital you did. It is also the first
thing in the game a single player almost certainly cannot build alone —
a cooperation gate that emerges from physics rather than from a lock.

### Hinkley's is a head problem, and the District's first real job

The suburb is a hundred metres above its own river, so a main would be
pumped forever. It stores instead: the **District tank** at 145 m, giving
the standpipe fifteen metres of head and no pump at all.

⭐⭐ **The works are the Hinkley Hills Improvement District's** (D27),
and the reason is a story the content already set up: the District ships
as a shell — charter `""`, treasury `""`, no departments, no seats,
described as having *"never once been asked to do anything else."*
**Water scarcity giving a paper institution its first real job** is a
better story than chartering a new authority. It also keeps the
aqueduct out of reach: a district of smallholders cannot fund one, so
the aqueduct stays the city's move and the District stays the
counterparty that cannot follow.

### ⚠ The valley road exists because of a live gap

Nothing in shipped content had an exit **into** Hinkley Hills except the
Teleport Authority route; the arrival room's only exit ran further in;
and nothing granted Hinkley's travel node. Fast travel is
**reach-before-travel** and Hinkley is not among the three a character is
born with — so a player had to walk there, and there was nowhere to walk
from. The e2e reached it with `wizard: true`, **which is the tell: a
wizard stand-in is a missing player path.**

    market square → the bank → the towpath → the narrows
                  → the shoulder → the Hinkley Hills stop

Five ordinary rooms, walkable both ways, no gate and no wizard check
anywhere on it. ⭐ It is also the working demonstration of D21: roads
follow rivers because rivers cut the only gradeable path, so the land
route between two places in one basin **is** the water route. Corridors
get a reason instead of being arbitrary, and the walk up the valley —
35 m at the towpath, 130 m at the stop — is how a player learns the
watershed.

## Where the code lives

| | |
|---|---|
| **kernel — the physics** | zone elevation (`lib/zone/Zone.ts` + `ZoneApi.elevationFor`) · the precipitation integral and the segment walk (`api/weather.ts` + `platform/idea/api/WeatherLogic.ts`) · the pressure fallback (`platform/idea/api/BiomeLogic.ts`) · the rain→soil edge (`lib/husbandry/Cultivable.ts`) · the supply vocabulary (`lib/supply/SupplyState.ts`) · the `water-right` kind (`lib/document/DocumentKinds.ts`) · `ParcelRecord.reach` · `Material.purifiedByBoiling` · `analyze water` + `boil` |
| **`water` pack — the works** | `Watercourse`, `WatercourseCatalogue`, `Conduit`, `ControlStructure`, `StorageNode`, `WaterRightRegistry` at **`/system/water/<branch>/`** — `packages/content/water/src/` |
| **content** | the basins and Terminus's works — `world-seed` (the watercourses + the localities' declarations), `terminus` (Wharfside, the valley road, the aqueduct), `hinkley-hills` (the District tank) |

The split follows arcana's membership test: **a capability pack holds
what other packs' content names.** The kernel never imports the pack —
where it must read one of these objects it goes by **shape**
(`SupplyReporting`) or by template path, the `HoldingView` seam the
residences build established.

## Dials

Everything tunable lives in **one** authored file,
`packages/content/water/content/settings/water.yaml`, because the
numbers only make sense read against each other: how hard it rains
decides how much a bed gets, which decides how much a reach carries,
which decides whether a right binds.

⚠ Every key has a **seeded literal at its call site**, so the kernel
behaves correctly with the `water` pack absent and the dials only ever
*retune* what already works. A dial that only works when a pack is
installed is a dial that fails silently.

## What this build deliberately did NOT do

- **Domestic metering.** The mains stay effectively unlimited at the
  household tap; rivalry lives at agricultural and industrial scale,
  where it belongs.
- **Mass conservation at watershed scale.** The sky supplies and the sea
  absorbs. Conservation applies to water in a **vessel**, which is where
  a player can count it.
- **Pipe segments and within-district networks.** *A conduit has two
  ends.* The review test for anything proposed here: **does this add a
  node between an intake and a delivery?** If yes, it is out of scope.
- **Treatment as a modelled process.** It is an attribute of a conduit,
  not a plant with stages.
- **Boats, swimming, on-water rooms.** Navigation is a *claim*; the
  vessel is fishing's and freight's. When that wave lands it will read a
  river that already knows where it is navigable, and knows that changes
  with the season.
- **A new civics jurisdiction tier.** A river authority is a **firm**.
- **Any new collection.** `pnpm lint:schema` still reports 48.

## History

The build landed as eleven waves (W0–W10) on `build/water`
(`681cb2d9a..` through the pre-merge sweep), against
`docs/plans/water-plan.md` D1–D27 — both the plan and the requirements
doc retired at the sweep, per workflow.md.

Four shifts between design and implementation are worth knowing, because
each one is a rule rather than a detail:

- **`boil` shipped as an OUTCOME and was rewritten as an ACT, in review.**
  Its argument required `BulkableMixin`, and a `CookPot` is
  `ManualBuild + Tool + Durable` and *not* `Bulkable` — so `boil pot`
  refused outright. It recorded nothing, so no recipe could ever have
  required "boiled", and its one consequence was hardcoded, so a second
  would have meant **a kernel edit per trade**. ⭐ The general rule: *a
  verb names what you DO; what it accomplishes is a property of what you
  did it to.*
- **The pack moved from `/water` to `/system/water`**, taking `/arcana`
  and `/residence` with it, once the question *"what class of thing does
  water belong to?"* was asked. The five-axis namespace taxonomy in
  CLAUDE.md is the answer, and its test is that **a system is true
  whether or not anyone is participating in it**.
- **The world scan is deliberately UNCACHED.** Memoising it meant a
  player who shut a sluice watched the river stay dirty for six
  game-hours. ⭐ Cache the expensive *derivation*; never the
  *enumeration*.
- **`ParcelRecord.reach` is a citation because the address tree does not
  mirror the content tree** — checked rather than assumed. The market
  square lives at `/world/terminus/market/square` and is addressed
  `terminus/city/counting-houses/market-square`.

⚠ **Still open after the build**: acceptance 20's live half (walk
Terminus → Hinkley Hills and `register` the terminal) is proven as
content and asserted in both directions with no wizard in it, but has
never been driven in a session — blocked by two boot failures in packs
this build does not touch. And `wiki-spoiler-fields.snapshot` walks only
the kernel tree, so a capability pack's `authorable` fields are still
invisible to acceptance 28's reveal audit — the same blind spot
`check-world-scan` had until this build widened it.
