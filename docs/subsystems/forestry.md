# Forestry

The forestry trade: the wood, the stand, the axe and the coppice. Pack
`trade-forestry` (`/trade/forestry`), the first venue the Hanging Wood
above Rejection (rows in the `rejection` pack, no code), the eight wood
materials in the commons. ⭐ **An authored wood is always a wood**:
nothing converts land use at runtime, nothing simulates a tree, nothing
generates a room. The one thing that moves is the stand.

## The four representations of a tree

A tree is the biggest single thing the game has tried to model, and it is
not one object. It is four, and each of a tree's axes lives in exactly one
of them:

| representation | what it is | where the tree's axes live |
|---|---|---|
| **a place** — the stand | a standing cover of trees on a `Wood` room, carried by the ground and drunk from its soil, read by `look`, felled with `fell` | *how much stands here, of what, and how fast it comes back*: `mix[]` — species, wood, seed, `standing`, `capacity`, `incrementPerYear` |
| **a slot-plant** — the standard, the stool | a `Plant` in a `Panel`'s slot: tended, cut, planted, yours | *growth and the cycle*: the shipped growth model (`GrowingMixin`) — stage, vigor, the coppice rotation as a polycarp cycle, `harvestTool`, `discipline`, `standardMaterialPath` |
| **a record** | — rejected here — | a herd is a record filed elsewhere because a herd MOVES; a stand does not, so the place carries it (the `Field`/sward precedent, not the herdbook's) |
| **a prop** | a landmark in a room's `details:` — the big oak the prose is about | scenery by design; never fellable |

What comes DOWN is a fifth thing: the **bole**, the felled trunk as one
loose object on the room floor, which is where bigness lives (below).

## The Wood — a place that is ground

`/trade/forestry/location/Wood` (`trade-forestry/src/location/Wood.ts`):

```
PersistableMixin(StandMixin(SoilMixin(ReservedMixin(SingletonCartesianLocation))))
```

- **`SingletonCartesianLocation`**, not the permissive base: an authored
  clearing is ONE row at a coordinate in its wood's zone, reached by an
  exit — the singleton cell. `PersistentCartesianLocation` states the
  rule for the durable version (*a durable room over the permissive base
  would silently share ONE `holder_snapshots` scope across every mint*);
  the Wood is that class's shape with two mixins inside the outermost
  `Persistable`, which is why it cannot `extend` it. No `WarrenMember`: a
  clearing lives in a zone, not a holding.
- **`SoilMixin(ReservedMixin(…))`** — the ground half, named first as an
  intermediate (inference through nested mixin factories collapses to
  `never`). The DERIVED half of soil only: the seeded `GroundCharacter`
  is `trade-farming`'s and a pack may not import another's `src/`. A
  Wood **installs its reserves from its area at registration**
  (`installWoodReserves`, idempotent — a restored Wood keeps what its
  record says; an authored `reserves:` block wins): moisture
  `areaM2 × 45 L` half full, nitrogen 100 %/60 %. Rain-fed and generous
  — the LIMIT is the increment, not husbandry.
- The three soil hooks as `Field` answers them: `watershedScope()` is
  itself (a place is its own scope); `soilCatchmentAreaM2()` is `areaM2`;
  `soilWaterDemandPerGameDay()` is **the stand's transpiration and only
  that**. A `Panel` standing in the room is its own soil checkpoint —
  *a Wood's soil is drunk by its standards; a Panel's by its stools*.
- Own fields: `woodName` (prose), `areaM2` (default 100 — a 10 m cell).

**How it is minted and restored.** An exit's destination resolves through
`resolveLanding` → `StuffApi.singletonOrClone` → `singleton()`:
**restore when a `holder_snapshots` record exists under the room's scope,
else seed the born-with `props:` (the panel) and capture the first
record.** So on a fresh boot the row's `mix:` hydrates and is captured;
on every later boot the record wins. ⚠ **An authored `mix:` is inert
after the first capture** — the row is what the stand STARTED as, the
record is what it has become. Tune it on a live world and nothing changes
until the DB is dropped, exactly as a `props:` edit never reaches a
booted world.

`lint:locations` needs no roster entry: the gate derives cartesian-ness
from the class file's `extends` chain through the import specifier
`@saxonberg/server/mud/platform/location/SingletonCartesianLocation`
(`Wood.test.ts` pins the specifier textually).

## The stand — `StandMixin`

`trade-forestry/src/lib/Stand.ts` — the standing timber as a cover over a
place, on the sward's shape (`trade-farming`'s `SwardMixin` is the first
instance; this is the second; a kernel `lib/husbandry/Cover` is the seam
when a third appears — two instances is where a pattern is NAMED, not
factored). Host constraint `Stuff & Reserved`; a pack mixin, registered
at pack discovery, nameable by `MixinApi.isActive(room, STAND_MIXIN)` —
no kernel list.

**Unit: whole standards** ("trees' worth"), never m³.

**Derive on read, stamped only by the axe.**

```
standingNow = min(capacity, standing + incrementPerYear × growthFactor × yearsSinceStamp)
growthFactor = clampUnit(soilMoistureFraction() / 0.35)   — null (unmodelled) = 1
```

The moisture factor is the Field's own drought curve for a sward. ⚠ It
is read at derive time, not integrated: a month of drought followed by a
wet day derives the month at the wet day's factor. The honest cheap
form; the sward's stepped integral is the upgrade and the Cover seam is
where it would be shared. **Reads stamp nothing.** `cut(species, nowS,
by)` settles EVERY species to its derived figure, refuses if the chosen
one is below one whole tree, takes one, stamps, logs — so the increment
continues from what is left, and a felled-out clearing is empty for
`capacity / increment` game years at full moisture, longer in drought.
Nothing but the increment and planting puts a tree back.

**The affordance**: `static commandContributions = { self, inventory }`
on the mixin class — on a LOCATION host, *the people standing in it are
its inventory* (`api/command.ts`), so `inventory` is what puts `fell` in a
player's commands. Never `environment`/`peers` (those reach the zone and
the neighbours). `bucketFilenames` unions the chain, so `Wood` declares
none of its own.

**The reading** (`markupAugmenters` on the mixin, rendered into the
room's `look`, synchronous, no memo): per species, in WORDS —
*"Oak stands here — about twenty-four trees' worth, old, planted by
nobody alive. Ash — about four trees' worth."*; then the plantings —
*"An oak sapling, planted by Tam Ferrier on the 4th day of the 2nd
year."*; when every species derives below one — *"Nothing stands here
that is worth the axe — stumps, brash, and the saplings somebody
planted."* Numbers as words (`GrammarApi.inWords`): a stand is a ledger a
player reads, not a gauge. The room's authored `longDescription` is
static and never states a number the stand holds.

**Fields**: `mix[]` (authorable — `speciesPath`, `name`,
`woodMaterialPath`, `seedPath`, `standing`, `capacity`,
`incrementPerYear`), `standStamp`, `plantings[]` (`plantKey`, `name`,
`planter` — an identity path, `planterName`, `speciesPath`, `gameDay`;
idempotent on the key), `cutLog[]`. `standTranspirationPerGameDay()` =
Σ standing × 120 L — the term the soil hook drinks by, and what makes a
full stand on a small cell run its ground dry in a dry month.

## `fell` — one act, and the bole

`trade/forestry/cmd/forestry/fell.yaml` + `FellController extends
ManualBuildController` (the kernel's engaged-act base; the endurance
spend is three lines inline). `FELL_MS = 30 000` game-ms (2.5 real s at
12×), `FELL_COST = 10`; `CROSSCUT_MS = 15 000`, `CROSSCUT_COST = 5`.

**The target is polymorphic, so the view declares `requires: any`** (the
`boil.yaml` shape — declared, gating nothing; `lint:arg-kinds` refuses an
object arg with no `requires`). Three things can stand in the slot and
share no mixin: a `Bole` on the floor (cross-cut), a growing `Plant`
(felled where it stands), or a bare species word that binds NOTHING —
the stand is the ROOM, and a room is not a bindable target, so `fell oak`
lands as `{ stuff: null, raw: 'oak' }` and `raw` is read against the
stand. ⭐ **The word wins when the stand knows it**: after the first
felling the binder matches `oak` to an oak LOG on the floor (materials
match), and the player meant the tree. `verb-gates.test.ts` drives the
real YAML through the binder.

**The axe is an argument, never hunted**: `default:
"reachable:[capability.felling]"`, `requires: [ToolMixin]`; the
controller reads `model.axe?.stuff` — a billhook → `wrong-tool` (*"It
will take a stool off at the ankle, and it will not take an oak."*),
none → `no-axe`. The felling axe row offers `felling`; `lint:capabilities`
wants every kind consumed and `lint:instrument-args` holds the hunt at
zero. The stand is `giver.getContainer()` narrowed by `isActive(room,
STAND_MIXIN)` — the mining trade's `workingOf` shape.

⭐⭐ **Bigness — the bole.** A standard oak is tonnes, and a tree is the
first Thing whose product exceeds a body. Felling drops **one bole** on
the room floor (`/trade/forestry/thing/Bole` — `DetailedMixin(Thing)`
with `lengthsLeft`; mass = the species' wood density × 0.9 m³, oak ≈
675 kg; *can't-budge is emergent from mass, never a flag* — `get bole`
refuses `too-heavy-to-lift`), four logs off the crown beside it
(`Firewood`, the species' wood), and a seed in hand. **Cross-cutting is a
second act on the bole**: `fell bole` takes one length of green timber
(24 kg, the bole's wood) per engagement until nothing is left but the
butt and the brash, and the bole is destructed. The bole **affords its
own cross-cut** (`commandContributions.self`), so a bole dragged to the
yard is still a bole there. The bole on the ground IS the seam
`trade-sawing` attaches to (boards, not lengths) and what the transport
pack's sledge and dray exist to move.

**A planted standard** (`fell sapling`): the product is sized by the
tree — `mature` → the same bole + logs + seed (the wood from the room
stand's entry for that species when the room is a Wood, else the plant's
`standardMaterialPath`); `young`/`established` → one whole carryable
tree (8/30 kg — the Christmas-tree case, the honest inverse of the
bole); `seedling` → `not-yet-a-tree`; a stool → `not-a-standard` (*cut
it with a billhook*).

**Every product is stamped to the feller AND placed** — `stampChattel`
then `followCustody`. ⚠ A stamp alone records no place: `drop`/`put`/
`get` call `followCustody` after every move, and a good minted onto a
floor has been moved by nobody. Without it a bole on the ride was
skipped by the room's capture (a player's good is the owner's to
persist) and never found by the room's overlay (`placedIn` had no row).
Found by restarting the server; with it a bole and its logs survive a
cold restart on the floor where they fell.

Refusals (`controller-rejected`): `no-axe`, `wrong-tool`, `no-stand`,
`no-such-species`, `stand-empty` (*"There is nothing left here that is
worth the axe"*), `not-a-tree`, `not-a-standard`, `not-yet-a-tree`,
`bole-spent`. Credits `silviculture` — `standard` for a standard, `easy`
for a cross-cut.

## The panel — the coppice, and it remembers being cut

`/trade/forestry/thing/Panel` =
`PersistableMixin(SingletonMixin(PostRegistrationMixin(GardenBed)))`.
One row is one panel: a room's `props:` mints a singleton through
`StuffApi.singleton`, the one path that restores-or-seeds a keyless
persistable, so a cut panel comes back cut and its rotation runs across a
restart (the shipped `GardenBed` in a transient room rebuilt full every
boot — a faucet through `restart`). The stools are keyed nested hosts in
its container slice. A class's own `commandContributions` SHADOWS the
composed mixin's, so Cultivable's four views are copied in beside `fell`
(a mature standard in a panel is felled where it stands).

⚠ **A panel ships FULL of earth** (`interiorMaterial` +
`interiorAmount`): a bed row that authors the capacity and no amount
refuses `plant` with *"has no soil in it"* — the shipped coppice panel
did, and nothing could reach it until an acorn existed.

**The ready line** (`markupAugmenters`): *"The stools are ready to cut."*
/ *"Some of the stools are cut to the stool and regrowing; at this rate
they will be ready to cut again in about three hundred and sixty days —
a year, near enough."* The polycarp fill is linear at full satisfaction,
so the estimate is the honest one, in words.

**The rotation is the shipped polycarp cycle, and it is ONE GAME YEAR.**
`hazel-stool.yaml` authors `fruitFillDays: 360`, `fruitSetCount: 8`,
`harvestTool: cutting`, `discipline: silviculture`, and is **ready on a
fresh boot**: `growthStage: mature`, `_vigor: 0.9` (above the 0.8
flowering threshold, so the first re-latch after a cut is immediate),
`_flowering: true`, `_seedSet: true`, `_fruitFill: 1` — every field one
the reconcile could have produced, hydrated because the hydrator applies
every persistent field a row carries. The kernel `harvest` cuts it:
`harvest.yaml`'s `tool` arg (`default: "reachable:[capability.cutting]"`)
and `HarvestController`'s `needs-tool` refusal for a plant that names
one. An unripe stool refuses `nothing-ripe` (AC 4's *"less if cut
sooner"* is read as *refused until ripe; graded by the cycle* — a
partial-yield rule would change every polycarp, and is the slate's).

**The charcoal arithmetic**, stated in each panel row's header and pinned
by `coppice.test.ts` from the rows: six stools × eight lengths a game
year = 48 lengths = six burns (8 a burn) = twelve baskets (2 a burn at a
draught of 0.45) = **six smelts per panel per game year** (2 a smelt);
three panels in the realm make eighteen. *Not a number to raise when
somebody runs out.*

## Coppice with standards — planting, and the deed

Eight slots, six stools: the two free are where an acorn goes. The
kernel `plant <acorn> in <panel>` runs unchanged (the land-use gate
passes — the wood's title is `agricultural`, NOT `wild`, because `wild`
admits no cultivation); the sapling rows
(`thing/plant/{oak,ash}-standard.yaml`) author `harvestTemplatePath:
null` (a standard is felled, not harvested), `discipline: silviculture`,
`standardMaterialPath`, `seedTemplatePath` (a mature standard sets a
seed into its panel — a wood that seeds itself), and
`daysToStage.mature: 5400` — **fifteen game years, stated on the row
and promised nowhere** (450 real days; nothing can observe it yet).

**The deed is written by the ground and told to its room.**
`PlantController` is kernel and must not learn forestry; `Panel.occupy`
does: when a standard arrives in the plant slot inside somebody's command
frame (`ExecutionContextApi.getActingAuthor()` — a restore at boot has
none), the panel asks its container. A Wood records the planting on its
stand (idempotent on the tree's key, because a persistence restore
inside a `go` frame — the room `singleton()`s on first arrival — re-seats
the same tree) and captures; the planter's chronicle takes one deed keyed
on the tree (`recordChronicleOnce('forestry:planting:<key>')` — *planted
an oak sapling in the oak clearing*). In the fuel yard the container is
not a Wood: the deed, and no planting. ⚠ Cultivable's own reseat test
(`candidate.getContainer() === this`) reads true on a real planting too
(contents first, then the slot), so it cannot be the discriminator.

`silviculture` (ISCED-F 0821, `specializes: [agriculture]`) is credited by
`fell`, by `harvest` of a plant whose row says so, and by `plant` of a
seed whose plant says so.

## The wood vocabulary — closed, and minted here

Eight woods in the commons (`base-library/…/material/wood/`), eight tree
species in the pack, each naming the other (`biologicalSource.speciesPath`
↔ the species' `commonNames`); the species' `_defaultMaterialPath` is the
LIVING organism's (plant tissue) — the wood is what is left when it is
felled. `wood-vocabulary.test.ts` walks every `material/wood/` reference
in the whole content tree and every `biologicalSource`.

| wood | density | hardness | toughness | absorb | autoign | heat | cond | the number that makes it so |
|---|---|---|---|---|---|---|---|---|
| oak | 750 | 40 | 60 | 28 | 570 | 16 | 0.17 | dense and hard |
| ash | 690 | 36 | **95** | 30 | 570 | 16 | 0.16 | the toughest common timber — a haft |
| hazel | 620 | 26 | 55 | 32 | 560 | 16 | 0.15 | light, fast, chars well — the coppice |
| beech | 720 | 42 | 65 | 30 | 570 | 17 | 0.17 | dense, hard, splits clean — the best firewood |
| elm | 560 | 30 | 70 | 35 | 570 | 15 | 0.14 | will not split; does not rot wet |
| willow | 420 | 18 | 60 | 40 | 550 | 15 | 0.12 | very light, very tough — the bat |
| pine | 510 | 22 | 30 | 35 | **540** | **18** | 0.12 | soft, resinous — lights first, burns hottest |
| yew | 670 | 38 | **110** | 25 | 580 | 16 | 0.15 | extreme toughness for its weight — the bow |

(kg/m³ · MPa · MJ/m³ · % · K · MJ/kg · W/mK.) `density` sizes a bole and a
length; `hardness`/`toughness` are materials-response; the fire pair is
the fire driver's. Cordwood is a `Crop` of hazel (what `harvest` takes
off a stool, stamped with the maker and the cycle's grade — it used to be
a `Provision`, which hung a freshness clock on a stick); timber is a
`Thing` of the bole's wood; a log is `Firewood`. The `timber-set` recipe
(`category: wood, minGrade: poor`) matches any of them through the
material's `wood` tag — ⚠ but has no by-hand path today (no vessel, no
anvil): `make timber set` answers *"work it by hand first"*, and the mine
buys its sets as it always has. The set-from-timber is the sawing
build's.

## Daylight, and no night

`lux = lumens / cellSize²`; bands `<1` pitch-black · `1–5` very-dim ·
`5–20` dim · `20–60` lit · `60–200` bright. Rejection's surface rows
author their daylight against a 10 m cell (8000 lm open hill, 2500 the
lamp-lit interiors, 800 the adit's portal; the wood 5000/1500/3500 —
dimmer than the hill, its clearings brighter than its ride) and the
Kestrel road's five rows 24000 against 20 m (their shipped 520–700 read
very-dim on an open road). No row authored a value before; every object
read *"something"*. `AmbientLit` is a constant nothing modulates:
**there is no night**, the authored value is the day value, and "at
every hour" is satisfied because every hour is this one. A band word is
what perception does, not what prose says.

## The second instance — a locality authors a wood

Zero code: a `CartesianZone` row (its own cell size, `address:`), rooms
on `/trade/forestry/location/Wood` with a `mix:` each (and any number of
plain rooms), `props:` naming panel rows on `/trade/forestry/thing/Panel`
(one row per panel, shipped full of earth), a `_biomePath`, an
`ambientIntensity` each, cardinal exits, and a title entry whose land use
admits cultivation. The `rejection` pack ships exactly that
(`hanging-wood.yaml` + four rooms + two panels) and no `src/`;
`second-wood.test.ts` hydrates two clearings from literal rows and fells
one. `lint:census` reads a Wood's `mix[]` citations and the saplings'
`standardMaterialPath` (a rowless one is a tree that comes down as
nothing).

**The restart procedure** (AC 8): a felled clearing, its bole and logs,
a planted sapling and a cut panel all survive a cold restart of the
process against the same DB — the stand and the panel from their own
records (`singleton()` restores), the loose goods through their owner's
estate and the room's overlay (`followCustody`). To see an EDITED row
(`mix:`, a panel's `props:`), drop the DB: a template-path move or a
tuned block is a drop, never a migration.

## Deferred seams

- **The Cover seam** — `SwardMixin` + `StandMixin` → a kernel
  `lib/husbandry/Cover` on the third instance (an orchard, a hedgerow, a
  reed bed); the stand's moisture factor integrated stepwise lands there.
- **The seeded site character for a Wood** — `GroundCharacter` is
  farming's; soil.md's *promote on the third non-farming consumer* rule.
- **A per-wood roll-up and `analyze wood`** (the sum over a zone's
  clearings; the increment, the factor, the year each fails) — the
  arithmetic is the player's today, which is the lesson.
- **The engaged-act base** — `MiningActController.engageAct` =
  `ManualBuildController.engageStep` + a spend; the third copy triggers.
- **A by-hand path for a tangible recipe with no vessel** (the timber
  set — decided for `trade-sawing`: a recipe with no build steps is its
  own hand path, so `make` runs craft-resolve directly and the tail
  mints the deed); **sawing, boards, seasoning** → `trade-sawing`, off
  the bole.
- **Partial yield below ripe**; **night**; **canopy as a light model**;
  **rain delayed under canopy**; **fire on the stand**; multi-product
  plants; masting; a reader for `Species.sexDeterminationSystem`; shade;
  browse; form as a stamp; the nursery ladder; hauling the bole; a
  planted standard folding into `mix[].standing` at maturity — each a
  line in `forestry-slate.md`.
- **`ToolMixin.epoch`** — a CLOSED vocabulary (`lib/craft/Epoch.ts`:
  `EPOCHS = prehistory · medieval · industrial · modern · future`, the
  `LAND_USES` shape; the setter refuses a sixth word, so a mis-typed row
  fails at hydrate; `null` = unstated; ordered earliest → latest so
  *"nothing later than X"* is an `indexOf`). The axe and the billhook
  say `medieval`. Knowingly unread: its first reader is the land-use
  covenant's predicate.

## Files

| file | what |
|---|---|
| `packages/content/trade-forestry/src/lib/Stand.ts` | `StandMixin` — the cover; `STAND_MIXIN` |
| `…/src/location/Wood.ts` | the place that is ground |
| `…/src/thing/Panel.ts` | the persistable singleton bed; the ready line; the deed hook |
| `…/src/thing/Bole.ts` | the felled trunk; `lengthsLeft`, `takeLength()`; affords its cut |
| `…/src/idea/cmd/forestry/FellController.ts` | `fell` — the standard, the bole, the planted tree |
| `…/content/trade/forestry/cmd/forestry/fell.yaml` | the view: `target` `requires: any`, `axe` on the felling atom |
| `…/content/trade/forestry/thing/*` | bole, timber, log, felled-tree, cordwood, hazel-stool, felling-axe, billhook, `plant/{oak,ash}-standard`, `seed/{acorn,ash-key}` |
| `…/content/trade/forestry/idea/Discipline/silviculture.yaml` | 0821 |
| `…/content/stuff/idea/species/plantae/…` | the eight trees |
| `…/content/stuff/idea/biome/outdoor/woodland.yaml` | the woodland biome (at the commons path; the base library's biome directory is a pinned demonstrative set) |
| `packages/content/base-library/content/stuff/idea/material/wood/*` | the eight woods |
| `packages/content/rejection/content/world/rejection/hanging-wood{.yaml,/}` | the venue: zone, treeline, ride, oak-clearing, hazel-cant, `thing/panel-{north,west}` |
| `packages/server/src/mud/lib/husbandry/Growing.ts` | `harvestTool`, `discipline` |
| `packages/server/src/mud/platform/thing/Plant.ts` | `standardMaterialPath` |
| `packages/server/src/mud/lib/craft/Tooled.ts` | `epoch` |
| `packages/server/src/mud/api/grammar.ts` | `GrammarApi.inWords` |
| `packages/wire/tests/forestry.dirty.wire.test.ts` | the drive |
