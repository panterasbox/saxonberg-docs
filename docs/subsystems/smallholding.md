# Smallholding — ground you own, and what grows on it

Living-world **phase 2**. Phase 1 ([husbandry.md](./husbandry.md)) built a
houseplant: one plant, one pot, on someone else's windowsill. This phase
is the same growth model on **ground you hold title to** — a garden bed
with several plants sharing soil, a crop you take off it, and a suburb to
put it in.

Three things arrive that phase 1 had no use for:

| Concern | Lives in |
|---|---|
| **What may be done on a piece of ground** | `lib/parcel/LandUse.ts` — the closed six, on `ParcelRecord` |
| **Ground that holds plants** | `lib/husbandry/Cultivable.ts` — the pot's surface, generalized |
| **The keyed-holder ground pattern** | `PersistableApi.restoreOrSeed` — see [persistence.md](./persistence.md) |

Read [husbandry.md](./husbandry.md) first: the growth model itself is
unchanged except additively, and every claim here rests on it.

---

## The governing reduction: a pot is a bed with one slot

The farming slate specifies a bed as *"a `Slotted` fixture with N slots;
each plant is a `Slottable`"*, and phase 1 built a pot as exactly that at
**N = 1**. `CultivableMixin` is that shared surface, lifted off `PlantPot`
unchanged: the soil read, the plant-slot vocabulary, the
populate-then-adopt applier, the carrier `onMoved` forward, and the soil's
own state.

**N is one authored number**, not a new field and not N named slots — the
`plant` slot's existing `capacity`. A pot says `1`; a bed says `4`. The
shipped pot seeds needed no change to become the degenerate case of the
new thing.

Every generalization in this phase reduces to phase 1's behaviour at
N = 1, and the phase-1 suite passing untouched is how that is known.

### The one structural difference, and where it is NOT

A bed cannot be carried and a pot can. That is **mass**, not class:

> *"Carry / drag / ride / can't-budge is emergent from mass vs. a bearer's
> capacity, never a type flag — you can't pocket a ship is a mass gate,
> not `instanceof`."* — `lib/stuff/Vessel.ts`

`GardenBed` composes the identical stack over `Thing`, weighs 340 kg
filled, and `GetController`'s encumbrance ceiling (≈ the bearer's own body
mass) answers *"it doesn't budge"*. Two consequences fall out for free: an
**empty** bed's frame could be shifted by a strong character, and a bed
stays `Containable` — which it must be, because containment is how a thing
is in a room, and a bed that could not be placed would leave the land-use
gate nothing to gate.

---

## Shared soil, and why density is a real decision

```
rootRoomPerPlant() = soilVolume / max(1, occupiedSlots)
```

At N = 1 that is exactly phase 1's expression. Above it, the **unchanged**
`satRoot` curve does new work for nothing: four plants in one bed each get
a quarter of the soil, so a crowded bed root-limits its plants exactly as a
too-small pot does, with the same cause line.

Dividing by **occupied** rather than capacity is deliberate — an
under-planted bed genuinely gives each plant more, so thinning is a real
choice and crowding is a real cost.

`fitsSlot` asks the question **prospectively** (`rootRoomPerPlant(1)`).
Asking it non-prospectively would wave a fourth plant into a bed sized for
three and only bind it afterwards.

---

## Soil state: the soil has a checkpoint of its own

The phase-2 change that touches phase 1's model. Water moved out of the
plant and into the ground; a fourth limiting factor arrived with it.

A `Cultivable` host owns:

- a **`moisture`** reserve (litres) — what the earth is holding;
- a **`nitrogen`** reserve (percentage points) — **beds only**;
- its own **`soilClockStamp`**, **`reconcileSoil()`** and **reentry
  guard**.

It drains moisture by the **summed** `waterDemandPerGameDay()` of its
occupants × warmth over its own elapsed window, and reports the window's
**mean** for a plant integrating that window whole. The plant only reads,
through three host seams on `GrowingMixin`: `soilMoisture()`,
`meanSoilMoisture()` and `nutrientLevel()`.

The full rationale — including why this is two self-contained checkpoints
rather than one split across two objects, the recursion hazard, and the
three ordering rules (both clocks start together; occupancy changes settle
the window; a re-seat is not a transplant) — is in
[husbandry.md](./husbandry.md) § the moisture callout. It belongs there
because it *reverses* a decision recorded there.

### Nitrogen is the fourth factor, and a pot never sees it

`satNutrient` joins `Math.min(satWater, satLight, satRoot, satNutrient)`
with its own cause line (*"The soil is spent."*). It reads `null` when the
ground declares no nitrogen reserve, and `null` means 1 — **so a
houseplant is never nutrient-limited, because a pot authors none.** That
is the seam that keeps phase 1's behaviour intact while the axis exists.

Harvesting **exports** nitrogen; `feed` puts it back. An unfed bed yields
worse each round, which is the whole loop.

---

## Quality: farming rewards your worst moment

`_worstLimiting` is a **monotone minimum** of the limiting satisfaction
over a plant's whole life, seeded at 1 and updated every reconcile step.
`harvest` maps it to a `GradeBand` through four dials.

It cannot be derived after the fact from the smoothed `_vigor`: a plant
nursed back from a drought **looks perfectly healthy** at harvest and must
still grade badly. Attention is the input, and you cannot buy the band
back at the end.

The crop itself is a plain `Thing` plus **`CraftedMixin`** — crafting
already models "a made thing whose quality is a verdict and whose maker is
recorded", and a harvest is a making. Same five-rung `Grade`, same
`renderVerdict()` prose, same maker's mark as a knife off a bench, and the
maker derives from the execution context rather than being a parameter.
See [crafting.md](./crafting.md).

> A crop carries a **maker**, not an **owner**. `CraftedMixin` stamps who
> grew it; chattel-title stamps who holds it. Once chattel-title lands a
> harvested crop may want both — see [chattel.md](./chattel.md).

---

## Land use — the closed six, and it refuses

`lib/parcel/LandUse.ts`: `residential · agricultural · commercial ·
industrial · civic · wild`, closed the way Module Categories and the
Material library are closed. Each use declares **capability + ceiling** —
how much cultivation it admits (`none · bed · field`) and a permissible
lot-area band.

`ParcelApi.landUseOf(path)` resolves by **longest prefix**: the covering
parcel, then its `parentParcel` chain, then `wild`.

### ⚠ `wild` admits nothing, and that is load-bearing

**Most parcel rows are not ground at all.** `/studio`, `/obj/lounge` and
the `/obj/…` roots are path-branch titles over the template tree, and they
all answer `wild`. Had `wild` admitted a bed, cultivation would be legal
on every branch nobody thought to zone. Stewardship's own gloss agrees —
*"~nothing built; passage and gathering"* is gathering, not farming.

### ⚠ It lives on the parcel row, never on the zone

Land use **gates behaviour**, which makes it access-check data, and
`config/parcels.yaml`'s header is explicit:

> *"ownership is declared HERE (a gated platform seed channel), **never on
> the editable `domain` zone template**… **access-check data lives only in
> this collection.**"*

On an editable zone template a content author could rezone their own land
— the precise forgery the retired `data.ownerGroupName` stamps were
removed to close.

### ⚠ Unparcelled ground is NOT policed — the hermit test

`landUseOf` is total and answers `wild` for ground nobody claims, but the
gate asks whether a parcel **covers** the ground before it asks what that
parcel permits.

> *A hermit in a forest. A shack, maybe a garden. Nothing parcelled.*
> **That works, with zero numbers.**

"Nobody has zoned this" is not the same statement as "this is zoned
against you", and conflating them turns every unclaimed acre in the world
into red tape. The abstract branches stay protected anyway, because they
DO carry rows: `/studio` and `/obj/lounge` are covered, declare no use,
and so still answer `wild` → refused.

Same principle as the acreage check degrading on unmeasured land:
**measure nothing, police nothing.** The cap is there to constrain player
expansion on land somebody has surveyed, not to make authoring a garden
require a survey.

### The land draw — what makes production scarce

```
draw = Σ over productive objects of landRequirementM2
```

Authored on the **bed**, not the zone and not a per-parcel declaration.
A cell count is not expressive (the only lever is how many cells, and a
barn inside a field zone would draw against farming); a declared per-zone
number is not honest (an author could claim a thousand-cell estate draws
1 m² and no player could tell). An authored constant on the productive
object is both, and it is backed by things a player can walk up to and
count.

**Only productive things draw.** Paths, farmhouses, barns, yards and
decoration are free — the distinction was never spatial, it is *does this
use produce?* A pot draws 0, which is the default: a houseplant is
furniture.

> ⚠ **Over-draw is permitted and there is NO penalty mechanic.** A hard
> cap is dishonest (real land does not refuse) and a soft cap is worse (an
> administered multiplier pretending to be physics). Crowding is
> competition for light, water and nutrients, so it belongs to the
> limiting-factor minimum and nowhere else. Do not reimplement it as a
> yield multiplier; there is a test asserting the draw is inert to growth.

### The gate applies to GROUND, not to furniture

`Cultivable.fixedGround` (authored data, not a class check) decides.
A garden bed is ground: what may be grown in it is the parcel's business,
and a bed on the Registry's civic floor is refused with the use named. A
pot is furniture — a houseplant on a windowsill in a rented office is not
agriculture, and gating it would make every commercial room in the game
unfurnishable.

### Area is DECLARED, never derived

`ParcelRecord.area: number` (m², `0` = undeclared), set at provision.
`subdivide` refuses a child outside its effective use's band — measured
against ground area, never `area × storeys`. It is a plain scalar because
the furnishing build's `ParcelApi.spaceOf` does arithmetic on it; the
`Quantity<'m²'>` display banding still exists and wraps it at read time
(`Quantity.of(area, 'm²').tag('lot')` → `"a quarter-acre lot"`).

**Do not derive it from room geometry.** `Location.getSizeScale()` is m²
too, but it is a **photometric denominator** (flux ÷ area → lux) with one
consumer. Deriving from it would make placeholder rooms load-bearing *and*
promote a lighting constant into a land-tenure fact, so every future
lighting tweak became a title migration. A structure's draw on its parcel
is its **authored blueprint footprint**, not a sum over its rooms.

The band is a **lot** band, checked only on a subdivided child. A
24-hectare `residential` district is not a contradiction with a 2-hectare
residential lot ceiling — a district is what lots are subdivided *out of*.

---

## The keyed-holder ground pattern

`PersistableApi.restoreOrSeed(host, key)` — key the host, then either
restore its `(scope, key)` record or lay down its born-with fixtures and
capture them. Returns `true` on restore, `false` on a fresh seed.

`DormWarren.admit` and `LotHolder` are its two consumers, and the
extraction earns itself on the branch it gets right: hand-rolled, the six
lines invite capturing on the restore path, re-seeding a room that already
has contents, or skipping the key stash so the next keyless capture writes
a second record.

`LotHolder` keys on the **parcel extent**, so title and durable state share
one identity — sell the lot and the garden goes with it, because there is
nothing else it could do. It is content, so it is **not** boundary-exempt,
and should not be (`DormWarren` likewise is not).

### Two objects: the catalogue and the provisioner

Selling subdivided ground crosses localities, so both classes are
general (`/obj/`) and each subdivision seeds **instances** — the way
`/obj/Plant` is the class and `/obj/plant/carrot` is a carrot.
`SingletonMixin` is one-instance-per-templatePath, so one of each per
subdivision is exactly what composing it means.

| | `PlatBook` | `LotHolder` |
|---|---|---|
| answers | *what is for sale, on what terms* | *how titled ground becomes a place* |
| data | `label`, `parentExtent`, `lots`, `priceMinor`, `areaM2`, `landUse`, `holderPath` | `roomTemplate` |
| grows | **outward** — terms, demand pricing, auctions, as land becomes a market | **inward**, and is the piece most likely to be replaced wholesale |

**They are split because the second one is going to be swapped.** The
current provisioning model clones one shared room template per lot and
tells the clones apart with a persistence key; the likely successor mints
a template *per residence*, at which point the template path becomes the
identity and no key is needed at all. Separated, that is **a `LotHolder`
subclass and a one-line edit to a book's `holderPath`** — nothing in the
catalogue, the `title` verb or the parcel layer moves. Fused, the same
change would reach into the object that also owns pricing.
`LotHolder.provision` is the `@hook` that swap overrides, and there is a
test that swaps it.

A second subdivision anywhere in the world is therefore **a seed file and
no code**: `title` enumerates books (`world:[class.PlatBook]`, the
system-mode MQL shape) and reads everything off whichever one sells the
named lot. It knows no locality. The only name hardcoded in the verb is
the Registry ROOM, and that is a fact about Terminus's institutions
rather than about any subdivision.

See [persistence.md](./persistence.md) and [residence.md](./residence.md).

---

### ⭐ A lot's room gets an IDENTITY, not a copy

`LotHolder` mints each room at **`<lotExtent>/<leaf>`** through
`StuffApi.clone`'s `asTemplatePath` channel — the identity doctrine's
*minted instance with a scheme-derived key*. The shared template is the
SOURCE; lot 2's yard is its own place.

Sharing one templatePath across N lots broke two things at once, and
minting fixes both with no special case:

| | shared template | minted identity |
|---|---|---|
| land use | resolved to the **district** — right only because every Hinkley lot is zoned alike | resolves per lot, from the path alone |
| avatar placement | recorded the shared template — log out in your yard, log back into a fresh one | exact; the dorm needs a Warren for this, an identity gets it free |

Title and durable state still share one identity, because the mint is
derived FROM the parcel extent. Sell the lot and the garden goes with
it.

### ⭐ A lot's room is NOT on the street's grid

This is the consequence of minting, and it took a live browser drive to
find: **the room class cannot be a `CartesianLocation`.**

N lots cannot share one coordinate, so a per-lot room was never a grid
member — and the grid says so itself. `CartesianLocation.addExit`
refuses a **non-cardinal** direction between two rooms in the same zone,
which is exactly the `lot-1` gate off the lane (see *the gate*, below).
The rule is right; the room was wrong.

So a lot's room is a **`FurnishableRoom`** — the furnishing build's
venue-generic persistable room ([furnishing.md](./furnishing.md)) — and
the class this build originally minted for the job (`TitledRoom`,
`PersistableMixin(CartesianLocation)`) is **retired**. Everything the
yard needs is already there: it is a persistence host, it composes
`Populates`, it is not singleton-shaped, and it authors no `air`
reserve, which is what "outdoors" means to `FireLogic`.

Losing `CartesianLocation` loses `getSizeScale()` = the zone's
`cellSize²` light denominator, and that turned out to be a **fix, not a
cost**. Hinkley's zone has `cellSize: 6`, so a cartesian yard divided
its 600-flux open sky by 36 m² and read **16.7 lux** — under the 120 lux
a carrot needs to be light-unlimited. `Location`'s flat 1.0 gives the
600 lux the seed's own comment always claimed. A yard is not a 6 m box.

The lots live in **their own zone**, `<district>/lots`, authored as one
`CartesianZone` template (`PlatBook.lotBranch`). It must be a *spatial*
zone, not a `FolderZone`: `ZoneApi.resolveZoneForPath` returns only
`SpatialZone`s and walks straight past folders, so a folder here would
put every yard back in the district's zone — the failure it exists to
fix. One authored row covers every lot, which matters because a minted
identity has **no row in `domain`** and `resolveZoneForPath` walks
template ancestry in Mongo: a per-lot zone could not have been minted.

### ⭐ The gate — how you actually reach ground you bought

The lane authored `north → <the yard template>` and that was wrong three
ways, none of which a unit test could see:

1. It named the **shared source template**, so walking north
   instantiated the template itself as a place — an unowned yard, on
   nobody's lot, that anyone could dig.
2. That live instance then collided with the minted identities at the
   singleton guard.
3. It was never expressible anyway. A subdivision has N lots and N
   rooms; no one static exit can mean "yours".

So the street's exits are **installed by the provisioner**:
`LotGateExit`, a `DeferredDestinationExit` (the `DormDoor` shape) whose
`computeDestination` calls `LotHolder.provision(lotExtent)`. Hung as
lots sell, and re-hung at boot from the title registry
(`LotHolder.postRegister` → `ParcelApi.childParcelsOf`) so an owner can
still get home after a restart. Deferred, so a lane with five sold lots
materializes no yards until somebody walks in.

**Direction is the lot's leaf** — `lot-1`, which is what is stencilled
on the stake. `north` collides the moment a second lot sells, and no
compass scheme survives an arbitrary plat. It is non-cardinal, which is
what forces the separate lots zone above. `go lot-1`; bare `lot-1` is
cardinal-only sugar.

**Ungated, deliberately.** A yard behind a house is not locked from the
lane; the house is. A gate that refused non-owners would put a property
boundary where the fiction has a fence, and trespass is the policing
layer's question. The dorm's key gate exists because a bedsit door
genuinely is locked; when these lots grow interiors, *those* get locks.

This is the *minted template per residence* model arriving early — the
engine already had the channel.

## `title` — the act the property build lacked

Every piece of title machinery shipped with the property build and none of
it had a verb. `title` / `title list` / `title buy <lot>`, in **civics**,
next to `government`.

Not `buy`: retail hands over an item off a `PricedOffer` and
chattel-stamps it, and land is real property on a different registry. And
not a verb conferred by the Registry counter — the shipped rule is that a
commerce object affords only its **commerce** verbs.

**The order is the design:** at the Registry → find the plat book that
sells the named lot → funds check → money through banking's settle chokepoint →
`subdivide` (stamping the use and area **the book declares**, where
zoning refuses a bad lot) → `transfer` → the book's provisioner stands
the ground up. The money
moves before the row is written, so an unfunded buyer changes nothing at
all — no parcel row, no chain-of-title event, no yard. A half-completed
land sale is worse than a refused one.

Two steps rather than one for the mint, deliberately: the chain of title
should read *subdivided, then transferred*, because that is what happened.

Sandbox-safe for free — `subdivide` and `transfer` both carry
`assertFieldMutation`, and `parcels` is REFUSE in `COLLECTION_POLICIES`. A
title minted in a holodeck would be a real title.

---

## Hinkley Hills

A Locality at `terminus/hinkley-hills` — **beside** the city, not inside
it, which is the geographic argument in one address. Terminus is dense and
has no room for a garden; the Hills are what happens at its edge.

It **governs itself**. That is the Tiebout point made concrete: a second
polity a short walk away, with different rules, so leaving is a real option
long before anyone writes a law worth leaving over. The Improvement
District is the thinnest government in the game — no departments, no
treasury, no seats, because it has none of those things, and writing empty
structure in would be pretending. (Which is also why the city's records
office takes the payment for a lot: a paper government's finances look
exactly like that.)

Three rooms: the stop, the lane, the yard. The **unbuilt neighbouring
lots** are `details:` prose on the lane, not nine empty rooms — their
emptiness is the story, and prose tells it better at a hundredth the cost.
The **house is prose too**, deliberately: the residence build is furnishing
real interiors in parallel, and a described house can be upgraded later
instead of rebuilt.

The zone runs **6 m cells** against the city's 3 — open ground with room
between things, which is what makes a room out here read as a yard.

---


### Known issue: the pre-sold lot cannot be re-driven

`e2e/tests/hinkley.spec.ts:163` ("WALK IN and WORK IT") is red in-suite
and green alone, and the cause is content shape rather than test code.

Lot-1 is **pre-sold**, so its yard is provisioned exactly once and never
reset. Every run that plants leaves a crop behind, the bed's slots fill
permanently, and the soil bulk sits at capacity — so the drive's
behaviour depends on the age of the database instead of on the code. A
crop in a bed slot is also not targetable by keyword (`destruct carrots`
→ "no match"), so the spec cannot tidy up after itself either.

Two real bugs in that drive were fixed (it never poured the soil it
cloned the sack for, so nothing planted and there was nothing for
`water` to water). What remains needs a **content** decision, not a test
edit: give the drive a lot whose yard it may provision fresh, or make a
bed resettable. Loosening the assertion was tried and reverted — a
weaker test that is still red is strictly worse.


## The cultivation verbs are EMBODIED acts

`plant`, `water`, `feed`, `harvest` and `repot` all carry
`requiresEmbodied` alongside `requiresAnimate`. Pulling a carrot and
turning compost into a bed are acts on **matter**, and a shade has no
hands for either — see [mortality.md](./mortality.md).

That is not a free-floating stylistic choice: `cmd/__tests__/
embodied-tagging.test.ts` **enforces** it over every verb in the
material categories, and it caught `harvest` and `feed` untagged at
merge time. The two builds were developed in parallel, so neither side's
suite could see the gap until they were in one tree — a reminder that a
build's own green suite says nothing about the verbs it added meeting a
rule that landed while it was in flight.

## Deferred seams

- **Nothing fills the bed's moisture from the sky.** Rain and real
  evapotranspiration are phase 4's; the reserve is now a *working* one
  rather than an inert seam, which is strictly better for it.
- **`landUse` answers `field`** and nothing implements a field. Phase 4.
- **`_worstLimiting` is the quality substrate.** Phase 4's per-stage
  sensitivities (drought at flowering costs *count*; at filling costs
  *size*) refine it.
- **The allowance cascade** stays inert.
- **No residence ladder and no ascent gate** — stewardship owns them, and
  both this build and the parallel apartment build decline them.
- **Perennials.** A harvest ends the plant; re-fruiting is a later
  question.

---

## ⚠ Buying a lot is not reachable at char-gen — and that is the economy

A lot is **4000**; char-gen's onboarding stipend is **20**. Land is
earned, and `issueCash` at char-gen was for a long time the only
conserved way money entered a character at all.

The Hinkley e2e briefly closed that gap with a **test-only cash faucet in
the backend** (`POST /auth/test-fund`, 126 lines across `Application`,
`Backend` and `TestAuthRoutes`, handing out 20,000). It was wrong and it
is gone — see [antipatterns.md § A test-only capability added to the
BACKEND](../antipatterns.md).

**What the suite does instead is what the fiction says.** Exactly one
authority may add to the money supply: the **Governor of the Central
Bank**. So the suite becomes that authority and draws the money the way
the Governor does — `reserve issue <amount>`, behind the shipped
`requiresGovernor`, over the conserved `issueCash` faucet. The
`⭐ BUY A LOT` spec asserts both directions: shut to an ordinary
character, open to the Governor.

It gets the authority through the **shipped deploy contract**, not a
seam: `FOUNDER_GOOGLE_EMAIL` names the founder, `OfficeRegistry` reads it
at boot exactly as in production, and the test-auth seam mints profiles
at `<handle>@e2e.local` — so pointing one at the other makes a session a
real founder, holding the founder-default seats. Nothing learns it is a
test. ⚠ That is **interim**: it grants every founder-default seat when
the spec wants one, and it is that way only because
[`office assign` cannot currently seat anybody](./governance.md).

`eval` is not an alternative: its sandbox exposes five names, and adding
`BankingApi` to reach around a gate in a test adds it for every wizard in
production.

**One lot also ships already sold** (`config/parcels.yaml`, held by the
developer group), which is what lets the walk-in-and-plant specs work
ground without buying it. It is better content besides — the lane says
*"Where a lot has been taken there is a gate in the fence and, behind it,
somebody's roof"*, a sentence that described nothing on a fresh world.

### ⚠ A crop in a bed's slot cannot be targeted

`destruct carrots` → *"no match for carrots"*, standing in the yard, with
the crop plainly listed. `look at carrot` fails the same way. The plant
occupies the bed's plant **slot**, and slot occupants appear not to be
reachable by keyword.

Two consequences, one small and one not. The small one: an e2e cannot
tidy up after planting, which is why the walk-in spec asserts verb
**dispatch** rather than outcome. The larger one: a player cannot
examine, name or interact with the thing growing in their own bed except
through the verbs that take the *bed* as their target — which is a real
gap in the cultivation surface, not a test problem.

## History — what the live drive changed

The build shipped green on ~1700 unit tests and was **unreachable**. Four
defects lived entirely between "a passing test" and "a feature a player
can get to", and every one of them is downstream of a single decision:
minting a room per lot (`b0cf1df4`; the merge that preceded it is
`725b2629`).

| what the suite saw | what a browser saw |
|---|---|
| the lane had an exit north | it pointed at the **shared yard template**, which stood the template up as an unowned yard on nobody's lot — and then collided with the minted identities |
| `TitledRoom` composed correctly | a per-lot room cannot be a `CartesianLocation` at all, and as one the yard read **16.7 lux** — under a carrot's light floor |
| exits were added | a non-cardinal `lot-1` gate needs a *spatial, authored* zone the lots do not otherwise have |
| `water` had a controller and a test | nothing in the yard **conferred** it — the can was missing |

Three lessons worth keeping, none of them about farming:

1. **A verb with a passing controller test can still be untypable.** This
   build hit that twice (five verbs in `3fe79102`, `water` here). The
   affordance chain — contributed → in scope → parseable → conferred by
   something present — is not exercised by a controller test.
2. **Retiring a one-week-old class is cheap; keeping it is not.**
   `TitledRoom` existed for days and was already a near-twin of the
   furnishing build's `FurnishableRoom`. The geometry argument that
   justified it turned out to be the argument *against* it.
3. **The engine's refusals were right every time.** The singleton guard,
   the cardinal-exit rule and the spatial-zone resolver each blocked
   something that genuinely should not work. None of them needed a
   change; the content did.

---

## Cross-references

- [husbandry.md](./husbandry.md) — the growth model, and the moisture
  reversal in full
- [parcel.md](./parcel.md) — the title registry this builds on
- [persistence.md](./persistence.md) — `restoreOrSeed` and the spine
- [address.md](./address.md) — the Locality tier
- [civics.md](./civics.md) — governments and jurisdiction
- [crafting.md](./crafting.md) — `CraftedMixin`, `Grade`, the maker's mark
- [banking.md](./banking.md) — the settle chokepoint the sale rides
