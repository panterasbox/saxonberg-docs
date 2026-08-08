# Furnishing — owner-based persistence and the furnishable room

A residence you **furnish with goods you own**, that stay where you put
them.

Before this build a room persisted its *contents* in its own record, so a
chair you carried into your dorm became, durably, part of the room —
indexed under it, restored by it, gone with it. That was wrong in three
directions at once: it lost the connection between a good and its owner, a
guest's dropped possessions rode a stranger's record, and "move house" was
impossible in principle.

This subsystem introduces the **second persistence scope** — owned chattel
persists with its *owner*, carrying a `place` — and the **furnishable
room** that consumes it.

Rides [chattel](./chattel.md) (title, the registry, `ownerOf`),
[persistence](./persistence.md) (the `(scope, key)` spine),
[parcel](./parcel.md), [residence](./residence.md) (the dorm — the simple
rung and every content precedent) and [posture](./posture.md).

---

## The two scopes

| | host-side (shipped) | owner-side (this build) |
|---|---|---|
| carries | a host's contents, its worn gear, its fields | every good its host holds **title** to |
| slice | `ContainerSlice` / `SlottedSlice` | `EstateSlice` |
| lives on | any `Persistable` | any `Estate` (today: `Avatar`) |
| routes on | containment | **`place`** |

They compose over one spine and never both own the same field. The seam
between them is one predicate, below.

## `place` — where the owner keeps it

A field on the good (`ChattelMixin._place`): a **room identity**,
`'inventory'`, or `'storage'` (the default).

It is deliberately *not* a registry. Ownership is a relation and therefore
lives in one; *where the owner keeps it* is not a relation between two
principals — it is one value, always present, changing on the same acts
that move custody.

**Writes go through one gate.** `ChattelApi.setPlace` is the single write
path, and it writes three things in one call:

1. the good's own `_place` field — what round-trips *with the good*;
2. the `chattel` row's indexed `place` — the **by-room index**;
3. the owner's estate entry, when that owner is live.

> **On the index.** A materializing room has to find the goods that belong
> in it, and their state lives in their *owner's* record — so something must
> be queryable by room. That is the row's `place`. **It is an index, not a
> second source of truth**, and the single gated write path is what keeps
> them from diverging. The duplication is real and is stated here rather
> than hidden.

`ChattelApi.followCustody(item)` re-derives the placement from where a good
now *is* — the one call a custody verb makes after moving something.

## The skip rule

`ContainerMixin.captureSlice` drops two kinds of content:

- a live player avatar (`HasInteractive`) — shipped;
- a good someone has been **stamped** as owning — this build.

Both in **one filter**, because the Container and Slotted slices read a
single content ordering and `PersistableLogic` builds the same index; two
passes would let the worn indices drift.

**It keys on the stamp, not on `ownerOf`.** That is semantically right — a
fixture under a parcel extent is *owned* (the parcel rung, below) but not
*stamped*, so it keeps riding its room's record where it belongs — and it
is also the only version that fits a capture walk that **cannot await**.
`ChattelApi.isStamped` is the synchronous predicate.

## Capture is synchronous, and that shapes everything

`captureSlice` cannot go to the store for goods it can no longer see. The
estate slice therefore serializes the host's **live map**: an entry whose
good is loaded is re-captured fresh (so a plant that grew keeps its
growth); an entry whose good is not loaded is **carried forward verbatim**,
which is the only correct thing to do.

That leaves one case: a good standing in a room that goes dormant **while
its owner is offline**. Nobody is live to re-capture it, and the room's
slice has skipped it — so without a third path it would be captured by
nobody and destroyed with the room, losing the book instead of leaving it
at your friend's.

So the room's capture **reports** what it skipped
(`CaptureContext.noteOwnedGood`), and `PersistableLogic` flushes each into
its owner's estate after the synchronous state build — in memory when the
owner is live, read-modify-write of their stored record when not.

> This is the one place the implementation amends the design. The
> requirements' D4 says "the room captures nothing owner-side on the way
> down"; that is unimplementable as an absolute. What survives is the part
> that matters and is enforced by test: **the host's own record never
> carries somebody else's goods.**

## The room overlay

A host, on materialize, does two things in order:

1. restore its **own** record — its fixtures;
2. **overlay** the owned goods whose `place` names it.

Order is the whole decision: fixtures first, so a placed good can rest on a
surface that exists. Each good is cloned **as its owner**, so provenance
and every principal-based gate resolve to the person whose good it is, in a
room that may belong to somebody else.

The room identity is `PersistableApi.placeIdOf(host)` — the host's scope,
plus its per-instance key **only when that key is explicit**. A keyless
host's stashed key is scope-derived, so folding it in would give one room
two identities either side of its first capture.

## Restore routing

| `place` | on materialize |
|---|---|
| `inventory` | cloned into the owner's own container |
| `storage` | **nothing at all** — the good is live in the registry and the record, with no presence in the world |
| a room identity | deferred to that room's materialize |

Storage is what makes "move house" work: it is the *absence* of a
placement, not a place. `ChattelApi.evictToStorage(prefix)` is the
lease-end sweep — intact, titled, recoverable, **never destructed**.

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

`lib/location/FurnishableRoom` is the **one class** every archetype is a
template row over. Its composition is **`DormRoom`'s stack minus
`WarrenMember`**:

```
Persistable → PostRegistration → Exitable → Detailed → Visible
  → Reserved → Populates → Location
```

Every layer is load-bearing and **every omission is silent**: without
`Populates` a seed's `populates:` is inert and no fixture ever lands;
without `Visible` its prose is inert; without `Exitable` you cannot walk
in. The shipped dorm room already had exactly this stack, which is why it
is mirrored rather than re-derived.

> **⭐ It is also the class OUTDOOR titled ground uses.** The Hinkley
> Hills build minted a near-twin for a lot's yard
> (`TitledRoom = PersistableMixin(CartesianLocation)`) and then retired
> it for this one — see [smallholding.md](./smallholding.md). The
> deciding constraint was that a room minted **per lot** cannot be a grid
> member (N lots, one coordinate), and losing `CartesianLocation` also
> lost the `cellSize²` light denominator that was dividing a yard's open
> sky down to 16.7 lux. So "venue-generic" is stronger than it reads: not
> just bank-bathroom-and-bar-kitchen, but *any* room that is one instance
> among many at its own identity, indoors or out.

### The dorm bed

Sleep-as-logout is the reason a residence is worth having, and the dorm is
the residence every player currently has — so its bed gained a `lie:1`
posture slot and `restQuality: 1.5`. Leaving it a `Surfaced` prop you
cannot lie on would have shipped the mechanic somewhere nobody could
reach it.

`Surfaced` is **kept, not replaced**: the two are orthogonal. `Surfaced`
is what rests ON the bed; the posture slot is who rests IN it.

And it is deliberately the **bottom rung** — a university-issue single
with a thin mattress, against the 2.0 of a bed you chose and paid for. The
ladder should be visible from where you start.

> The retrofit changed **no `class:` path**. The capability landed on the
> class and the slot spec + `restQuality` landed as seed DATA — exactly the
> way `/obj/Chair` works — which is why it was a seed edit rather than a
> migration of every live per-unit record.

> **The dorm room is itself an archetype** — a *bedsit*: bed, desk,
> footlocker, tap, one room. It is not a fifth **kind** of answer; it is the
> other four collapsed into a single room, which is evidence that archetypes
> **compose** rather than partition. It keeps its own class today only
> because `DormRoom` also carries `WarrenMember` and the theme overlay, and
> because its live records make a `class:` change a data migration rather
> than a refactor.

It is **venue-generic**: a bank's bathroom and a bar's kitchen are the
same class with a different `populates:`. Home-ness is supplied by the **parcel above
the room** — title, land use, the lease — never by the room.

Four archetypes ship, and they are four *different kinds of answer*:

| archetype | kind | why |
|---|---|---|
| **bedroom** | **function** | sleep is the logout state; the bed is the substrate's first real rest surface |
| **kitchen** | **bundle** | cooking is conferred by heat + pot; the room collapses an errand |
| **bathroom** | **presence** | a residence without one reads broken; what it is *for* is not modelled |
| **living** | **audience** | the room you bring a visitor into — and it ships **empty**, because filling it is the point |

They add **zero new classes, mixins or verbs** — a test enumerates the six
shipped classes every fixture uses.

> ⚠ **The archetypes need a provisioner, and this build does not ship one.**
> On a persistable host `applyPopulates` only **retains** the specs;
> `seedBornWith()` lays them down, driven by a provisioning flow
> (`DormWarren.admit`'s shape: clone → key → `hasRecord ? materialize :
> seedBornWith`). A bare `clone` never calls it — and cannot, having no unit
> key to decide with.
>
> Verified in a live world: cloning `/obj/room/bathroom` and walking in
> gives the room's **prose** and **no fixtures at all**. That is correct
> spine behaviour, not a defect — but the consequence is sharper than
> "Wave 6 is deferred": **the four archetypes are content that nothing in
> the world currently instantiates.** The fixtures work individually (a
> cloned tub accepts `lie`, a cloned toilet exposes nothing), and the
> assembly waits on the unit Warren. Three of the four will never earn a
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

Re-occupancy runs last in `restoreRecord`: after placement, after fixtures,
after the overlay. Every failure degrades to the room floor — standing,
1.0, **no error and no teleport**. Restoring a player must never fail
because their furniture moved.

> **The verb that reaches it did not exist.** `lie` was unreachable by any
> player — the posture views were never contributed and no actor composed
> `SlottableMixin`. Both are fixed (see
> [posture.md](./posture.md)); without them a bed you can occupy is worth
> nothing, because nobody can occupy it. Found by driving the world, not by
> a test — every unit test called `SlotApi.occupyAll` directly and so tested
> the wrong end of the rope.

## Deploying this build

**One row needs deleting on any already-seeded world.** `SeederManager` is
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
db.domain.deleteOne({ path: '/domain/eternal/duncan-hall/dorm-fixtures/bed' })
// then restart; SeederManager re-inserts it with the lie:1 slot + restQuality
```

Verify by querying rather than trusting the file — the boot log's
`0 new templates inserted` is not evidence either way:

```js
db.domain.findOne({ path: '/domain/eternal/duncan-hall/dorm-fixtures/bed' })
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
- **Room-general cadences** — energy (on the fixture), debris (a
  room-level field), air quality (the shipped `AtmosphericMixin`), and
  **pests, which get no field at all** because they are emergent from
  debris plus food.
- **Spoilage / preservation** — `ptomaine.yaml` ships a food-poisoning
  condition with no producer, and keeps having none. Spoilage belongs to
  the food and its container.
- **Washing / sterility** — `DressingMixin.dressingQuality` (consumed by
  `treat`, no producer) and `Condition.contagion` ("reserved, no consumer
  v1") are the two seams. Handwashing *is* the chain of infection.
- **Further dorm "dedupe" — checked, and there is none.** The bed **is**
  done (below). The rest turned out not to be duplication at all, and this
  is recorded so the next reader does not go looking:
  - the **tap** already uses `/obj/UnboundedReceptacle`, the same class the
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
  its lease-gated door and its revert. The *rule* ships
  (`evictToStorage`); the Warren-shaped content does not.
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
[posture](./posture.md) · [slot](./slot.md) ·
[metabolism](./metabolism.md) · [fire](./fire.md) ·
[crafting](./crafting.md) · [spatial](./spatial.md)
