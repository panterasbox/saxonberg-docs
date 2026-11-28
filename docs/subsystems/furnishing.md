# Furnishing — the furnishable room, kept by its container

A residence you **furnish with goods you own**, that stay where you put
them — because the room keeps them.

⭐⭐ **The rule (persistence-model, 2026-11-24): a thing is kept by its
CONTAINER.** A kept room's record is everything in it, whoever holds title;
a player's own record is what they *carry*; a good left in a place that
keeps nothing is lost at that place's next restage. Title
([chattel](./chattel.md)) is a claim, never storage. The owner-side scope
this subsystem first introduced — the estate's goods, the chattel `place`,
the skip rule, the room overlay — is **retired** (§ *History* below).

Rides [chattel](./chattel.md) (title, the registry, `ownerOf`),
[persistence](./persistence.md) (the `(scope, key)` spine),
[parcel](./parcel.md), [residence](./residence.md) (the dorm — the simple
rung and every content precedent) and [posture](./posture.md).

---

## Where a good is kept

| a good is… | kept by | after a restart |
|---|---|---|
| carried or worn by a player | the player's own record (their carried tree) | on them |
| in a kept room (a `PersistentLocation` row, a dorm unit, a holding's rooms) | that room's record, whoever holds title | where it was left — on its surface, in its container |
| in a kept container standing anywhere (a chest, a `Locker`, a consignment shelf) | that container's record (keyed by its part name in a plain place — [persistence.md](./persistence.md)) | inside it |
| hung on a kept room's wall | the room's `Adornable` slice, under `hung` (by slot), re-hung at the restore | on the wall |
| set down in a place that keeps nothing (a street, the plaza, a tavern floor) | nothing | gone at the next restage |

`Container.isKept()` answers the question for any container: a host that
composes `PersistableMixin` and has an established record keeps, and
anything else asks its own container, up the chain. A body is the
exception (`Creature.isKept`): a body keeps itself or nothing, and never
by standing in a kept room. **No field and no list** — the classes that
compose `PersistableMixin` are exactly the ones that keep, and
`lint:instanceable` invariant 15 holds that set to containers and bodies.

⭐ **The player is told.** `look` at a place ends *Kept:* or *Not kept:*
with a line saying what that means, `look in <thing>` says the same of its
inside, the Explore card's place section carries a *kept* row
([navigation-card.md](./navigation-card.md)), and the first time you set
your own good down somewhere that keeps nothing, a teach-once line
(`Avatar.teachOnce`, the `hints.kept` setting) says so — once, and once
more on a later game day, then never.

### Goods on the WALL and ON a surface

A good can be *in* a room, *on* its wall, or *on* something in it. A
hung good (`hang` moves an `AdornmentMixin` good onto the room's
`Adornable` fixture map) is captured whole in the room's fixtures slice
under `hung` (by slot) and re-hung at the restore; the authored fixtures
(`_appliedFixtures`) are not, because the row lays them down again. Mount
slots for owned goods are `mounted:<chattelId>`, stable across restarts
and never colliding with the synthetic `fixture:<n>` counter an authored
`adornments:` uses. A good resting on a surface (a cup on the table)
records its placement in the container slice's `ContentPlacement`, so it
comes back on that surface.

`get` is how a hung good comes down, and it is the guarded direction:
your own good comes off freely, anything else needs write authority over
the room — the bar's neon stays on the bar's wall.

### Lease end — the house's storeroom

Moving out used to send every titled good to "storage" (`evictToStorage`),
a place with no presence in the world. Now a letting house names a
`storeroom:` (the office PLACE whose first non-sale `Locker` is the
store), and `LettingWarren.boxGoods` checks the tenant's goods — what
stands in the unit that its rooms' `props:` did not lay down — into that
locker as held goods. The former tenant takes each back with
`rack reclaim` at the office ([retail.md](./retail.md)). A good too big
for the locker (`fitRefusal`) stays and is lost with the unit; the tenant
is told which, and a diagnostic records it. Duncan Hall's lobby has a
storeroom cage; Seznick House has a boxroom.

## ⭐ Cold storage is a property of a SPACE, not an appliance

A cellar is cool because it is underground; a walk-in is cold because a
chiller holds it there. Either way the mechanism is the same and already
shipped: a `Location` composes `AtmosphericMixin`, so it carries an
authorable `_temperature`, and anything carried in drifts toward it
through `ThermalMixin.restamp` → `BiomeApi.resolveTemperatureFor`. **No
class, no appliance, no new field** — the hospitality `cellar` bundle is
simply a room authored at 285 K.

The archetype's `coldStorage` need is satisfied two ways, and ⭐ **cold
is the word that matters on both rungs** (`COLD_K = 283`, ≈ 10 °C):

- **the SPACE** — the venue's own air at or below `COLD_K`: its
  authored `_temperature` first, else the last integrated envelope,
  read through `Atmospheric.getOwnTemperatureK()`. It reads the
  venue's *own* value rather than resolving outward, because the
  checklist is synchronous and *reported, never enforced* — a room
  that is cold only because its biome is cold is not a claim the
  venue gets to make.
- **a HOLDER of things** in it — `Thermal` + `Sealable` +
  **`Container`**, whose *interior* (`getContentsTemperature()`) is at
  or below `COLD_K`. The shipped one is `Icebox`
  ([spatial.md](./spatial.md) · [thermal.md](./thermal.md)): a
  `CoolboxMixin` chest whose interior follows the coldest thing in it
  while the lid is shut. An empty icebox in a warm kitchen is a box,
  not a larder.

⚠⚠ **Both rungs were wrong until 2026-09-28, in opposite directions,
and both silently.** The space rung asked whether the venue was
`ThermalMixin` and **no `Location` is** — a room carries an
`Atmospheric` AIR, not a thermal body — so the rung had literally
never fired, and a 279 K cold store authored for exactly this
satisfied nothing. The holder rung asked only *insulated and
closable*, with **no temperature test at all**: every `Bottle`,
`Flask` and `Thermos` is `Thermal + Sealable`, so a bag of ice on any
floor was "cold storage", and Dave's Bar reported its cold capability
MET on the strength of an EMPTY ice bin at room temperature.

⭐ A capability that answers wrongly is worse than one that is
missing: a missing check is silent, a wrong one is confidently
misleading, and every room report that ever named cold storage was
reporting on insulation rather than on cold. The repair changed what
four archetypes say about rooms that already existed — kitchen,
hospitality, brewhouse and winery were all re-read.

⚠ **They are not the same thing and should not be collapsed.** The
difference is the design:

| | cellar | walk-in |
|---|---|---|
| cold by | earth, passively | a chiller, actively |
| roughly | 285 K (12 °C) | ~275 K (2 °C) |
| running cost | **none** | power |
| failure mode | **none** | dies with the power |
| right for | wine, cask ale, spirits, dry stock | kegs, produce, anything that must be *cold* |

Only the cellar ships. The walk-in is the upgrade a venue buys into, and
it waits for the power meter to be worth having — a running cost and an
outage that spoils what is inside are the whole point of it.

## `ownerOf` — three rungs

```
ownerOf(item) =
  explicit stamp                                  // changed hands
  ?? covering parcel's owner                      // a fixture in a let unit
  ?? authorOf(templatePath)                       // shipped fallback
```

The parcel rung keys on the **template path, not the location**, which is
what makes displacement recoverable: a fixture carried out of a unit stays
titled to the parcel, so it is theft (custody without title), never a
transfer. Only an explicit stamp transfers it.

> ⚠ It is built on `ParcelApi.coveringParcelOf`, **not** `ParcelApi.ownerOf`
> — the latter is *total* (it falls back to the state) and would make the
> author rung unreachable, silently retitling every authored good in the
> world.

`ChattelOwner` widened to mirror `ParcelOwner` so a group-held parcel is
expressible. **Read-side only**: only the `player` arm is ever stamped, so
no stored row gains a group owner.

## Acreage — ground and floors

Two quantities get called *area*, and only one is conserved.

- **Ground** — any child consumes ground whether it is a building or a
  sub-lot, so there is no structure/non-structure distinction — and a
  building's **footprint needs no field**: it *is* that building parcel's
  own `area`.
- **Floor** — `ParcelRecord.storeys`, default 1. `subdivide` conserves
  children against `area × storeys`.

### The space account — the numbers you actually read

`area × storeys` is a **ceiling**, and a ceiling on its own is useless:
nobody plans against a maximum. What matters is how much of it is spoken
for and how much is left, and both are **measurable the moment children
exist**, because every child declares its area at `subdivide`.

`ParcelApi.spaceOf(extent)` returns all four, derived on read, none stored:

| tier | `allocated` | `unallocated` | `utilisation` |
|---|---|---|---|
| a **lot** | its buildings' footprints | the **yard** | **site coverage** |
| a **building** | its units' floor area | **common area** — corridors, cores, stairs | **efficiency** |

The same three numbers read differently by tier, because `area` means
*ground* for a child of a lot and *floor* for a child of a building. Real
buildings land near 0.75–0.85 efficiency; the gap is the circulation you
had to build to reach the units.

A building that has let **100% of its gross floor** reports zero common
area and an efficiency of 1 — visibly absurd. The model does not forbid
it; it **shows** it. Enforcement (a minimum circulation share) would be a
regulation, and regulation belongs to the legislature, not the engine.

`ParcelApi.workableAreaOf` is the `unallocated` half on its own.

Multi-storey is why: a 300 m² footprint at four storeys offers ~1200 m² of
interior, so a rule written against ground area alone refuses apartments on
the second floor. `storeys` rather than a `grossFloorArea` field because
the row **already encodes floors** — `slotOfExtent` parses a trailing
`f<floor>-r<pos>`, built for the dorm.

⚠ Area is **declared at provision, never derived**. Do not sum rooms and do
not read `getSizeScale()` — that is `cellSize²`, a photometric denominator
with exactly one consumer (the vision walk dividing flux into lux), and
deriving tenure from it would make every lighting tweak a title migration.

Unmeasured land is not policed: a parcel with no declared area subdivides
exactly as it did before the fields existed.

> ⚠ **`area` means two different things** depending on the parent it is
> declared against: **ground** for a building on a lot, **floor** for a unit
> in a building. The ceiling switches meaning with the parent, which is why
> one formula serves both — correct today, and ambiguous to read.
>
> Both are addressed by
> development-slate, which makes
> efficiency a **measured** ratio (lettable cells ÷ built cells) rather than
> a deduction, and splits ground conservation from floor conservation. It
> also corrects a third thing: `subdivide`'s ceiling currently makes **every**
> child draw against the parent, when only *productive* children should —
> a circulation sub-parcel consumes lettable capacity it does not produce
> from. That fix needs `landUse`, which is build-2's field and is not on
> master, so it is deferred rather than forgotten.

## The room class and its archetypes

`lib/place/PersistentLocation` is the **one class** every archetype is a
template row over. Its composition is **`DormRoom`'s stack minus
`WarrenMember`**:

```
Persistable → PostRegistration → Exitable → Detailed → Visible
  → Reserved → Staged → Location
```

Every layer is load-bearing and **every omission is silent**: without
`Staged` a seed's `props:` is inert and no fixture ever lands;
without `Visible` its prose is inert; without `Exitable` you cannot walk
in. The shipped dorm room already had exactly this stack, which is why it
is mirrored rather than re-derived.

> **⭐ It is also the class OUTDOOR titled ground uses.** The Hinkley
> Hills build minted a near-twin for a lot's yard
> (`TitledRoom = PersistableMixin(Location)`) and then retired
> it for this one — see [smallholding.md](./smallholding.md). The
> deciding constraint was that a room minted **per lot** cannot be a grid
> member (N lots, one coordinate), and losing `Location` also
> lost the `cellSize²` light denominator that was dividing a yard's open
> sky down to 16.7 lux. So "venue-generic" is stronger than it reads: not
> just bank-bathroom-and-bar-kitchen, but *any* room that is one instance
> among many at its own identity, indoors or out.

### The dorm bed

Sleep-as-logout is the reason a residence is worth having, and the dorm is
the residence every player currently has — so its bed gained a `lie:1`
posture slot and `restQuality: 1.5`. Leaving it a `Placing` prop you
cannot lie on would have shipped the mechanic somewhere nobody could
reach it.

`Placing` is **kept, not replaced**: the two are orthogonal. `Placing`
is what rests ON the bed; the posture slot is who rests IN it.

And it is deliberately the **bottom rung** — a university-issue single
with a thin mattress, against the 2.0 of a bed you chose and paid for. The
ladder should be visible from where you start.

> The retrofit changed **no `class:` path**. The capability landed on the
> class and the slot spec + `restQuality` landed as seed DATA — exactly the
> way `/platform/thing/Chair` works — which is why it was a seed edit rather than a
> migration of every live per-unit record.

> **The dorm room is itself an archetype** — a *bedsit*: bed, desk,
> footlocker, tap, one room. It is not a fifth **kind** of answer; it is the
> other four collapsed into a single room, which is evidence that archetypes
> **compose** rather than partition. It keeps its own class today only
> because `DormRoom` also carries `WarrenMember` and the theme overlay, and
> because its live records make a `class:` change a data migration rather
> than a refactor.

It is **venue-generic**: a bank's bathroom and a bar's kitchen are the
same class with a different `props:`. Home-ness is supplied by the **parcel above
the room** — title, land use, the lease — never by the room.

Four archetypes ship, and they are four *different kinds of answer*:

| archetype | kind | why |
|---|---|---|
| **bedroom** | **function** | sleep is the logout state; the bed is the substrate's first real rest surface |
| **kitchen** | **bundle** | cooking is conferred by heat + pot; the room collapses an errand |
| **bathroom** | **presence** | a residence without one reads broken; what it is *for* is not modelled — **and no waste, as a standing decision**: characters do not excrete; omitting it costs nothing, simulating it costs dignity (the uncanny is asymmetric); the door is closed but labelled — if pedagogy ever wants the excretory system, metabolism already teaches intake honestly (graduated from the bathroom slate, 2026-09) |
| **living** | **audience** | the room you bring a visitor into — and it ships **empty**, because filling it is the point |

They add **zero new classes, mixins or verbs** — a test enumerates the six
shipped classes every fixture uses.

A fifth, **gym** (`generic-objects/archetypes/gym.yaml`, nutrition-and-
fitness), is a **function** archetype like the bedroom: a room is a gym
exactly when something in it is a `load` you can lift or a surface you
can work on — a barbell dropped on a dorm floor makes the corner a gym
with no code and no second archetype, and `analyze rooms` reports it in
any room. Its `load` slot is met by `platform/thing/LoadDevice` (the smith's
barbell); its `pace` slot waits for the first treadmill. See
[exertion.md](./exertion.md).

**Three trade bundles** joined them in libations, each a `PersistentLocation`
row with a `props:` of the trade's own fixtures: hospitality's
**`bar`** (`/trade/hospitality/place/bar` — back-bar, well, the tools,
tap, ice bin, water tap, basin, the glass rack with its pool, seating,
the house tablet) and **`cellar`** (racking, a keg, a cold store);
distilling's **`warehouse`** (`/trade/distilling/place/warehouse` —
racking; each yard adds its own Stock + hand), which the cash-and-carry
and every producer's yard used to spell out in full. ⭐ Since the
template-inheritance build they `extends:` the bundle row instead and
state only their own additions — Dave's Bar is five lines — and `as:`
on a `props:` entry is what lets a child substitute one fixture (the
lounge's own tablet) in place rather than gaining a second.
The lounge's `Bar` keeps its own class (a `SingletonMixin` room the
Warren wires) and lists the bundle's fixtures by reference.

> ✅ **CLOSED by the residences build (2026-08-31).** The provisioner
> shipped: `HoldingWarren.wake()` stands every room of a holding up as
> a keyed instance of its authored row and drives exactly the flow this
> warning describes (`restoreOrSeed` → `seedBornWith` on the no-record
> branch). Hinkley's houses instantiate the generic bedroom, living and
> bathroom rows directly; Seznick House's units author their own bare
> rooms and cite the generic bathroom. The four archetypes are content
> the world now instantiates, and `analyze rooms` reads them back
> ([holding.md](./holding.md)). The warning is kept because the SPINE
> behaviour it documents is unchanged and still catches people: a bare
> `clone` of an archetype row still gives you prose and no fixtures, and
> still correctly so.
>
> ⚠ **(historical) The archetypes need a provisioner, and this build does not ship one.**
> On a persistable host `applyProps` only **retains** the specs;
> `seedBornWith()` lays them down, driven by a provisioning flow
> (`DormWarren.admit`'s shape: clone → key → `hasRecord ? materialize :
> seedBornWith`). A bare `clone` never calls it — and cannot, having no unit
> key to decide with.
>
> Verified in a live world: cloning `/stuff/place/room/bathroom` and walking in
> gives the room's **prose** and **no fixtures at all**. That is correct
> spine behaviour, not a defect — but the consequence is sharper than
> "Wave 6 is deferred": **the four archetypes are content that nothing in
> the world currently instantiates.** The fixtures work individually (a
> cloned tub accepts `lie`, a cloned toilet exposes nothing), and the
> assembly waits on the unit Warren.

Three of the four *conditions* will never earn a
class: spoilage belongs to food and its container, pests to debris plus
food, cleanliness to items and bodies — none of them to the room.

### The LOD ladder

A fixture is modelled to the depth at which the world actually reads it,
and no deeper. The bathroom carries all three rungs at once — toilet
(prose), basin (real water), tub (a real affordance) — and the living
room's television closes it from the far end: `watch` names a streamer and
fills the cockpit embed from anywhere, so a TV object would model nothing.

**One rule: a fixture becomes an object only when something reads it.**

### The kitchen's air reserve

The one authored decision with teeth. A kitchen is the only room where you
deliberately run a fire indoors, so it authors a finite `air` reserve:
`FireLogic` computes `ventilated = isSkyExposed || openNeighbours > 0`, and
a lit range behind a closed door burns **incomplete** — smoke and carbon
monoxide, an un-breathable medium, and the fire self-smothers.

Forgiving by construction: any open neighbour ventilates. Deleting the
block restores `airReserveOf → null` → open air, nothing else touched.

## The posted designation

A room carries `postedAs` — **what the sign on the door says**. Free text,
default `unrestricted`.

**The kernel reads the sign; it never enforces it.** Nothing in movement,
`AccessApi`, an exit or a locomotion check consults it, and a test walks
the source tree to keep it that way.

The reason is the shipped types, not delicacy. To gate entry the engine
must pick an identity axis, and it ships two that are deliberately kept
apart — `SexedMixin` (biology; `xy` yields male/female/intersex,
`monoecious` yields `male-and-female`, `none` yields nothing) and
`GenderedMixin` (pronouns, self-set, default `they`). **Neither partitions
in two.** Any rule would have to answer *what about `they`* and would
compile a position into the engine for every locality forever.

Compliance belongs to the layers that model it, ascending: **norm**
(reactions, regard, renown), **witness**, **law/policy** (the civics
`charter` three-tier resolve), and **wall** — a locked door and a
credential, which is shipped and identity-blind: it asks what you carry,
never who you are.

## Sleep as logout

Metabolism already recovers by `posture × restQuality`, reconciled on read.
The shipped rest model simply **does not stop at disconnect**: log out
lying on a good bed and the next reconcile integrates the elapsed hours at
that bed's multiplier. Parity, never a bonus, and never below the floor.

The one piece of engine work: `PosedMixin` persists the posture, but
`getOccupiedHost()` is a **live scan**, so the bed was gone on restore and
the multiplier read 1.0 on the very reconcile meant to pay out.
`Slotted.occupy` now fires `onSlotOccupied` — the symmetric twin of the
shipped `onSlotReleased` — and `PosedMixin` records `restingOnPath` +
`restingSlot` for **posture-bearing** slots only.

Re-occupancy runs last in `restoreRecord`: after placement, after
fixtures. Every failure degrades to the room floor — standing,
1.0, **no error and no teleport**. Restoring a player must never fail
because their furniture moved.

> **The verb that reaches it did not exist.** `lie` was unreachable by any
> player — the posture views were never contributed and no actor composed
> `SlottableMixin`. Both are fixed (see
> [posture.md](./posture.md)); without them a bed you can occupy is worth
> nothing, because nobody can occupy it. Found by driving the world, not by
> a test — every unit test called `occupyAll` directly and so tested
> the wrong end of the rope.

## Deploying this build

**One row needed deleting on any already-seeded world** (historical — the
pack installer's reconcile has since replaced the seeder). `SeederManager` was
**insert-only** — it inserts a template only where no document exists, and
says so: *"Schema migrations on already-seeded templates are out of scope;
the dev workflow for 'I changed the seed YAML' is `db.domain.deleteOne(
{path}); restart`."*

Everything new here lands by itself: the four archetypes and seven fixtures
are new paths, so they insert on first boot, and **a fresh world is correct
with no action at all.**

The exception is the **dorm bed**, which *edits* an existing seed row —
the first time this build family has changed a shipped fixture rather than
adding one. On a world seeded before this build it keeps the old,
un-lieable row until it is dropped:

```js
db.domain.deleteOne({ path: '/world/terminus/eternal/duncan-hall/dorm-fixtures/bed' })
// then restart; SeederManager re-inserts it with the lie:1 slot + restQuality
```

Verify by querying rather than trusting the file — the boot log's
`0 new templates inserted` is not evidence either way:

```js
db.domain.findOne({ path: '/world/terminus/eternal/duncan-hall/dorm-fixtures/bed' })
  .data.staticSlots   // non-null = the retrofit is live
```

> **Not a job for a content pack.** `PackApi` reconciles (and *adopts*
> existing rows), so it could do this — but standing a pack up for one row
> is ceremony. Packs earn their keep when content changes **routinely**,
> which is plausibly the archetypes' future and is not the bed's past. Move
> content into a pack when the editing is real.

## Deliberately not here

- **Condition / stewardship** — what a room's state *means*, and the
  `restQuality` aggregation from bedding freshness and tidiness. Needs the
  condition model.
- **Room-general cadences** — energy (on the fixture) and air quality (the
  shipped `AtmosphericMixin`). ⭐ Debris and pests SHIPPED with the
  cleanliness build, on the floor rather than a room field: debris is a
  `debris`-form deposit on the floor, and pests are still no field — a
  derived read of debris plus open food (see
  [cleanliness.md](./cleanliness.md)). Rest reads as kept
  (`restQualityAsKept`).
- **Spoilage / preservation** — `ptomaine.yaml` ships a food-poisoning
  condition with no producer, and keeps having none. Spoilage belongs to
  the food and its container.
- **Washing / sterility** — ⭐ shipped with the cleanliness build: a
  dressing's quality is its derived clean tier, and the hands that dress a
  wound seed it as they are kept ([cleanliness.md](./cleanliness.md)).
  `Condition.contagion` ("reserved, no consumer v1") is still a seam.
- **Further dorm "dedupe" — checked, and there is none.** The bed **is**
  done (below). The rest turned out not to be duplication at all, and this
  is recorded so the next reader does not go looking:
  - the **tap** already uses `/platform/thing/UnboundedReceptacle`, the same class the
    bathroom basin uses. Two rows over one class with different prose **is**
    the archetype pattern; collapsing them would delete content;
  - the **desk** is a `Surface` *plus* two things a `Surface` cannot be: the
    **affordance carrier** for `remodel` (a container does not afford its
    own verbs to occupants — a co-located sibling does) and a **theme
    discriminant** (`DormThemes.roleOf` switches on `instanceof`). Same for
    the footlocker, and for the bed;
  - **`DormRoom`** carries `WarrenMember`, the theme overlay and the
    population witness.

  Each earns its class. The only thing shared is the *composition*, and
  that is guarded by test on both sides rather than by collapsing them.
- **The unit floorplan and lease** — provisioning a multi-room leased unit,
  its lease-gated door and its revert. The *rule* ships (lease end boxes
  the tenant's goods into the house's storeroom `Locker` —
  `LettingWarren.boxGoods`, above).
- **Land use and the building zone** — `ParcelRecord.landUse` and
  `landUseOf` are build-2's and are not on master yet; provisioning will
  read them.
- **Real-world parity** — the mirror slate. Its only claim on this
  subsystem is that a room *can carry* state of its own, which it does.

> **The lesson this build paid for**, recorded in
> [antipatterns.md](../antipatterns.md) § *Testing the layer you wrote
> instead of the layer a player reaches*: all four of its real defects lived
> between a passing test and a reachable feature. Ask three questions of
> anything player-facing — is the class composed, is the verb contributed,
> is the content in the world.

## Cross-references

[chattel](./chattel.md) · [persistence](./persistence.md) ·
[parcel](./parcel.md) · [residence](./residence.md) ·
[holding](./holding.md) ·
[posture](./posture.md) · [slot](./slot.md) ·
[metabolism](./metabolism.md) · [fire](./fire.md) ·
[crafting](./crafting.md) · [spatial](./spatial.md)

## History — the owner-side scope retires (persistence-model, 2026-11-24)

`origin/master..HEAD` on `build/persistence-model` (MR !654). Until this
build a good titled to a player persisted with its **owner**: a
`ChattelMixin._place` field (and an indexed `place` on the `chattel` row)
said where the owner kept it; `ContainerMixin.captureSlice` **skipped** any
player-stamped good (`isOwnerPersisted`); a room going dormant reported
what it skipped (`noteOwnedGood`) and `PersistableLogic` flushed it into
the owner's `EstateSlice`; a self-persisting room **overlaid** the goods
whose `place` named it (`ChattelApi.placedIn`) after its fixtures; and
lease end sent every titled good to `storage` (`ChattelApi.evictToStorage`).

It retired (D9) because two persistence scopes overlapped in ways no
player could predict — a good in a friend's kept room was in two records'
reach, naming promoted a good into a third, and a chair on a public lane
was quietly kept by nobody. Now a good is kept by its container, whoever
holds title; `EstateMixin` keeps only the succession facts (`escheatedAt`,
`beneficiary` — [credit.md](./credit.md)); the residency pin moved from the
chattel row to `holder_snapshots.pinned` ([residency.md](./residency.md));
a wall-hung good rides its room's `hung` slice; and lease end boxes into
the storeroom `Locker`. Gone with it: `ChattelApi.placedIn`/`pinned`/
`evictToStorage`, the `_place` field, `Chattel.setChattelPlace`/`isOwnerPersisted`,
`Containable.getKeptWithin`, `SlottedSlice.wornOwned`, `EstateEntry`,
`PersistableApi.captureDetached`/`restoreDetached`, and the `chattel`
collection's `place` and `pin` columns.
