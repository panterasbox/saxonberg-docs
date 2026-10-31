# Ceramics — clay, the brickmaker and the potter

> Built by the ceramics & moulding build (2026-10, `reqs/ceramics`). The
> mechanism a firing runs on is the kernel's and is documented where it
> lives — [thermal.md](./thermal.md) § Kinetics (firing is ONE rule),
> [faces.md](./faces.md) (coats and glazes), [materials-response.md](./materials-response.md)
> (brittle bodies, flaws, `ring`). This doc is the TRADE: what clay is, how
> it is won, weathered, tempered, shaped and fired, and who does it.

Owners: `trade-ceramics` (root `/trade/ceramics`) — the trade's instruments,
readings, Disciplines and dials; `trade-quarrying` owns the `Clay` CLASS
(`dig` mints it); base-library owns the commons a clay is made of and fired
into (`mineral/clay-mineral`, `earth/clay`, `ceramic/*`, `glaze/*` — R4,
commons name only commons). Venues are rows in their realm packs.

## ⭐⭐ Clay is a composition (M3)

A clay MATERIAL row tabulates what a kind of clay is made of — the common
subsoil clay (`earth/clay`) is clay mineral 0.55, sand 0.30, iron (goethite)
0.06, lime (limestone) 0.03, organic (peat) 0.06; the Delight's alluvial clay
is fatter (0.72 clay mineral, 0.12 sand); Rejection's fireclay, the coal's
seat-earth, is nearly all clay mineral with almost no iron. A lump's own
`alloying` (`CompositionMixin`) records what was kneaded in after.

`/trade/quarrying/thing/Clay` = `CompositionMixin(MoistureMixin(Lump))`
reads everything off the effective composition, sorting constituents by
their materials' TAGS (`clay-mineral`, `silica`/`grog`/`breeze` = openers,
`iron`, `carbonate`, `organic`) — so a second clay of any realm reads with
no code:

| read | from |
|---|---|
| `plasticity()` | the clay-mineral share, raised up to a third by weathering |
| `crackRisk()` | `plasticity × 1.6 − openers × 1.2`, clamped — fat clay cracks drying; sand opens it |
| `firedColor()` | iron ≥ 4 % red · ≥ 1.5 % buff · else white |
| `refractoriness()` | iron + lime + organic < 5 % and clay mineral ≥ 60 % → a fireclay |

The fired body itself is NOT read here: firing is the material's heat
transitions (`earth/clay` → terracotta, earthenware, stoneware; fireclay →
refractory), the one rule.

## ⭐ Clay has a year — weathering by the freeze-thaw count (S2)

Dug in autumn and left in a heap OUTSIDE, a winter breaks a clay up: every
day its place's air crosses freezing, the water in it freezes and thaws and
opens the plates. `Clay.weatheredDays` (persistent) is advanced on read by
the heap's place's `freezeThawDaysSince(stamp)` — whole days only, the stamp
day-aligned — up to `ceramics.weathering.ceilingDays` (40); a place that
holds its own air counts 0, so a heap kept in a shed stays raw. Weathering
raises plasticity. `look` says *fresh-dug and raw, unweathered* · *crumbling
at the edges* · *frost-broken and mellow*. Two heaps pooled mass-weight
their weathering and their temper (`onMerged`, the `Ore` rule); a lump cut
off a heap is the heap's clay.

## ⭐ Tempering is mixing — the temper pit (S3, R1)

`/trade/ceramics/thing/TemperPit` is the kernel's `CompoundingMixin` dug into the
yard (`Thing`, fixed): put clay and sand (grog, breeze) in, `knead pit`, and
the kernel's blend folds them into ONE clay by mass — the pit's row names the
clay's own row and material as the blend, so tempered clay is still `Clay`,
leaner by exactly the sand that went in, its weathering carried by mass. No
recipe and no ratio. What the pit adds is the WATER: it must hold some (a dry
pit refuses *it wants water*), the tread takes up
`ceramics.temper.waterLPerKg` per kilogram, the clay comes out plastic, and
the pit's water is never a constituent. `ceramics.temper.minutes` of both
hands; credits the pit row's `discipline` (brickmaking).

⭐ **It reclaims (B99).** Green ware is still clay, so a slumped or
frost-burst piece put back in the pit with water is trodden into clay again
(the ruined piece's own refusal says so). The pit takes only what treading
can work in: plastic clay and loose granular openers (sand, grog, breeze).
A fired pot, a stone or a tool is refused by name (`not-treadable`); fired
ware broken to sherds goes to the millstone and comes back as grog. ⚠ "One
piece that already IS the product" (nothing to work in) now compares the
piece's KIND as well as its material: a lump of the pit's clay alone is
refused, a jug made of that clay is the reclaim.

## `analyze clay` (the reading)

`/trade/ceramics/idea/reading/clay` (`ClayReading`) — no instrument: roll a
worm of it (fat / workable / lean, for anybody), how it will dry (novice), the
colour it will fire (competent), lime, fireclay or common, and whether a
winter has been at it (proficient). Words, never a figure.

## ⭐⭐ Green ware — the water decides (M4, R2)

A shaped piece of clay is GREEN until it is fired: the kernel's
`GreenwareMixin` (`lib/material/Greenware.ts`, composed by the pack's ware,
bricks, flower pots and cones, and by dairying's setting pan — two packs with
no common ancestor, which is the kernel test). Each composer composes
`MoistureMixin` itself: green ware re-wets the way turf does.

- **State is DRYNESS against the air**: `d = (1 − moisture) ÷ (1 − the air's
  equilibrium)` — `plastic` below `green.leatherAt` (0.3), `leather-hard` to
  `green.boneDryAt` (0.85), `bone-dry` past it. *Bone-dry* is as dry as that
  air lets it get, damp shed or dry loft.
- **Leather-hard** trims, joins and stamps (`canTrim`/`canJoin`/`canStamp`);
  **bone-dry** is fireable (`isFireable`).
- **The weather**, read off the piece's own record on every read:
  - its wetness rising past `green.slumpWetting` (rain on an open yard)
    SLUMPS it — the hack cover is what stops it;
  - its temperature below freezing (frost) or above boiling (a wet piece in a
    kiln — the steam burst) while it still holds water SHATTERS it — the
    ruin is the fact, with no crack flaw beside it (B99);
  - a ruined piece goes back to the temper pit (above), where it is
    trodden into clay again;
  - drying faster than its clay stands CHECKS it (a drying crack) — the
    water model's one rule for every drying solid (`cure.crackRate`), a
    board in a hot kiln included; green ware only supplies its clay's crack
    risk (fat clay high; sand, grog, breeze low, read off its composition by
    the ONE rule `MaterialApi.dryingCrackRiskOf` — the number `analyze clay`
    reads, so the reading and the pot agree; weathering raises a clay's
    plasticity, never its crack risk, because a piece does not carry its
    heap's winter — review R2-B2).
- ⭐⭐ **Green ends at firing (R2).** Every read answers only while the body
  is its plastic clay and its heat history has converted nothing. The first
  rung ends it: a fired pot's relation to water is its porosity, weep and
  seep ([faces.md](./faces.md)). The heat-work reconcile asks green ware to
  read its weather FIRST, so a wet piece's burst is seen before a rung swaps
  it to terracotta.

## Forms and the potter's hand (S4, D16)

`shape [<material>] [with|on|at <instrument>] [into <form…>]` is the
platform's (promoted from glass — `command-spec.md`). The INSTRUMENT plans
(`FormingMixin.shapePlanFor`), else the MATERIAL plans by hand
(`ShapeableMixin.shapePlanFor` — `Clay` is one, so a lump in hand affords
`shape`); the clay
YIELDS the piece (`Clay.yieldPieces`).

- **A form** (`lib/craft/FormSpec.ts`) is a class (open · closed · flat ·
  block · cone) × a capacity × a proportion (tall · squat · wide · narrow).
  Words are `/platform/idea/Form` rows (`<root>/idea/form/<word>`: `key`,
  `formClass`, `capacityL`, `proportion`, `product`, `seats`, `greenScale`,
  `discipline`), loaded by path from the instrument's `forms:` list — nothing
  scans. `tall jug holding 3 pints`, `open vessel holding 2 litres`; an
  unknown word is refused naming the five classes.
- ⭐ **The instrument is a kernel `FormingMixin`** (`lib/craft/Forming.ts`,
  made general in review, 2026-10-28): it owns the words, the class check,
  the product and the hand check for ANY plastic mass — the wheel and the
  brick table work clay, the glass blowpipe works a gather — and each
  composer says which mass it works (`formingMassClass` / `formingMass`) and
  plans its own work (`shapePlanFor`); the mixin affords `shape`. The mass
  implements `Formable` (`formingHand`, `refuseBeyond` —
  `lib/craft/Forming.ts`).
- **The product row**: the word's `product`, else the instrument's
  `formTemplates[class]`, else the mass's material's `formTemplates[class]`.
- **The hand decides, deterministically** (`Clay.formingHand`): wall mass =
  the form's nominal (`0.75·V^⅔` kg for an open vessel, more for closed and
  tall) × `ceramics.wall.factor.<band>` (untrained 1.8 … expert 0.85);
  the largest vessel = `ceramics.wall.maxL.<band>` — asked for more, the
  refusal names the band; grade = the band less one. Below your band a pot
  is heavier, never failed. A moulded block, a tile or a handle is its row's
  own mass × `greenScale` — the hand does not change a brick.
- **The piece** is green clay of the clay's material and composition, at its
  water, made by the actor, described by its form, its vessel capacity the
  form's; a form with `seats:` (a jug's `handle`) is born with that bill line
  EMPTY, so `fit handle to jug` fills it. It lands in the actor's hands (a
  day's bricks in the room) and its host is captured (the `fell` precedent —
  the craft-mint defect does not reach green ware). The clay is consumed by
  mass; a heap worked out is gone.
- **The wheel** (`Wheel`, capability `wheel`) throws open, closed, flat and
  cone forms, one pot to a `shape`. **The brick table** (`BrickTable`,
  capability `brick-table`) moulds blocks and tiles: a game-day
  (`ceramics.brickTable.hours`, 8) at `ceramics.brickTable.rate.<band>` an
  hour, ONE stack, as far as the clay goes; the moulder ends up muddy.
  **Hands** coil any class, slowly (`ceramics.hand.minutes`), into the clay
  material's `formTemplates`.
- The goods: `ClayVessel` (`Greenware(Marked(Moisture(Assembled(Named(
  Durable(Composition(CraftVessel)))))))` — mugs, jugs, jars, crocks, bowls,
  dishes, flasks, crucibles, the water filter), `ClayPart` (a pulled handle:
  green, marked, composed and worn, but no vessel — it is FITTED, `fit handle
  to jug`), `Brick` (a green stack), `FlowerPot` (the kernel
  `PlantPot`, potter-made), `Cone` (a kernel `Mould` thrown green — the
  sugar cone). Dairying's `SettingPan` composes green ware too, and ships its
  own `setting-pan` form under its own root. ⭐ Named for their composition
  (review, 2026-10-29): the vessel class was `Ware`, which to everyone but a
  potter means merchandise and read as a second `Good`; and a handle filed
  as a vessel claimed bulk slots a strap of clay does not have.

## ⭐ Trimming and the maker's mark (S5, W4)

At LEATHER-HARD a green piece is worked on, and the piece answers the verb
(`GreenwareMixin` is a `CarvableMixin` and an `ImpressibleMixin`, which
afford `carve` and `impress`):

- **`carve foot on mug`** / `carve trim on jug` / `carve design on bowl` —
  the green piece's `carvePlanFor`: a few minutes with any edge, a little mass
  pared off, the work recorded in words (`a turned foot`, the piece's
  `workedFeatures`, which survive firing), pottery credited. Plastic refuses
  (`too-soft` — it would smear), bone-dry refuses (`too-dry` — it would
  crack), a ruined piece refuses (naming the reclaim). ⭐ Once it is fired,
  `carve` and `impress` are the HOST's own again: green ware defers to the
  class below it (a clay mould carves a riser and takes a pattern like any
  mould) and never shadows it for life (B99).
- **`impress stamp on jug`** — `takeImpression`: the base takes the DEVICE
  the stamp's face bears (`MarkedMixin.markDevice`, an identity path —
  `takeDeviceFrom`; a device is relief by definition, felt as well as seen,
  and stamping leaves the piece's own writing in whatever form it was —
  B106). The potter's
  `Stamp` (`/trade/ceramics/thing/Stamp`, a `Marked` + `Crafted` good: an
  authorable `markDevice`, else whoever cut it) affords `impress`; a tool
  that bears no device leaves only a dent. The same leather-hard refusals.
  ⭐ One mark, one home (review, 2026-10-28): the potter's stamp, a die's
  device on a struck token and the device cut in the stamp or the die are
  all `MarkedMixin`'s — ware, bricks, flower pots, ingots, castings, stamps
  and dies compose it, a stamped brick does not stack with a bare one, and
  `read` names whose mark it is alongside any writing. ⭐ **Whose mark is
  the VIEWER's knowledge** (`Marked.deviceFor(viewer)`, B106): *your own
  mark*, *Hester's mark* when you know Hester (your recognition of her,
  read by `knownNameOf`, which works while she is away), else *a maker's
  mark you do not know*. Learning whose mark is whose is connoisseurship,
  not a free fact. A commercial BRAND (`BrandedMixin`) is a
  claim of whose product, kept separate: the two can disagree (a forgery).
- **The base**: once fired, a piece's look reads *it bears X's mark* (as
  the viewer knows X; the
  device's own line) and *it was fired <date>* — the date is its heat history's
  `lastTransitionAtS`. ⭐ A shelf of a player's own pots is their improvement,
  visible: the heavy first mug and the light later one, marked and dated.

## The clamp, the kiln, linings and crucibles

### ⭐⭐ The clamp grades by position (S6, D15)

`BrickClamp` (`/trade/ceramics/thing/BrickClamp`) = `ContainerMixin(Firebox)`
— the charcoal clamp's chain, but its own class (a `CharcoalPit` would afford
`char`). A clamp is not a chamber at one heat: the bricks by the fire
channels get far more than the casing does, and **each brick's own
heat-work decides what it comes out as, by the clay's three transitions**
(B1) — there is no grade table here.

- **Setting** (`onFireStart(by)`, the `Burner` hook `fire` calls before the
  hold): every brick stack in the clamp is split three ways with
  `Stackable.split` — a channel share, a skin share
  (`ceramics.clamp.skinShare` 0.25) and the middle — and each part is
  stamped with its position **on the brick** (`Brick.clampPosition`, a
  persistent field: a stuff id would not survive a restart mid-burn). The
  channel share is the setter's `brickmaking` band:
  `ceramics.clamp.channelShare.<band>` untrained 0.35 · novice 0.3 ·
  competent 0.22 · proficient 0.16 · expert 0.12 — ⭐ a better setter makes
  fewer over-burnt bricks, deterministically. A stack under three, or a
  piece that is not a stack (a pot in a pit firing), sits in the middle. The
  setter is recorded (`setBy`) and the wind over the clamp at lighting is
  read once (`skinWindMs`).
- **The fire answers each brick by its place** (`heldTemperatureKFor`):
  channel × `channelFactor` 1.25 · middle × 1 · skin × `skinFactor` 0.9,
  less `rainK` 100 K × the clamp's own wetness and `windKPerMs` 6 K per m/s.
- **The numbers** (clamp row 1474 K, draught 0.6 → completeness ≈ 0.71,
  held ≈ 1366 K; a three-day burn; clay's transitions at 1200/1330/1480 K,
  zK 30, 7200 s — over three days a rung completes ≈ 47 K under its
  reference): the middle ≈ 1366 K → **earthenware** (the sound brick); the
  channels ≈ 1707 K, past the stoneware rung's 1560 K scorch → **clinker**
  (over-burnt); the skin ≈ 1229 K → **terracotta** (the soft place brick).
  A soaked skin (≈ 1129 K) does not complete terracotta at all — under-burnt.
  ⭐ Fired `--draught half` the burn is starved (completeness ≈ 0.59, under
  the kernel's 0.6) and **reducing**: the middle ≈ 1322 K comes out the
  earthenware's reducing twin — the blue brick — and the skin grey.
- It glows (a `Firebox` is a `LightSource`) and smells of its fuel's
  combustion products (the fire build's).
- ⭐ **A split brick is the stack's brick.** `Brick.onSplit` carries the
  material, the composition, the water state, the temperature, the heat-work
  record, the maker, the mark and the grade onto the split-off — a fresh
  clone of the row otherwise came back green (the `Clay`/`Ore` precedent).
- The rows — `clamp`, `pit-firing` (same class: 1250 K, oak, draught 0.45 —
  smoky and reducing), `updraft-kiln` (an `Oven` with a 30-firebrick lining)
  — stand in the Delight brickfield's clamp ground and potter's shop (W6).
  ⚠ The clamp answers to `clamp`, never `brick`/`bricks`/`stack`: `put
  bricks in clamp` must bind the bricks, and the brick stack beside it is
  the counter.

### Linings and crucibles (D19, D20)

The lining is a part line on a chamber furnace, worn per firing and relined
with `fit lining to <furnace>` — see [fire.md § Linings](./fire.md). The
keeper's brain `relines` lives in this pack (`src/behavior/relines.ts`). The
Rejection store sells firebrick (an import until the brickfield supplies it),
the smelting pack's fired crucible (the potter's `/trade/ceramics/thing/
crucible` row is GREEN ware — a crucible for sale is fired), a pig of lead
and a bar of tin (the glaze fluxes, commons rows).

### `measure size` — the statute brick (S11, D30)

`/trade/ceramics/idea/reading/size` (`SizeReading`, instrument `gauging`,
discipline `brickmaking`): a brick's length is read off its VOLUME (mass ÷
its material's density) in the statute's proportion, so a brick from an
oversize mould — or a clay that shrank less — reads **off standard** past
5 %. The statute is the innermost polity's `standards` keys `brick-length` /
`brick-width` / `brick-depth` (Terminus: 9 × 4½ × 2¼ in; the realm default
the same). The figure is blurred by the reader's band like any reading
(competence resolves detail, never access) — ⚠ an untrained reader's ±8 %
can blur a 6 % oversize brick back to *true*. A seated gauger's measure is
*of record*; nothing is written on the brick.

## The works and their people

### The Delight brickfield-and-pottery (S20)

One works, two trades, on the river flats southeast of the valley gate —
`hearts-delight/content/world/terminus/hearts-delight/brickfield/**`, a
`CartesianZone` of its own (20 m cells) carrying its **own `deposit:`**,
`idea/deposit/valley-alluvium`: an honest column — loam 0…−0.3 m, the
Delight's alluvial clay to −0.9, a lens of river sand to −1.2, gravel
(`drift`), granite. ⭐ **The water table at −1.2 is what "worked out"
means:** the clay pit (`location/clay-pit`, an `OpenWorking`) is authored
at `floorDepthM: 1.2`, its floor the wet gravel, so clay and sand are WALL
faces (`dig clay`, `dig sand` — about 800 lumps and 400 loads on a 200 m
face) and the floor refuses *at the water*. The valley's first pit
(`location/old-pit`) is authored flooded — a `Locale` with a
`WaterFixture` pond (a flooded pit cites no Watercourse, so not a
`Shore`); the dynamic flood of a working is mining-slate's.

| place | what is there |
|---|---|
| weathering ground | open (no roof, no walls) — heaps lie out the winter here; the brickmaster |
| clay pit | the working; three quarry spades |
| old pit | the pond; the **old moulding table**, its mould cut before the statute (`idea/form/old-brick`, `greenScale` 1.6 — a brick off it measures *off standard*) |
| temper yard | the temper pit (a world row: the blend is CALLED the Delight's alluvial clay, which a trade row may not name — R4), pails, starting heaps of breeze and sand |
| moulding shed | two brick tables; the gauging rod (`measure size`); the two moulders |
| hacks | `enclosure: none`, a canvas `roof:` — rain kept off, the field's air (a frost still reaches a wet brick) |
| clamp ground | the clamp, the pit firing, coal; the **brick stack** (the brickmaster's counter: bricks, firebricks, tiles, and breeze as its one import line); the setter |
| potter's shop | the wheel, the updraft kiln, the ash- and lead-glaze tubs, the potter's stamp (`markDevice` the potter), four sets of cones, the **ware shelf**, a water butt, ash, a pig of lead, charcoal; the potter |

Every place but the pit (`OpenWorking` is persistable itself) and the
pond is a `PersistentLocale`: a heap's winter, a day's green bricks, a
clamp mid-burn and the ware on the boards survive a restart (AC 22). The
floors are the alluvial clay, unworked outdoors and beaten in the sheds
— the ground soils you.

**Glazes are mixing, not recipes (R1).** Each tub is a kernel `GlazeTub`
(a `CompoundingVessel` that is also a coat BATH — B101) whose
`productMaterial` is its glaze (`glaze/ash`, `glaze/lead`, the slip): put
ash (or lead) in with a little clay and water, `stir tub`, and the slurry is
that glaze by the vessel's name and its make-up by what was weighed. Then
`dip` a pot in it. A bath with less left than the pot's skin takes coats
nothing (`bath-too-low`).

⭐ **The work is the matter's (B101).** A compounding vessel says whether its
contents are KNEADED (a deformable mass: plastic clay, raw rubber; a quarter
game-hour, heavy) or STIRRED (powders, granules, liquids; two light minutes),
from what is in it (`workVerb`), and the wrong verb is refused, naming the
right one. Stirring the temper used to be a two-minute shortcut past the
kneading. The *vessel* names the glaze — there is no
dominant-flux naming. Salt glaze is the chamber-vapor path with any salt a
player brings (the Delight potter is an earthenware potter).

### The businesses and the people

- **The brickmaster** (`idea/brickmaster`, titled the field): seats
  `master`, `moulder` (headcount 3, two rostered — an opening), `setter`;
  each slot carries its `station`. Hal Brennick holds the judgment seat
  (idles); **Jory Sallow** (proficient) and **Bet Amory** (novice) run
  `moulds`; **Emmet Crane** (proficient) runs `sets`.
- **The pottery** (`idea/pottery`, titled the shop — the longer extent
  wins): seats `potter` and `hand` (an opening). **Wenna Hale** runs
  `pots`, the shopkeeping trade's `consigns` (ware shelf → the Terminus
  stall) and `relines` (the kiln's lining, firebrick off the brick stack).

The brains (`trade-ceramics/src/behavior/`) are the `reels` shape: ONE act
per beat, furthest along first, each stage only when its input is there,
every step a LITERAL verb through `forceCommand`, every walk an AUTHORED
way, and ⚠ **never a player's goods** (a good stamped to a player stops
the step that would move it).

- **`moulds`** — bricks in hand → the hacks (`drop bricks`); the gang's
  green bricks on the pallet → `get 10 bricks`; tempered clay in hand →
  `shape clay into brick` (a day at the table); a tempered clay in the pit
  → fetched; raw clay in the pit → an opener in (`put sand in pit`) and
  `knead pit`; raw clay in hand → into the pit; else `get spade`, `dig
  clay`.
- **`sets`** — a lit clamp is left alone, and stopped (`douse`) once its
  bricks are burnt; burnt bricks in hand → `put bricks in counter`; a cold
  clamp's burnt bricks → `get bricks from clamp`; bone-dry bricks in hand →
  `put bricks in clamp` (damp ones go back); a clamp charged to `batch`
  (40) → `stoke coal`, `ignite clamp`, `fire clamp for 3 days`; else fetch
  bone-dry bricks from the hacks. ⭐ The setter's own band sets the channel
  share (`BrickClamp.onFireStart` reads who fires it).
- **`pots`** — a lit kiln is left, and doused when its load is done;
  fired ware in hand is lead-glazed inside and put back (jars, crocks) or
  shelved (`put jug in shelf`); a cold kiln's finished ware drawn a piece a
  beat; green ware in hand onto the boards (`drop`); a kiln-load (4) fired
  (`fire kiln for 10 hours`); bone-dry ware off the boards into the kiln;
  clay in hand thrown into the next form in turn while the boards hold
  fewer than 6; else dig. ⚠ She throws the clay as it comes out of the
  ground, and a fat clay that dries fast checks — the clay's honest
  behaviour.

⚠ Nothing in the valley is booted: the brickfield runs while something
loads it (a visitor, the general store keeper's errand).

### The Terminus stall and the re-pointed lines (D28)

The **potter's stall** (`terminus/…/market/thing/potters-stall`, propped
on the square) is the market's counter, self-service like the fish stall,
with eight lines — plant pots, mug, jug, crock, bowl, jar, flask — each
`from: { supplier: the pottery, importBackstop: 1.5 }`. The potter
consigns into it; a player potter may too. The general store's plant pots
and jar (the jar replaced the jar of honey — the honey is the
beekeeper's), the chandlery's jar and flask (added beside its soft soap
and lye) and the Rejection store's firebrick (supplier: the brickfield)
name the same shape; the general store keeper's `stocks` beat has an
authored way down the Delight road to the potter's shop. The mechanism is
[retail.md § Buy local first](./retail.md). ⭐ Every row those lines name
is the FIRED good (the row's material is the fired body), so an import
arrives finished with no maker, and `shape` restamps clay onto the
potter's own. The two plant-pot rows carry the farming pot's soil,
moisture reserve and plant slot (content copied — the trades share no
dependency).

### The cones (D31)

`temperature.yaml` carries `handTool: pyrometry`; the pottery props
`/trade/ceramics/thing/pyrometric-cones` (a `Tool`). `analyze temperature
kiln` on a lit burner with a set in reach adds the cones to the glow:
anybody counts how many have gone over; a competent reader names them
(08 · 04 · 6 — terracotta, earthenware and stoneware heat, ±15 K
bending); a proficient one says what the ware has had. The set is SPENT
by the read (a bent cone does not stand up again).

## Settings (`settings/ceramics.yaml`)

| key | default | meaning |
|---|---|---|
| `ceramics.weathering.ceilingDays` | 40 | freeze-thaw days past which a heap gets no better |
| `ceramics.temper.minutes` | 20 | game-minutes of treading a pit |
| `ceramics.temper.waterLPerKg` | 0.05 | the pit's water a kilogram of clay takes up |
| `ceramics.clamp.channelFactor` / `skinFactor` | 1.25 / 0.9 | the clamp's heat by position, × the held heat |
| `ceramics.clamp.skinShare` | 0.25 | the casing's share of the charge |
| `ceramics.clamp.channelShare.<band>` | 0.35 … 0.12 | the setter's channel share, untrained … expert |
| `ceramics.clamp.rainK` / `windKPerMs` | 100 / 6 | K off the skin for a soaked clamp / per m/s of wind |
| `ceramics.lining.relineBelow` | 0.4 | lining health under which a keeper relines |
| `ceramics.crucible.wearPerMelt` | 0.12 | a crucible's wear per melt (read by smelting) |

⚠ Pack dials are read by string key (`AppApi.setting('ceramics.…')`) with a
literal fallback — no kernel `AppSettingKeys` entry.

## ⭐ Moulding and founding — the ceramics side of W7

The moulding half of the build lives in [mould.md](./mould.md); what is
ceramics' here:

- **Plaster** — `rock/gypsum` burns to `mineral/plaster` by the one heat
  rule (~420 K for an hour; the foundry's core oven); `stir` it with water in
  the **plaster tub** and the slurry (`plaster-slurry`, a `kinetics:` set of 15
  minutes → `cast-plaster`) poured into the two-piece **plaster case**, a
  model pressed in before it sets, cures into a **plaster mould** with the
  model's cavity.
- **Slip casting** — `slip` authors an `absorbedByMould` drain: poured into
  a plaster mould (`absorbent`), the mould drinks the water into its own
  wetness, slower as it wets, and a cup of clay sets in the cavity at
  leather-hard (`get mug from mould`). In a mould that does not drink, slip
  never sets.
- **Pressed tiles** — the **tile mould** (`pressKind: press`) on the **tile
  press** (`ScrewPress`): a pat of tempered clay is pressed the turn the
  screw beats its stiffness (`(1 − moisture)²` — wetter clay gives sooner)
  and the green tile comes straight out.
- **Recast** — a fired pot salvages to sherds by its body's
  `brokenTemplate`, whole by mass (`CraftingLogic` salvage, ceramics S18).

The rows stand in Rejection's foundry (`plaster-tub`, `slip-tub`,
`plaster-case`, `tile-mould`, `tile-press`); `pottery` is the discipline
the plaster case, the plaster mould and the tile mould credit.

## Deferred — where each piece lands

Salvaged from the build plan at the pre-merge sweep (2026-10-29); the plan
retires, and these are the attach points it recorded.

| seam | destination |
|---|---|
| casks, pitch-lined amphorae, tin-lined copper onto faces | `maturation.md` § a recorded target (W7's sweep writes the note; no migration) |
| coats on building faces, the painted image, binders | paint-slate (rides `Faced` for goods, construction layers for buildings) |
| wet clothes and sweat reading `evaporativeDepressionK` | `exertion.md` / `textiles.md` deferred lists |
| frost tilth, frost damage to brick (the count's next readers) | repair W7, `soil.md` |
| lime mortar by cure, repointing, the spall, `maintain` | repair W7 |
| the Scotch and Hoffmann kilns, the pugmill, the jigger, extrusion | brickmaking-slate § 5, pottery-slate § 4 |
| kintsugi (lacquer from a tapped tree) | `taps.md` deferred list |
| porcelain and kaolin | pottery-slate § 9 |
| salt-glaze fumes as harm | the potash slate's smoke ordinance |
| a fire code, zoning the brickfield, a public kiln | construction-slate, civics |
| ringing the curfew on a schedule | civics (the curfew ordinance) |
| coin striking authority; the die as a credential | `banking.md` coinage, attestation-slate |
| bricks and wares by barge; breakage in transit; straw | river-slate, freight-slate |
| the per-brick levy | brickmaking-slate § 3 |
| drain tiles, brick sewers, flue liners, clay lamps | watershed's conduit ladder, cleanliness's flue, fire's lamp — rows on this build's bodies |
| `interchangeable` as an authored joint flag | assembly tail (the derived skip ships here) |
| a pit you climb into (the bell mould as a place) | moulding-slate (if the owner wants it) |
| a working that FLOODS when it reaches the water table (the valley pit standing as a pond is authored, not dynamic) | mining-slate (*a pit that fills, and a pump*) |
| the clamp's over-burnt CLINKER grade as the paving brick (Dutch *klinkers*) — roads-slate § 4's surface table has no brick today | roads-slate (the coordinator edits the slate) |
| drain tiles under a McAdam road (the brickfield's tile mould, laid for drainage) | roads-slate |
| `hammer` promoted behind a protocol | command-taxonomy-slate |
| the oven's own warm-up (thermal mass on the furnace) | `fire.md` deferred (unchanged) |

From the review's combining re-plan (R1):

- `dissolve` → `dissolvedSolidsKgPerL`, never `makeup` (R1-D9).
- A bar wire drive (`hospitality.dirty.wire.test.ts`): pour · stir · strain
  a martini and a negroni by serving, read the glass's dose.
- `nutrientAmounts` units are heterogeneous (per litre, per 100 g, per
  serving) and display-only; a nutrition pass should re-base them on
  `servingKg` — a slate item, not this MR.
- The dough rows hydrate above 100 % by mass (flour 1 L = 0.565 kg against
  0.6 kg water, `dough.yaml`); a batter, not a dough. Rows, baking's.
- `Clay`'s stash fold → `absorbMakeup`.

⭐ **Judged legitimately different — do not re-litigate** (the review's
overlap survey, R2; the overlaps it did NOT judge different are
matter-model-overlaps-slate):
Thermal shock vs materials-response heat (wounds through coverings vs flaws
in an object); furnace lining wear (already `Assembled.wearLine`); relines
vs residence maintains (only the walking code shared — D8); the screw
press's force (already the mould's `loadN`); `wetBulbK` (single-sourced);
Ringable as a one-verb hook (only its sound shape, D4); Clay/Gather hand
tables (per-material tuning behind the kernel `FormingHand`);
`Material.formTemplates` (a fallback behind `Form.product`);
`heatWork.firedBy` vs `Crafted.maker` (different people); `Impression`
(no key-copy mechanism to overlap); freeze-thaw counting vs residence
shell weathering (only if wall spalling is built).

## Cross-references

[thermal.md](./thermal.md) · [faces.md](./faces.md) ·
[materials-response.md](./materials-response.md) · [weather.md](./weather.md)
(the freeze-thaw count) · [content-packs.md](./content-packs.md)

## History

- **Build** (2026-10-20 → 10-27, `reqs/ceramics`, MR !437): W0–W8 — one
  rule fires everything, faces and the lead leach, brittle bodies and
  `ring`, clay's year, the potter's hand, the clamp and the kiln, the works
  and their people, moulding and founding, crystallizing and the filter.
- **The owner's review** (2026-10-28 → 10-29, `d22c2a429..6e61def6a`): two
  re-plans — R1 *combining is one rule* (the one fold, composition one
  record) and R2 *the overlap sweep* (six bugs, nine duplicates, the design
  overlaps slated) — then a mixin-by-mixin pass, B94–B109: `IntegrityMixin`,
  castings Crafted, `MoistureMixin`, the failure policy (refuse · skip ·
  throw), Seasoning, Greenware, Faced, Compounding, Forming, PatternStock,
  Mould, Thermal, Marked, brains moving by `go`, `ClayVessel`/`ClayPart`,
  and pH. Each change's reasoning is in its commit message.
- **The finalize** (2026-10-30, from `a7176166b`): the drives ran once in
  full (run 3), and each failure was fixed and re-checked in isolation on a
  standing world. The fixes: a slip is stirred (B101 had made every clay a
  knead); a pool freezes IN a vessel that holds things; grog treads in by its
  form; the cast lands before its runner; the bell pit is loam; eleven
  "glazed" rows had kept the retired `fused` state, and the dairy setting pan
  is glazed earthenware again. The plan and requirements docs retired here,
  and the split of these drives into blocking and soak went to build-4 and
  shipped with the soak harness (2026-10-30): each blocking drive stages its
  span steps' outcomes, and its `.soak.dirty` twin keeps the spans —
  testing.md § The soak lane.
