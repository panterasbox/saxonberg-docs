# Painting — a building's outside, the coat that weathers, paint and glue

What a building's outside is and how it wears; how a coat sets and how it
goes; what paint is; what glue is; and the three hazards the trade carries.
The kernel holds the mechanism; `trade-painting` (`/trade/painting`) holds
the colorman's trade, and Wharfside holds the colorman and the house painter.

Built by the paint build (2026-10-31). Sibling docs:
[construction.md](./construction.md) (places, structures, the landing),
[faces.md](./faces.md) (faces and coats), [assembly.md](./assembly.md)
(joints), [holding.md](./holding.md) (the term), [taps.md](./taps.md),
[fire.md](./fire.md), [metabolism.md](./metabolism.md).

## 1. The landing — where a structure meets the weather

A structure is an Idea, so a player engages it from a place that touches
it: *a place that does not hold its own air, with an exit into a member
that is enclosed or is the entrance — or with the structure lying alongside*
(construction.md § the landing). Landed structures join the room's `here`
pool: `look`, `analyze condition`, `measure paint`, `paint` and
`maintain` bind a building from its landing and nowhere else, and refuse elsewhere in words (*"You cannot reach
the outside of the cottage from here."*).

## 2. The outside: a face and a condition — no new mixins

A building's outside is two shipped mixins, composed once on
`lib/structure/Building` — a structure with an outside. Building rows name
the `platform/idea/Building` twin; a holding's buildings are
`HoldingBuilding`s. ⭐ A structure that is not a building has no outside of
its own: the Waponi Woo peak (a bare `Structure` until the mountain slate)
cannot be painted or maintained. A `Ship`'s hull takes coats (weathered
wherever she is, area from her hull) and — since the merge into harbors —
has a fabric under them too: her seams, a `Durable` condition worn at the
building dial once the coat is gone, which `maintain` with a caulker's kit
makes sound (and which, worn, LEAKS into her bilge). The house painter
works on buildings only.

⭐ **A holding's buildings are AUTHORED** (harbors review, 2026-11-03 —
the paint build grouped them by a `building:` key on the retired
`floorplan:` list, and the merge into harbors briefly derived them from
how rooms connect; see [holding.md](./holding.md) § Condition): the
programme row names `buildings:` (main first), each a building row with
its own `extent` and `entrance`, and the holding mints one
`HoldingBuilding` per row, its rooms the plan's rooms under that extent.
A farmstead's house and its barn are two buildings, each with its own
coat and condition;
open ground — the yard, a tract, Hinkley's garden — is no building's, and is
where they are painted from. A holding's rooms are keyed clones of shared
rows, so no path names them (holding.md § Identity): **a place asks its
warren which building it is in** (`Warren.structureOf`, a hook the holding
answers from its buildings' room instances; `Place.structureNow` asks it
first, then the location graph by identity path). ⭐ A structure answers
its outside as facts — `Structure.readOutside()` (coat, band, cause, area,
litres, and `weatherSurface()`: *wall and roof*, a ship's *hull*) — and
three reads word them (controller-liquidity W17, when the owner retired
the `survey` verb): `look <building>` says the coat and the condition
word; **`analyze condition <building>`** says WHY (*the paint has gone;
the weather is at what was under it*), to every band; **`measure paint
<building>`** (a `Reading` row, `PaintReading` — an instrument rung, a
rule or a gauging rod, unbanded) says the job: the area to the weather
and the litres one coat takes. `RoomsReading` (`analyze rooms`) asks
the structure a room is part of for the shell line. So `analyze rooms`
in a field reads no fabric and `maintain` there finds nothing to keep. By path, only
the main building (the first named) answers, at the holding's own extent —
what the ascent gate reads; the others are kept under
`<extent>@<row leaf>`.

| mixin | what it claims |
|---|---|
| `FacedMixin` | one outside a coat can be laid on (`coatAreaM2` = the exterior walls and roof of its members), **and the weather works on it**: the coats thin where it reaches them, set by the air, fade in the sun |
| `DurableMixin` | a condition — the fabric under the coat — which the weather wears only while the coat is gone |

⭐ **Weathering is every good's** (folded into `FacedMixin` at review): it
is decomposition's exposure clock for a coat. Indoors, in a pack or on a
rack the weather reaches nothing, so nothing changes — Law 2 holds by
construction — and a cart's coat in the yard, a sign board hung outside and
a building weather by the one walk. What the weather does to the BODY once
the coat is gone is the host's `bareWearPerDay`: a building answers its
dial; a good answers nothing yet (decomposition's per-material rate).

### Setting — the binder's chemistry, read off the air

`KineticSpec.sets` says what drives a material's first change: `heat` (the
default — every glaze, cure, firing), or the air: **carbonation** (a
limewash; it wants air and time, slow where the air does not change),
**oxidation** (an oil; it wants air and warms as it does —
`exothermicJPerKg`), **drying** (a size, a tempera; it wants dry air),
**solvent** (a varnish, a rubber cement). `Faced.airCoats(segments)` walks
the air: rain on a wet coat RUNS it, frost CRAZES it (the shipped fault
states). `heatCoats` never touches an air-set coat. A thermal good sets
its coats over its room's air (`Thermal.reconcileCoats`), and an oil warms
it.

### Weathering — thickness is the arithmetic (`Faced.reconcileWeather`)

`Material.weatheringMPerGameYear` is the rate a SET coat of it thins where
the weather reaches it (an oil two seasons a coat, a limewash one or two, a
glaze or tar never). Outermost first; when a coat is gone the next meets the
world; two coats last twice as long; a crazed or run coat goes twice as
fast. The walk is `BiomeApi.airSegmentsFor` at the thing's `weatherScope`
(a good's container; a building's standing landing), so a wet day and a dry
one do different things; a long absence integrates the tail the weather walk
keeps. It reconciles on read — a look, a read of lettered words, an
`analyze condition`, a coat laid. A building with no landing standing leaves its coat unwalked
until one is (nothing is lost), but a bare one still wears. A thermal good
sets its own wet coats over its room's air (`Thermal.reconcileCoats`, which
also takes an oil's heat); everything else sets them in this walk.

### The fabric — a building's `Durable` condition

A building's condition wears at `0.5 / structure.weather.daysToWorn` per
game day — **only while the coat is gone** (`bareWearPerDay`; reconciled
whenever the condition is read). It reads as `Durable`'s word, no gauge
(`sound` · `worn` · `badly worn` · `broken`); the cause line is read off
the coat (`Faced.weatherCause`: *"the paint is going"*, *"the paint has
gone; the weather is at what was under it"*, *"the new oil is still
curing"*). `maintain` restores the condition and nothing else (*"a coat is
a brush's work"*).

### Staged and owned

A building persists only once it is **sold** — a PERSON holds the covering
parcel (`isTitleSold`). A staged building writes no record and comes back as
its row authored it whenever the world restages (today: every boot). A ship
keeps her hull with her record, coats and seams both (she keeps a record
whoever holds her — harbors; `Faced.persistsCoats` is `true` on her). A holding's house persists
always (a holding exists because somebody holds it).

## 3. Paint is a mixture

`lib/material/Paint`: a binder (any coat-laying material that sets) with
pigments by mass, perhaps a thinner. Its color is the pigments' colors
weighted by share × hiding power; its lightfastness its least fast pigment;
its coat word the binder's. The makeup rides the liquid's payload
(`BulkPayload.paint`) and is kept on the coat (`CoatLayer.paint`); a paint
coat's MATERIAL is its binder, so the binder sets and weathers it. There is
no paint table: a second paint is a pigment row.

**Pigments are items** (a paper of yellow ochre), because a pot's liquid is
one slot: the binder is poured, the pigment put in, the pot stirred (`stir pot`;
a paint is runny, so `knead` refuses it — W19) (`PaintPot.payloadFor`). An egg in the pot binds as tempera.

## 4. `paint` — one verb, the target plans it

`paint <thing> [with <pot>] [using <brush>]` — a brush in hand affords it
(`platform/thing/Brush`, a static on a class). The TARGET plans the coat
(`Faced.coatPlanFor`): a wet film `structure.paint.coatThicknessM` over its
area, exactly that many litres from the pot (a pot with less coats
nothing), the weather walked to now first. `write <words> on <board> with
<pot>` letters the words as a coat (`CoatLayer.marks`): anyone reads them,
until the weather takes the coat and the words with it.

## 5. Glue — the glued joint takes a real glue

A joint document may say `takes: glue` (`glued`). `fit`'s raise arm draws glue from
an open vessel in reach (`crafting.glueLitresPerJoint`) and marks it on the
joints; the GLUE decides (`Material.adhesive`): the line sets by the glue's
air rule and is weak until it has; a knock before it sets costs more; the
damp slackens it by what the glue does not resist (a hide line lets go in a
wet cellar, a casein one holds); past `softensAtK` it lets go; salvage reads
the glue (an animal glue comes apart hot and wet, casein never). Glue is
made from what the world throws away (the `GluePot`: hide scraps, bone, fish
skins), from curd and lime (casein), flour (paste), rosin (cement), rubber
and turpentine (cement).

## 6. The hazards

- **Lead.** Every lead row doses the toxin type `lead` (the shipped rows
  named the condition and every dose was dropped). The colic band comes at a
  burden of two; grinding a lead stock and brushing lead paint dose the body
  (`metabolism.leadDustDose`, `leadSkinDosePerL`; a glove halves the skin).
- **Oily rags.** `SelfHeatingMixin` (kernel, promoted from hay): a heap
  heats from its host's hook and sheds by its own; a combustible heap whose
  core passes its autoignition point catches. A rag dipped in the pot heats
  from its oil coat — a heap catches, one spread rag does not, a wet one
  does not heat.
- **Turpentine.** A volatile: an open pour or a stir puts its vapor in the
  room, toxic per breath; the room's air changes clear it.

## 7. Resin

A tap names its tool (`TapSpec.instrument`, default `boring`): a Scots pine
is cut with a blade and bleeds gum through warm weather (Hanging Wood's
sugarbush pines). The still cuts the gum to turpentine and leaves rosin;
rosin and turpentine stir to varnish.

## 8. Wharfside

The colorman's (Ira Pell) off the towpath: a shop with its counter and
hearth, a yard with the quern, a shed for the rags. The towpath cottage is
the town's limewashed building, its own parcel so it can be sold. Tam Rook,
the house painter (he answers to *painter*; a boat's line is a *bow
line*), works a rota of the towpath and Mayfield Row and paints the landed
building whose coat is gone (the `paints` brain, through the literal verb).
A swept flue gives lampblack where the realm names it
(`cleanliness.sootTemplate`).

## Open

- The corrosion jar (vinegar on lead or copper → white lead, verdigris) and
  the madder-lake strike are not built: the colorman stocks them.
- A pour joins only like with like, so turpentine cannot thin a pot of oil
  paint from the verbs.
- Only `fit`'s raise arm consumes glue; re-making a glued joint by the
  replace arm does not.
- The house painter does not restock his own pot.
- **Upkeep by layer did not ship.** The repair plan's W7 (`maintain` lays
  a named material from hand into the worst layer, by that layer's trade,
  at the hand's band; the keeper's employer buys what is laid) was
  replaced by this build's `Faced` + `Durable` model, where `maintain` with
  a kit restores the fabric with no materials and no band. The layered
  fabric, the materials-in-hand mend and the par-sheet keeper remain
  repair-slate's; masonry and
  brickmaking route lime mortar, repointing and the spall to it.

## History

Built by the paint build (MR !513, 2026-10-31 → 11-02). Review moved
four load-bearing things, and the reasons are worth keeping:

- **No new mixins.** The build shipped `WeatheringMixin` and `ShellMixin`;
  review folded the first into `FacedMixin` (every outside weathers, on
  decomposition's exposure rule) and the second into `DurableMixin` (the
  fabric is a building's condition) — `e9ce36e85`.
- **A Building is not every Structure.** The outside lived on the
  `Structure` twin every row names, so a ship and a mountain peak had a
  building's; `lib/structure/Building` and its twin split off —
  `e01aa1599`. ⚠ The location graph tested `instanceof` the twin and lost
  every building until it read the base — `08125e0b7`.
- **No new path concept.** `Stuff.getPlacePath()` (a persistence key passed
  off as a place) became the warren's say, `Warren.structureOf` —
  `cbb51116d`.
- **A sailed ship lands nowhere.** `Exit.alongsideNow()` gained its third
  answer (`undefined`) so a way aboard never falls back to its berth —
  `a6ea93601`.

Merged into harbors (MR !511, `5cfd1a5db`): harbors' own kernel
`WeatheringMixin` on the warren was retired in favor of this model, so a
ship's hull is `Durable(Faced(Structure))` on the same
`structure.weather.daysToWorn` dial, and the boat's bow rope took the
name *bow line* (`/platform/thing/bow-line`), leaving *painter* to the
house painter. Harbors review then made a holding's buildings authored
rows (`8125deeed`, `a20704a98`).

The decision numbers cited in code comments (`paint D1`…, `B1`…) are the
build plan's, retired at the sweep; read them with
`git show a0183ceb2:docs/plans/paint-plan.md` (its drive record and
deferred seams too — the seams went to the respiration, decor,
construction, visual-arts and decomposition slates).
