# Holding — the residential ladder: plats, warrens, programmes, condition

A **holding** is a place somebody holds: a dorm room, a let unit, a
house on a bought lot. This doc is the source of truth for the substrate
all three ride — the two-tier elastic institution, the authored plat
plan, the per-holding programme, the front door, the weathering clock
and the acts that read and repair it.

The governing shape, and the whole reason there is one doc rather than
three:

> **A holding is a warren one level down.** An institution (a dorm, an
> apartment building, a subdivision) is a `Warren` whose members are
> holdings; a holding is a `Warren` whose members are ROOMS. The two
> tiers are the same machinery at two cardinalities, which is why the
> dorm's one-room bedsit and Hinkley's six-room house are one code path
> with different authored data.

The ladder as shipped:

| Rung | Institution (a `CirculationWarren`) | Holding (a `RoomWarren`) | Tenure | Upkeep term |
|---|---|---|---|---|
| Granted | `DormWarren` (Duncan Hall) | one room | parcel grant (lease) | `institution-all` |
| Let | `BuildingWarren` (Seznick House) | a multi-room unit | parcel grant (lease) | `landlord-shell` |
| Owned | `PlatWarren` (Hinkley Hills) | a house on a lot | parcel **title** | `owner-all` |

The dorm is documented end-to-end in [residence.md](./residence.md);
this doc is what it converged ONTO.

---

## Every warren is a RoomWarren or a CirculationWarren, and the compiler asks

Before the four roles, the vocabulary they sit in — because it applies
to **any** warren, not just residential ones:

| | members are | occupancy means | it owns | base |
|---|---|---|---|---|
| **`RoomWarren`** | ROOMS you stand in (a house's hall and kitchen, the lounge's satellites) | who is standing in a member | the rooms | `lib/location/RoomWarren` |
| **`CirculationWarren`** | HOLDINGS (each a `RoomWarren`) | who is anywhere inside a member's rooms | the CIRCULATION that reaches them — lane reaches, corridors, stairs, minted from a `PlatPlan` as holdings fill and reaped outside-in | `lib/location/CirculationWarren` |

⭐ Named for what each IS, not for being the other's opposite (the user,
2026-10-18, in the cleanliness review: *"InnerWarren and OuterWarren …
imply a kind of universality the warrens don't earn"*). *Circulation* is
the architects' word for corridors, stairs and streets taken together, and
the class's own vocabulary already (`circulationForNode`).

`Warren` declares `occupantsOf` **abstract**, so a new warren cannot be
written without choosing. That is not ceremony: the leaf answer used to
be the concrete default on `Warren` and the circulation tier had to remember
to override it — and a warren that forgot would read every holding as
empty and **reap an occupied house**. The choice is now made once, by
which base you extend, and the compiler will not let you skip it.

A `RoomWarren` also refuses a warren member outright (`addMember`
throws), so the tiers cannot quietly nest the wrong way round.

The naming rule falls out: **every warren ends in `Warren`.**
`DormWarren`, `BuildingWarren`, `PlatWarren` are `CirculationWarren`s;
`HoldingWarren` and `LoungeWarren` are `RoomWarren`s.

## The four roles

Four classes, and each answers exactly one question. Getting a change
into the right one is almost always the whole design decision.

| Role | Class | Question it answers |
|---|---|---|
| The institution | `lib/location/CirculationWarren` (kernel, abstract) | *Which holdings exist, and how do you get to them?* |
| The layout | `lib/location/PlatPlan` (kernel, value object) | *Where does the next holding GO?* |
| The holding | `residence/idea/HoldingWarren` (pack, an `RoomWarren`) | *What rooms is this one made of, and what shape is it in?* |
| The way in | `residence/idea/FrontDoorExit` (pack) | *May you cross this threshold?* |

Two of the four are kernel and two are a capability pack's, and the line
is not arbitrary: the elastic-graph machinery and the layout algebra are
substrate anything could use, while a *residential programme* is content
vocabulary — a floorplan, a tenure term, a weathering clock. The kernel
never imports the pack; where it must read a programme (the `survey`
verb) it does so through the `WarrenMember` back-ref BY SHAPE, or —
where it needs an institution — through the location graph's record,
which attaches each institution to its own `parentExtent` (the scan
sweep, 2026-10-14; a pack catalogue that scanned the template table was
retired). ⚠ It used to reach through MQL **by class NAME in a
string**: a kernel→pack dependency the type system cannot see, priced at
the size of the world. `CirculationWarren.admitFor` and `title list` were the
two sites; the `maintains` brain was a third and moved into the pack
outright.

---

## Tier 1 — `CirculationWarren`, the institution

`abstract class CirculationWarren extends Warren`. Everything three
institutions had in common, lifted out of `DormWarren`:

- **the holdings map** (`_holdingsByKey`) — a holding per parcel extent,
  stood up lazily and reaped when empty;
- **the circulation map** (`_circulationByNode`) — the corridors, floors
  and road segments you reach holdings THROUGH, minted on demand from
  `circulationTemplateFor(node)`;
- **the provisioned + keyway caches**, refreshed off
  `ParcelApi.childParcelsOf(parentExtent)` — who has a grant or a title,
  and what keyway opens their door;
- **`admit(key)`** — cache → stand the holding up → wire it → return its
  entry room. The single way in, from a gate, a door, a re-entry after
  dormancy, or a boot re-hang;
- **`capacity()`** — the operator dial (below);
- **the reap invariant**.

### The reap invariant

> **Circulation reaps outside-in.** A node never reaps while a live
> holding hangs off it, or while a live node sits beyond it on its road
> or stair. The graph stays contiguous back to the authored entrance.

An empty middle floor stays standing if floor 3 is occupied, because a
tenant on 3 must be able to walk down. The same rule, horizontal, is
what makes Hinkley's lane honest: segment 2 is impassable until its
frontage sells, and it does not vanish once segment 3 has a house on it.
The rule lives once, in the base's `reconcile`; no institution
re-implements it.

### The policy hooks

Four, and a subclass is expected to override exactly these:

```ts
protected standUpHolding(key: string): Promise<MemberStuff>
protected circulationTemplateFor(node: string): string | null
protected wireCirculationNode(node: string): Promise<void>
protected entryEdgeFor(key: string): Promise<Exit | null>
```

`DormWarren` stands a holding up as a programme over the `dormroom`
row and hangs a `DormDoor`; `BuildingWarren` does the same with
`FrontDoorExit` off a corridor; `PlatWarren` does it with a
`LotGateExit` off a road segment. Nothing else differs.

---

## Tier 1½ — `PlatPlan`, the layout as authored data

Where the next holding goes stopped being arithmetic in a class and
became a field on the institution's row. Three shapes:

| Shape | Node | Slot grammar | Who uses it |
|---|---|---|---|
| `static` | an authored node | authored | a venue with a fixed set of rooms |
| `linear` | an index (a floor) | `f<n>-r<p>` / `f<n>-u<p>` | Duncan Hall, Seznick House |
| `branched` | `(road, segment)` | `lot-<n>` | Hinkley Hills |

The surface is four reads — `nodeOfSlot`, `nextFreeSlot`,
`nodesInOrder`, `isAuthored` — and it holds no state. The branched shape
carries `roads: [{ key, segments, frontagesPerSegment, branchesFrom? }]`,
which is enough to say *"the lane runs west four frontages at a time, and
a court branches off segment 2"* without a line of code, and enough for
`isAuthored` to mark segment 1 as the hand-written lane while segments 2+
are minted from a road-segment row.

That last point is the reason the shape exists: **bespoke streets,
minted homes, one institution.** A locality author writes the piece that
should be written and lets the rest grow.

---

## Tier 2 — `HoldingWarren`, the holding

An instanceable `FloorplanWarren` (the kernel base it shares with a ship's
warren — [the lift](#-the-lift-landed-floorplanwarren-in-the-kernel-harbors-p1))
— deliberately **not** a singleton, because there is one live
instance per holding, all sharing one row and separated by their
persistence key.

**Authored on the row** (one row, every holding of that kind):

- ⭐ **the plan** — `extent`, `entrance`, `footprintM2`, `buildings:`
  (harbors review 3; 2026-11-03 — on the programme row itself, never a
  blueprint row: a `Structure` is only ever a thing standing in the
  world). It replaced the `floorplan:` list, a second schema that
  re-stated every room's exits, its entry and its layout outside the
  rooms. `extent` holds the kind's rooms authored as an ordinary place —
  ordinary rows with ordinary `exits:` between them, the authored
  Hesper's shape — and `entrance` is the way in. A locked threshold is the
  way's own `kind: /system/residence/idea/exits/keyed-door`; the holding
  hands it its keyway at mint (`fitExit`). The way back out is a plain
  passage the plan authors. A tract's `share` is the tract row's own
  field; `footprintM2` is the ground the buildings cover; `buildings:`
  names them, main first. See
  [§ Plans](#-plans--a-kind-of-building-as-an-ordinary-place-on-its-owner-harbors-review-3-2026-11-03).
- `upkeepTerm` — the tenure vocabulary (below).
- `addressBase?` — the human address the per-room addresses hang off.

**Persistent per instance:** the floorplan and the term. ⭐ The SHELL is the
holding's BUILDING's now (paint build D6/D13): `wake()` stands up a
`HoldingBuilding` per building of the holding (or reads the Building row
its units stand in — Seznick House), and the fabric lives there.

**`wake()`** is the whole of standing a holding up: every room of the
plan is stood up as a **keyed instance** — `(scope = the room's row,
key = <holdingExtent>/<leaf>)`, the leaf being the row's last path
segment — then each room's authored ways are copied (a way into the
extent leads to the same room of this copy; a way out stays), the keyed
doors are fitted, and each room is stamped with its address.

### Dormancy is whole-holding, never room-by-room

The population witness aggregates across the holding's rooms. When the
last interactive leaves, the programme captures **every** room and
itself, then reaps the whole set. A half-reaped house — a bedroom
dormant while the hall is awake — is not a state the model admits, and
the reason is placement: a good is placed against a room's `(scope,
key)`, so a room that sleeps while its siblings wake would have to
resolve placements against a graph that is half gone.

### The plan is the INITIAL mint, never the closed set

⭐ **A member persists only if it keeps state of its own** (tracts build
review, 2026-11-01). `wakeRoom` restores a room's `(row, key)` record only
when the room composes `Persistable`; a plain room — a farmyard — comes up
from its row each wake and keeps nothing. Before, every member was forced
to persist. Persistence is the exception: a room keeps a record when it
must lay an owner's goods back down (the room overlay — a house's rooms,
a farm's barn) or carries state nothing else does (a tract's soil).
A ship follows the same rule (harbors review 2): her **hold** (`Hold`,
`Persistable(Compartment)`) keeps a record — her cargo — and her deck and
cabin are plain `Compartment`s. Because a ship sleeps whenever the last
person leaves, her warren also stows what is loose up top into the hold
before she sleeps and keeps the row gear each plain room was short of
(`ShipWarren.stow` / `shortOf`), so a plain room's `props:` are her
fit-out and never a faucet.

(⚠ `admitPlot`, the runtime-added-member seam below was written for, was
retired with `plot` by the tracts build; a field is marked ON a tract
now, never admitted as a room.)

⚠ Load-bearing for the farming build's Stage B: a programme's member set
stays **open to runtime-added members**. Break-ground buds field rooms
into a holding from a programme-level ledger, on this base. Nothing in
`wake`, dormancy or capture may assume floorplan-only membership.

---

### ⭐ Tracts in a layout — shares, footprints and the mirror (tracts build, 2026-10-31)

A farm is a holding like a house: its plan holds its rooms AND its
**tracts** of open ground (soil.md § Tracts and fields). Three things
make one layout serve every parcel it is sold on:

- **`footprintM2`** on the programme row (the plan's owner) — the ground its
  buildings stand on that no tract covers (the farmstead 670: the yard
  400, the barn 150, the farmhouse 120; a Hinkley house 220). (It was per
  floorplan room; only the sum was ever read, and it is a set of rooms'
  fact.) **`openGroundM2()`** = the parcel's workable area
  (`ParcelApi.workableAreaOf`) less it.
- **`share`** on a tract's own row (`Tract.share`) — its fraction of that
  open ground. The tract
  sizes itself when the holding wakes it (the kernel's
  `WarrenMember.onWarrenWake`, called from `wakeRoom`), so a copy on a
  bigger parcel has bigger tracts, in proportion; on unmeasured ground it
  is nothing, and `mark` says why.
- **The mirror** — `mirrorAxis`, set by the plat from the side of the
  road the lot is on (`PlatPlan.mirrorAxisOfSlot`: a lot on the LEFT of
  the road's heading, ring index 1/3/5, is reflected across the road's
  axis) before the holding wakes; the copy reads every way it copies
  through it (`up`/`down` and the axis never flip), and its layout is read
  from those ways. So every
  copy faces the lane, and the way out is the gate reversed
  (`wireHubExit` — it was hard-coded `south`). ⚠ Hinkley's left-side
  lots mirror now: the house on the south side of the lane has its hall
  south of the yard.

The first layouts: the valley farmstead (Hearts Delight, sold off a land
board at the valley lane; drawn for a lot east of the lane — nothing west
of its yard, which is the way out) and the Hinkley house's garden.
⚠ The authored venue farms are NOT holdings yet (see § Deferred seams).

## The way in — keys, doors and the ascent gate

Keys mint at the **chokepoints**, and there are exactly three:
`title buy` (the sale), `lease` (the grant), and the dorm's `provision`.
Each does the same three calls in the same order:

```
Lock.mintKeyway() → ParcelApi.setKeyway(extent) → Lock.issueKey(holder, keyway, 'pin-tumbler')
```

`FrontDoorExit` is the generic threshold: eager on its face with the
entry room's **row** (a real template, so the zone resolves and the exit
reads honestly before anything is minted), `canTraverse` gated on
`opensFor` against the holding's keyway (a sync read off
the warren's cache — an empty keyway admits nobody), and
`computeDestination` faulting the holding in through `admit`.

Locked in, free out: you need the key to get IN, never to get out.

### ⭐ Which way the gate faces — the gate ring

An edge off a road into a holding needs a *direction*, and the obvious
one is wrong. Naming it for the holding (`lot-7`, `unit-3` — what is
stencilled on the stake or screwed to the door) is not a direction at
all: `Location.addExit` refuses a non-cardinal direction between
two rooms of the same zone, so every such edge had to be pushed across a
zone boundary before it was legal, and it never had an inverse.

The plat answers it instead. **A road runs along ONE axis, which leaves
the other six planar compass points free** — which is how a street
actually works: your house is on the north side of the lane. So
`PlatPlan.gateDirectionOfSlot` returns, in order, the **right, left,
ahead-right, ahead-left, behind-right and behind-left** of somebody
walking the road in its heading, less any point a road branching off that
reach already leaves by. A road's heading is authored (`heading:`) and
defaults to the direction it branched in, so a court that leaves its lane
southward runs south rather than turning.

It is a pure function of the plan, which it has to be: a sold lot's gate
is re-derived and re-hung at every standup, so it must be the same answer
on every boot.

Consequences: the gate is **cardinal** (legal inside one zone, so the
boundary stops being load-bearing), it has a **known inverse** (`go
north`, `south` back out), it can never collide with the road's own
onward/back exits (the axis is excluded), and **six frontages per reach
is the ceiling** — five where a road branches off — checked at parse so
an over-subscribed plat fails at install. `title buy` tells the buyer
which side theirs is on, because a cardinal gate is otherwise a guessing
game.

⚠ **The let rung has not moved to this yet.** A landing's unit doors are
still `unit-<n>` and still cross the `seznick-house/location` zone to be
legal. The same ring applies (a corridor is a road with a heading), and
it is the obvious next step — but a landing is not a street and the
`right/left` framing wants a second look before it is copied over.

### The ascent gate

At `title buy` and at `lease`, before the money moves: read the
condition of every residential holding the actor already has
(`ParcelApi.heldUnitsOf`) and refuse below
`residence.ascent.minCondition` (shipped default `0.5`), **naming the
band**. Holding nothing passes; the dorm grant has no gate.

The rung above is earned by keeping the rung you have — which is a
statement about upkeep, not about money.

The read is over **every** unit the actor holds a live use-grant on
(`ParcelApi.heldUnitsOf`), and that decides two things the stewardship
and tenancy designs asked for. ⭐ **A LET unit's condition gates the
buy** — the landlord reference: you do not get the house while the flat
you rent is shabby, and that, not a rule, is why a tenant tends a place
they will leave. And because the gate reads all of them, **holding more
than you can keep is what it refuses** — obligations scale with what is
held, so no ownership cap needs writing; the obligations are the cap.
Money is necessary and not sufficient: coin buys nothing while the
condition read fails. The threshold is one universal setting, not
per-rung. ⚠ A titled house is not a grant — `title buy` transfers title
and mints none — so today an OWNED holding sits outside the read; the
gate is honest about leases and silent about the house you already own.

#### Why the gate is condition and not coin — the lesson it carries

*(Absorbed from `residence-ladder-design-pack` § Part 6 — Pedagogy,
2026-09-21.)* The ladder is where stewardship teaches the most
transferable real-life skill in the pillar: ⭐ **don't buy more than you
can maintain.** *Obligations scale with what you hold → over-holding is
negative-sum* is a personal-finance lesson **lived rather than
lectured** — the single most common real-world money mistake (the
house, car or lifestyle you cannot keep up), rendered as an honest
mechanic rather than a warning. That is why the read is over every held
unit and why coin alone never passes it: the gate is the calculation
*"you can afford the manor — can you keep it above the band?"*, asked by
the world instead of a tutor.

---

## Condition — the weathering clock

⭐ **Moved to the kernel by the paint build** (the house's `Durable`
condition, see [painting.md](./painting.md)): the fabric is the BUILDING's
— the holding's house, or houses — and it wears **only while the
building's coat is gone**, at `structure.weather.daysToWorn` (default 45
game-days from sound to worn, the dial that was
`residence.weather.daysToWorn`). A holding's house starts bare, so an
unpainted holding wears exactly as the shell always did. Reconciled on
read, stamp-forward, persistent; honest across a restart.

⭐⭐ **A holding's buildings are AUTHORED** (harbors review, 2026-11-03 —
the owner, on the paint merge's derived grouping: *magic*). The plan's
owner — the programme row — names them: `buildings:`, a list of building
rows, **the main one first**
— and each is a Building row (class `HoldingBuilding`, `extends:
/system/residence/idea/holding-building` for the plain house's words) with
its own `extent` and `entrance`, like any building. A building's rooms are
the plan's rooms **at or under its extent** — the Structure rule, read
off the paths; nothing reads how the rooms connect, which ones hold their
air, or the order of an `exits:` map. Ground under no building's extent (a
yard, a tract, a garden) is open ground and no building's. A copy mints
one `HoldingBuilding` per named row at the holding's wake, persisted under
the holding's extent (the main one) or `<extent>@<row leaf>`.
`Floorplan.load` refuses loudly a building outside the plan, one with no
rooms, an entrance that is not one of its rooms, a room two buildings
claim, or two buildings with one leaf. A plan that names **none** (a
Seznick flat) mints none: its rooms read the Building row whose extent the
holding lies in (Seznick House), and the location graph attaches no
building a programme names to anything.

| programme | `buildings:` |
|---|---|
| valley farmstead | `farmstead-house` (`location/house`, main) · `farmstead-barn` (`location/barn`); the yard and three tracts are open ground |
| Hinkley house | `house-building` over `house/indoors/` (hall, kitchen, living, bedroom, bathroom); the yard and garden are open ground |
| Duncan Hall dorm | `dorm-building` over the `dormroom` row |
| Seznick flat | none — every unit stands in Seznick House |

A ship's hull is the same fabric on `Ship` (her seams a `Durable` fabric
under her coat; worn, she leaks — see [ship.md](./ship.md)).
(`WeatheringMixin`, harbors' review-3 clock on the warren, retired in the
paint merge: the paint review had already folded the same clock into
`Faced` + `Durable`.)

The readout is **only ever a word** — `Durable`'s, one scale for every
condition in the game (paint review; the five residence bands retired):

`sound` · `worn` · `badly worn` · `broken`

with a cause line that says what happened in words ("the paint has gone;
the weather is at what was under it"). No number, no percentage, no gauge —
`survey`'s suite asserts that no digit reaches the readout.

### ⚠ Two clocks, and they must never touch

A **building** weathers on the passage of days — its coat where the
weather reaches it, and its fabric once the coat is gone. A **good** wears
only when you use it (economy Law 2): its coat weathers only where it is
left out in the weather, and its own condition never does. A regression
test pins a durable good at unchanged condition across a game YEAR that
takes the house from sound to broken. If a durable ever loses condition to the calendar, every
object a player owns rots while they sleep, and the second law of the
economy is gone.

### Terms — who OWES the upkeep

A validated open string on the holding (`HoldingWarren.upkeepTerm`, its `UPKEEP_TERMS`), not a
closed kernel vocabulary — a future `hoa-shell` should be a content
edit:

| Term | Means |
|---|---|
| `institution-all` | the institution keeps it, inside and out (the dorm) |
| `landlord-shell` | the landlord keeps the shell; what you put in it is yours (a let unit) |
| `owner-all` | all of it is yours (a bought house) |

The term says who *owes* the upkeep. It does **not** say who *may*
perform it — `maintain` refuses nobody, because a neighbour who paints
your window frames has painted them, and modelling permission there
would be modelling permission where the world models work.

### The acts

- **`survey`** (platform, perception — afforded actor-side beside
  `look`): the room archetypes with what met and missed each, and at
  home the whole holding plus the shell band, its cause and the term in
  words. Read-only. See [the archetype half](#archetype-satisfaction).
- **`maintain`** (residence pack, crafting — conferred by a carried
  householder's kit): restores the shell to sound and wears the kit. The
  kit is a class rather than a row precisely because **an affordance is a
  static on a class**, so the tool is what makes the verb visible.
- **the `maintains` brain** (the residence pack,
  `/system/residence/behavior/maintains`): the agency that performs a
  term. Katie walks the dorms, Walter the Seznick shell, on a cadence,
  bounded by `batch`, through the LITERAL verb via `forceCommand` — so
  every gate on a typed `maintain` gates the beat. Take the kit off
  Katie and the dorm weathers.
  ⭐ It lives **with its subject** rather than in the kernel: it reads
  holdings, which are the residence system's, and the move deleted the
  kernel's last `class.HoldingWarren` string. It asks
  the location graph's record for the institutions over or inside its
  extent — which matters more here
  than anywhere, because this is the per-NPC, per-cadence beat, and it
  was the site that read the whole world most often.

---

## Capacity — the operator dial

D10's "runtime operator dial" is the shipped `AppSettings` mechanism, and
nothing else. An institution row authors a `capacityKey` and a
`defaultCapacity`; `capacity()` reads `AppApi.setting(capacityKey)` and
falls back to the authored default; `assertBelowCap` refuses at the cap
**with the reason named**. The wizard-gated `config` verb is the
adjustment surface:

```
config set hinkley-hills.lotCap 60
```

Shipped defaults live in the residence pack's own settings contribution
(`content/settings/residence.yaml`): the plat cap, `dorm.roomsPerFloor`,
`mayfield.unitCap`, the weathering rate and the ascent threshold.

Refuse → raise → admit is the whole loop, and it needs no rebuild:
capacity is read at provision time, never baked into a roster.

---

## Archetype satisfaction

`lib/archetype/Archetype` grew a third derived read beside `describe()`
and `materialize()`:

```ts
satisfies(space: Container | Container[]): Satisfaction
```

Per capability slot: satisfied, and **by what**. Evaluated over the union
of one or more spaces' contents AND their fixtures, so a holding answers
as a whole and a lamp on the wall counts as much as one on the floor.

`industry` became optional — a **room** archetype derives nothing,
because no recipe makes a bedroom, and that absence is exactly what
distinguishes it from a venue archetype. Two needs joined the vocabulary:
`rest: n` (a posture-bearing `lie` slot with `restQuality ≥ n`) and
`presence: <keyword>` (deliberately the weakest need available, and the
honest one for a toilet that composes no capability mixin on purpose).

Four room archetypes ship in `generic-objects`: bedroom, kitchen,
bathroom, living.

⚠ **Nothing consumes a satisfaction.** An unrecognized room provisions,
persists and functions identically; no multiplier, gate or price reads
one. A source walk asserts it, because a habit is not a guarantee.

---

## Owned goods in a holding

The furnishing subsystem's owner-based persistence carries straight
across ([furnishing.md](./furnishing.md), [persistence.md](./persistence.md)):
a stamped good persists in its OWNER's estate against a `place`, and a
room going dormant simply forgets furniture it never owned.

Two residences-era additions:

- **the keyed place.** `placeIdOf(host)` is `scope#key` for a keyed
  host, so many rooms sharing one row keep distinct placements — and a
  captured placement records `{ container, containerKey }` so a
  restoring host re-enters through `CirculationWarren.admitFor(key)`. That
  is the "log out in your own yard, log back into the same yard"
  acceptance.
- **the mount marker.** A good hung on a wall (`hang`) records
  `mounted: { slot }` on its estate entry, derived from where it
  actually hangs so it can never disagree with the fixture map; the room
  overlay re-attaches it as a fixture instead of dropping it on the
  floor. `AdornableMixin` contributes a capture pass that reports
  stamped fixtures to `noteOwnedGood` — without it, a room that slept
  while its owner was offline took the lamp with it and nobody captured
  it.

---

## Identity — `templatePath` always resolves to a row

The D17 split, shipped in this build and now an invariant the
`lint:census` gate enforces:

- **`templatePath`** is a clone's LINEAGE and it always names a real
  content row.
- **`identityPath`** is an instance's minted identity — a hard-private
  stamped slot, read through `getIdentityPath()`, defaulting to the
  template path when unstamped.
- The `byTemplatePath` registry indexes on **identity ?? template**, so
  every existing lookup (`findByTemplatePath('/platform/agent/Avatar/<pid>')`)
  is byte-identical, and principal readers (grants, group membership,
  chattel stamps, snapshot owners, the domicile stamp) moved to
  `getIdentityPath()` with their VALUES unchanged — **and a row is
  enumerated by prefix**, so `findAllByTemplatePath(row)` returns the
  row's unstamped clones *and* the instances minted under it. Asking the
  index for an identity when you mean every instance of a row is the
  mistake the two read names now prevent; see
  [persistence.md § The three registry reads](./persistence.md).
- ⭐ A holding room's durable per-instance handle is **row + key**, not a
  minted identity: `restoreOrSeedImpl` takes `scope = getIdentityPath()`
  (the row, since nothing was stamped) and keys the room on
  `<extent>/<leaf>`. Nothing about a holding room is stamped, so nothing
  about it moved in the addressing build; what it gained is a NAME for
  the pair — `getDurableHandle()` → `<row>#<extent/leaf>`.
- ⭐ So **no path answers "which building is this room in"** for a holding
  room: the room asks its warren (`Warren.structureOf`, which
  `HoldingWarren` answers from its buildings — paint review), never a
  persistence key passed off as a place.
- `StuffApi.clone`'s `asTemplatePath` option is **gone**, replaced by
  `asIdentityPath`. The lint proves the channel stays retired and that
  every template-path-valued field in every shipped row resolves.

Rowless minted paths were how the old `PlatWarren` conjured a yard, and
they are why a bought lot's rooms could not be edited, addressed or
resolved to a zone. Keyed instances of real rows replaced them.

---

## The packaging

Four packs were cut for this build (D18), and the membership test is
arcana's — *a capability pack holds what other packs' content names*:

- **`residence`** — the substrate classes and the substrate's own verbs,
  settings and goods: `PlatBook`, `PlatWarren`, `LotGateExit`,
  `HoldingWarren`, `BuildingWarren`, `FrontDoorExit`, `KeyedDoorExit`,
  `DeedDesk`, `HouseholdersKit`, `maintain`.
- **`eternal-university`**, **`terminus`**, **`hinkley-hills`** — the
  three localities, each homed whole (content + its own `src/`).

`world-seed` kept the four remaining localities and the commons.

⚠ **A locality is not a path prefix.** The first cut homed University
Avenue in `eternal-university` because it shared the `/world/terminus/eternal`
path root with Duncan Hall — a fact about the filesystem, not about the
world. The avenue is a **Terminus** street: it runs from the terminal
mouth past the campus wall down to the Counting-Houses, its own prose is
full of travellers off the Terminus trains, and the campus Locality row
has said `_address: terminus/city/campus` all along — the campus is a
district *inside* the city, and the gate is the boundary. It now ships
in `terminus` at `/world/terminus/university-avenue`. When homing
content, read what the room SAYS it is, not where its file happens to
sit.

---

## The decision index — what `(D…)` and `(P…)` mean in the source

⭐ **This table is why the requirements and plan docs could retire.** The
build's source carries **159 citations** of the form `(D17)`, `(P10)` —
they are how a reader gets from a line of code to the argument that put
it there, and retiring the artifacts they name would have turned every
one of them into a dead reference. The titles live here now; the
reasoning lives in the sections of this doc that the titles describe.

`D` = a **requirements** decision (what the build must be true of).
`P` = a **plan** decision (how it was built).

| | requirements decision |
|---|---|
| **D1** | The ladder's three rungs are dorm → apartment → Hinkley house |
| **D2** | The house is minted rooms behind a locked door |
| **D3** | The apartment building is a Warren of units, landlord-owned |
| **D4** | Condition is shell-weathering, reconciled on read |
| **D5** | The obligation is shell upkeep, and WHO owes it is a tenure term |
| **D6** | The ascent gate is a read at the acquisition chokepoints |
| **D7** | Furniture retail at the general store; keys at the chokepoints |
| **D8** | apartment-requirements.md is superseded and retires at sweep |
| **D9** | The apartment building fronts its own residential side street |
| **D10** | Elasticity: singleton institution, on-demand stock, runtime caps |
| **D11** | Furnishing is the domestic face of the economy |
| **D12** | The four-role uniform model; one institution, two tiers |
| **D13** | The plat plan: layout is authored data, and branching ships |
| **D14** | The realtor: one Terminus realty office fronts every plat book |
| **D15** | An archetype is a satisfiable checklist, never a room class |
| **D16** | Warren-of-warrens: the residential programme |
| **D17** | The identity split: `templatePath` always resolves to a row |
| **D18** | Packaging: three locality packs + the `residence` capability pack |

| | plan decision |
|---|---|
| **P1** | D16 convergence depth: the dorm converges FULLY, in two steps, keeping its classes |
| **P2** | D17 mechanics: `identityPath` becomes a stamped slot; `asTemplatePath` is retired, not renamed-in-place |
| **P3** | Capacity dials ride AppSettings keys named by the institution row |
| **P4** | Plan shapes: dorm = linear (floors), Hinkley = branched (the live exercise), static = pinned by a synthetic suite institution |
| **P5** | Archetype satisfaction extends the SHIPPED archetype substrate, not a new one |
| **P6** | The realtor fires the purchase AS THE BUYER via a prompt, not an NPC dispatch |
| **P7** | Names (content is free to re-flavor prose, not paths) |
| **P8** | Where the keyed rooms' rows live (the zone lesson, applied) |
| **P9** | Mounted goods persist as owner-side placements with a `mounted` marker |
| **P10** | Condition, terms, and the maintenance act |
| **P11** | Packaging: the pack cut (D18) and the relocation map |

---

## Cross-references

- [residence.md](./residence.md) — the dorm rung end to end, and the
  multi-instance persistence model this generalizes
- [smallholding.md](./smallholding.md) — the holder half (plat book, lot
  holder, the title gate) and the cultivation half beyond it
- [furnishing.md](./furnishing.md) — owner-based persistence, the estate
  slice, `place`, the room overlay
- [persistence.md](./persistence.md) — the spine, keyed hosts, the
  `mounted` marker
- [parcel.md](./parcel.md) — title, grants, `heldUnitsOf`, keyways
- [credential.md](./credential.md) — keys as credentials
- [archetype coverage](./furnishing.md#archetypes) — what a room is FOR

---

## Deferred seams

Named, not stubbed — each is a clean attach point:

- **inn rooms** — a nightly grant with an expiry, on the same base;
- **remodel** — the dorm's theme overlay generalized to a holding
  (a programme-level prose + fixture personalization);
- **HOA / a third-party shell term** (`hoa-shell`) — the vocabulary is
  already open; what is missing is the body that collects for it;
- **valuation and resale** — a lot sells at the book's authored price
  and nothing values it afterwards; there is no market;
- **rent as a recurring charge** — a lease is a grant with no money leg;
  the contract substrate is where that belongs;
- **a second realty desk anywhere** — already possible (populate a
  `DeedDesk`); nothing has needed one;
- **the branch that a plan authors but no frontage has reached** — the
  court off Hinkley's segment 2 exists in the plan and materializes as
  frontage fills; deeper trees (a court off a court) are untested.

### ⭐ Plans — a kind of building as an ordinary place, on its owner (harbors review 3, 2026-11-03)

A warren that mints copies of a set of rooms copies a **plan**: an
`extent` whose rows are the kind's rooms and an `entrance`, the way in.
⭐⭐ **The plan's fields live on whoever owns the plan** (harbors review,
2026-11-03 — the owner: a blueprint `Structure` row was not a standing
thing, only a specification the warren read, and its building rows then
sat inside it — two layers of Structure rows where only one described
something real). So there is **no blueprint row**, and **a `Structure` is
only ever a thing standing in the world**:

- a **holding** — the programme row (`HoldingWarren`) authors `extent`,
  `entrance`, `footprintM2` and `buildings:` on the kernel
  `FloorplanWarren`'s own fields;
- a **ship of a kind** — her KIND's own `Ship` row names `extent` and
  `entrance`, exactly as the authored Hesper's row does; her
  `ShipWarren` row names nothing, and the warren reads the plan off her
  (`ShipWarren.planOf`, the `FloorplanWarren.planOf` `@hook`);
- a **mine** copies none (its spine is singletons, its workings carved).

`lib/location/Floorplan.ts` (`new Floorplan(owner, fields).load()`) reads
the rows — FOLLOWED, never searched (`lint:template-scans`): the entrance,
then every room a way inside the extent leads to from a room already
found. A room no way reaches is in no copy. A one-room plan (the dorm's)
names its room as both extent and entrance, and its copy is keyed by the
holding's own key with no leaf. ⭐ Its BUILDINGS are named the same way —
`buildings:`, building rows each with its own `extent` and `entrance`, the
main one first (see [§ Condition](#condition--the-weathering-clock)); a
plan derives none. The shipped plans:

| owner row | extent | rooms |
|---|---|---|
| Hinkley `house-programme` | `hinkley-hills/house` | `hinkley-hills/house/*` (the house's own rooms under `house/indoors/`, its building's extent; `living`/`bedroom` extend the generic rows), own zone `house.yaml` |
| valley `farmstead` | `hearts-delight/farmstead` | `hearts-delight/farmstead/{location,tract}/*` (+ `location/house` extends the generic bedroom), own zone `farmstead.yaml` |
| Seznick `unit-programme` | `seznick-house/location` | `seznick-house/location/*` (its zone; + `bathroom` extends the generic row) |
| Duncan Hall `dorm-programme` | the `dormroom` row | the `dormroom` row itself |
| `Ship/smack` (the kind) | `/stuff/location/ship/smack` | its deck, cabin and hold (each room's own `floorM`); `ShipWarren/smack` names nothing |
| `ShipWarren/authored` | **none** — an authored ship's own rows already ARE the place, and they are served; the warren adopts her live rooms | — |

**The copy** (`FloorplanWarren.wake`): every room row is cloned keyed
`(room row, <key>/<leaf>)` with its `exits:` withheld (a way applied by
the clone would stand the plan's own room up), then ONE rewrite: a way
whose destination is inside the extent is installed to the same room of
this copy (its `kind:` honored, `fitExit` the subclass's chance to fit it
— a keyed door gets its holding); a way out is applied as authored.
Mirroring is the copy's: every copied direction is reflected, and the
layout (`layOut`, the entrance at the origin) is read from the copied
ways. ⚠ **A plan's rows author no `coords:`** — a row's cell registers on
its zone's grid, and every copy of a row stands in one zone (every
Hinkley yard would claim one square of the lots' grid).

**Never served.** Nothing boots a plan's rows and no way leads in. A
programme row is a warren, not a structure, so its `extent` attaches
nothing; the location graph also skips a Structure row that mints its
rooms through a `warren:` (a ship kind — her extent is a plan) and every
building a programme names (`LocationGraphRegistry.deriveAttachments`),
so a copy's room under the extent never stands one up or claims it as
its structure. A ship of a kind is told by her warren, never by a path
(`Structure.memberExtent` `@hook` — `''` for a kind, since every smack's
rooms share their rows). The plan's rooms are kinds (non-singleton
classes), so the projection and `lint:location-graph` never see them as
nodes — no `unreachable:` value was needed. An author holding title to
the extent can stand a room of it up by the shipped author path (`clone
<row>`, gated on `AccessApi.canAtPath`, then `goto`) — ⚠ do it where no
copy of that row is live: a plan room's own ways resolve by row
(`StuffApi.singleton`), which hands back a live copy, or refuses when
there are several. Copies are always minted from the ROWS.

### Standing venue holdings (tracts build D32, deferred)

The plan was to re-home the authored venue farms as STANDING holdings of
a static plat (`dormancyOf: standing`, per-slot programmes, authored
placement). It met the load rule: a holding exists only once its plat is
loaded, and the authored exits into those farms resolve their rows on
use — so either the plat is booted (forbidden: load on use, never boot)
or an exit walked first mints a stray room. Doing it right flips the
reference: the venue's tract (or approach room) names its holding, and a
venue holding ADOPTS its authored singleton rooms rather than cloning
keyed copies. `PlatPlan` already parses static slot objects
(`programme`, `gate`, `mirrored`/`mirror`) for it.

### ⭐ The lift landed: `FloorplanWarren` in the kernel (harbors P1)

The second consumer arrived as a **ship**, not a mine: her deck, cabin
and hold are a floorplan and she sleeps whole. Her hull weathers as a
house's fabric does — the owner's rule was *"whatever the rule, it works
the same way as residences"* — but the fabric is the STRUCTURE's, not the
floorplan's (paint D13): a `Durable` condition on the building or the
ship, under its coat, on the `structure.weather.daysToWorn` dial. Of
everything the floorplan owns, **only the upkeep term is residential**
(`FloorplanWarren.getUpkeepTerm`, which a holding answers).

`packages/server/src/mud/lib/location/FloorplanWarren.ts` (abstract,
`Persistable(WarrenMember(Container(RoomWarren)))`) now owns the plan's
fields — `extent`, `entrance`, `footprintM2`, `buildings` (it was a
`floorplan` list until harbors review 3, then a `blueprint` row until
2026-11-03), read through the `planOf` hook — the keyed room mint
(`wake` / `wakeRoom` / `roomForLeaf` / `roomForKey` / `entryRoom`), the
intra-plan wiring, `layOut`, the whole-warren capture (its rooms that
keep a record are its `dependentPersistables`, so one
`PersistableApi.capture(warren)` writes them all — harbors review 2
retired `captureAll`; a ship declares her warren the same way, so saving
her saves her hold) / `teardown` / `revert`, the
tracts build's layout reads (the persistent `mirrorAxis` every copied
direction is read through, *only a persistable room restores*,
`onWarrenWake` after a room is woken, a revert deleting each room's
`dependentPersistables`; a tract's `share` is the tract row's own field
now). It speaks through five hooks: `planOf` (whose fields the plan is —
the warren's own, or a ship kind's row), `reconcile` (who hears a
population change — still abstract), `fitExit` (what a copied way needs
from the copy — a keyed door gets its holding's keyway; the base does
nothing), `addressFor` (the address a woken room is stamped with — a
read, not a wake hook: anything a room does on waking is its own
`WarrenMember.onWarrenWake`, the ONE wake hook since harbors review 2;
a ship's deck stamps itself a shore of her sea there), and
`getUpkeepTerm()` (who owes the upkeep — none by default; a holding
answers its term, and `survey` reads it).

`HoldingWarren extends FloorplanWarren` and keeps only what is
residential: `addressBase` (answered by `addressFor`), the parcel keyway
(`refreshKeyway` before the base's wake; the keyed doors fitted in
`fitExit`), `openGroundM2()` (the PARCEL's workable area less the
footprints), `reconcile` → the
institution, `holdingKey()` (= `warrenKey()`), `entryRowOf`, the tenure
term, and its BUILDINGS (`house()`, `standingBuildings()`,
`buildingOf(room)`, answered as `structureOf` — paint D6/D13; the
buildings its programme names, harbors review 2026-11-03). `maintain` works on the building a room is part of, or the
ship whose room it is, and speaks in that fabric's words — a
householder's kit and putty, a caulker's kit and pitch.

⭐ **Superseded in part by harbors review 3 and the paint merge:** the
shell clock left `FloorplanWarren` (review 3 gave it a `WeatheringMixin`
on the warren; the paint merge retired that for the paint build's model —
the fabric is a STRUCTURE's `Durable` condition, under its coat, on
`structure.weather.daysToWorn`). And the
`floorplan:` list, with `share`/`footprintM2`/`shareOf`/
`getFloorplanExits`/`wireLockedEdge`, is gone for the plan (§ Plans):
ways are the rooms' own `exits:`, a locked edge is a keyed-door `kind:`
fitted by `fitExit`, `share` is the tract's, the footprint the
programme's. ⭐ And `MineWarren` now `extends FloorplanWarren` (review 3,
the base swap its header planned): it copies no plan, its carve mints keyed members through the base's
`mintKeyed`, its cells sit on its zone's grid (`laysOutOwnFrame` false)
and keep their own records; it has no fabric. `CirculationWarren.conditionOf`
reads the holding's main building (`MixinApi.isDurable`); `SurveyController`
reads the structure a room is part of (a `Building` or a `Ship`) and the
term off `FloorplanWarren.getUpkeepTerm`.

## History

- **Harbors (2026-10/11, MR !511).** P1 lifted the floorplan half of
  `HoldingWarren` into the kernel `FloorplanWarren` so a ship's warren
  could share it. Review round 1 then reshaped it: one wake hook
  (`WarrenMember.onWarrenWake`; `onRoomWoken` deleted, `abdae7e98`);
  saving the warren saves the rooms that keep a record
  (`dependentPersistables`; `captureAll` deleted, `e3141e2d3`); a
  kernel `WeatheringMixin` on the warren (`015a246da`) was retired again
  when paint merged (`5cfd1a5db`), the fabric being the building's or
  the ship's `Durable`; the `floorplan:` list became a `blueprint:` row
  (`b00f8d45c`), the buildings were authored rather than derived
  (`8125deeed`), and the blueprint row went, the plan living on its
  owner (`a20704a98`); `MineWarren extends FloorplanWarren`
  (`8b5f04273`).
