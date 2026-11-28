# Moulds — the cavity, and the envelope round it

A **mould** is a vessel that answers one question: *when what is in me
sets, what shape comes out?* Shipped by the dairy build (2026-10) with the
**cheese hoop**, the **ingot mould** and the **ice pan**; made an
**envelope** by the ceramics build's W7 (2026-10-20): impressions of
anything, a pour that obeys its receiver and Chvorinov, one load from a
weight or a screw, the cure and the slip cast, the founder and the bell.

Source: `packages/server/src/mud/lib/mould/Mould.ts` (`MouldMixin`, the
`Mould` interface, `SetPool`, `ReceivedPour`),
`lib/mould/Impression.ts` (the copied shape), `lib/material/DrainSetting.ts`
(the drain set's law, the matter's), the instanceable twin
`platform/thing/Mould.ts`, and `platform/thing/ScrewPress.ts` (the press).

## ⭐⭐ The shared thing is the cavity (M2)

The two kinds of moulding are one shape and different matter:

- **casting** — a fluid sets in a container;
- **pressing** — a plastic solid is pushed in and released.

A mould's shape is an **authored row** (`castTemplate`) or an
**impression** (`MouldMixin.impression`, an `ImpressionData`). How matter
gets in (pour, press, blow) and how it sets (freeze, drain, cure, release
at once) are the MATTER's. The set site asks the container first and falls
back to the material:

```
template    = blown       ? material.castTemplate          (a ruined lump)
            : impression  ? impression.kindPath            (castTemplate empty)
            :               mould.castTemplateFor(material) ?? material.castTemplate
destination = mould ?? the vessel's own scope
```

A plain vessel keeps its lump *in* it when it holds things (a temper pit's
water freezes in the pit and thaws back into it — the ceramics finalize),
and drops it *beside* it when it does not (a jug). A mould keeps what set in it, in its own shape, **inside it** (a
reusable one) or breaks to release it (a consumed one).

> ⭐ **A firing asks the mould too** (rubber build; the ceramics build made
> it the HEAT TRANSITION, not the verb): a piece whose heat-work reaches a
> rung while it stands in a mould not its own cast hands the mould a pool —
> the same `castInto`. See [thermal.md](./thermal.md) § heat-work and
> [rubber.md](./rubber.md). `holdsCast()` is true only for a CAST (the
> mould's row, the impression's kind, anything carrying `castFrom`, or a
> cracked mould's lump): a mould charged with solids waiting to be fired or
> pressed holds a charge, not a cast.

## ⭐⭐ Impressions — anyone, anything (M9, D21)

`impress <pattern> in|on <medium>` (platform, `cmd/crafting/impress.yaml`;
`ImpressController` is dumb). The MEDIUM answers through the
`Impressible` protocol (`ImpressibleMixin`, `lib/craft/Impressible.ts` — an
`ActOutcome` or an `ActRefusal`, at once):

| medium | what it takes |
|---|---|
| a leather-hard pot (`Greenware`) | a **mark** — the stamp's identity, on its base (S5) |
| a mould with `impressible: true` — a sand flask, a wax block, a plaster case whose slurry has not cured | an **impression** of the whole shape |
| anything else | *"is cut to its own shape. Nothing presses into it."* |

What a thing leaves pressed into something soft is its own answer —
`TangibleMixin.leaveImpression(nowS)` (a `@hook`):

- **the kind** — its template path (lineage, never a minted identity);
- **its volume** — mass ÷ its material's density;
- **its form and parts**, as description;
- **whose it was** — the chattel stamp, else its maker, else (a body)
  itself; the name it went by, kept for when they are not in the world;
- **what the shape carries because the shape IS the function** — a key's
  wards (`carried.keys`, every non-`keycard` entry), a seal's token;
- **undercut** — something stands out of it (more than one bill line);
- a living body leaves a **likeness** (`TemplatePaths.likeness`) and
  NOTHING else — ⚠⚠ *a cast of a face is a likeness, never a disguise*: no
  belief is written anywhere, and the test asserts it.

A carved **pattern** (`trade-carpentry` `Pattern`) overrides it with the
impression it was CUT to carry — the thing's shape, oversize by the metal's
shrink (below).

**A cast from an impression** is the impression's kind in the poured
material: one piece (`setBill(null)` — its look says *It is one piece*),
`shortDescription` *"pewter cast of an arming sword"*, seams and no maker's
mark in its prose, *"cast from an impression of Hollin's arming sword"*,
and `Tangible.castFrom = {kindPath, takenFromIdentity, takenAtS}` — the
provenance, recorded and never judged. Name, mark, grade, seat and history
never copy. ⭐ A **key** cast gets `addKey` for each carried keyway — so a
wax impression of a key, cast in brass, opens the lock. Its **mass** is the
cavity's volume in the poured metal, shrunk by that metal's own shrink; the
metal that overfilled the cavity is broken off as a *runner and riser* lump
beside it — never lost. (A slip's overfill is simply poured back.)

**Refusals, in the medium's own words:** `not-impressible`,
`already-impressed`, `occupied` (it holds a pool or a cast — except plaster
still curing), `no-shape`, `too-big` (the pattern's volume over the
cavity's capacity), and ⭐ **`undercut`** — a shape with something standing
out of it will not draw out of a **one-piece reusable** mould; a consumed
mould (broken to release) or a mould of **two halves** (a `bill` — the
platform `Mould` composes `Assembled`, so a two-piece mould is worn and
mended at its joint like any assembly) takes it. The plaster case is the
shipped two-piece.

### The shrink, and the pattern (D23)

`Material.castShrinkage()` = `thermalExpansion × (meltingPoint − 293)` —
DERIVED, never a table: pewter about half a percent, bronze and brass about
a sixtieth, against the pattern-maker's rules. A cast taken straight off
the thing comes out *"a shade smaller than what it was taken from"*. A
pattern carved for the metal (`carve pattern of <thing> from <board> for
<metal>` — the board or blank plans it, `PatternStockMixin` in carpentry's
`lib/`) holds the impression scaled `(1 + s)³` with `allowance: s`, and
casts TRUE. It must be cut from SEASONED stock: green wood shrinks after it
is cut, so it is refused in those words. The pattern is graded like any
carving, by the weakest link, so a checked board makes it no better than
fair.

## How a thing sets — the MATTER's business

| process | the material's fact | driven by |
|---|---|---|
| **freezing** | `meltingPoint` + `latentHeatOfFusion` | the thermal reconcile — `Thermal.reconcileBulkPhase` asks `castInto` when the pool sets (passing `temperatureK`, the set edge) |
| **draining** | `Material.drainSetting { days, becomes?, drains?, moisture, loadRef, absorbedByMould? }`, read as a `DrainSetting` value class (`loadRatio`, `ratePerDay`, `moistureAt`, `drainedKg`, `getBecomes` — the cure's `Kinetics` shape, B104) — curd, slip, hard soap curd, latex | `MouldMixin.reconcileSetting` integrates it |
| **curing** | the material's `kinetics:` set (`Kinetics.poolSet()` — a point curve with a ladder) | `MouldMixin.reconcileSetting` — no act, no load; the rate at the mould's temperature |
| **crystallizing** | `Material.solubility` + `crystallizesTo { product, residue, productAs?, drainDays?, metastableRatio?, purity?, minCrystalPurity?, gradeByPurity? }` (on the SOLUTION) — a sugar strike, salt, alum | the thermal reconcile — the crystallize arm of `reconcileBulkPhase`, once something starts it (a seed, a stir, deep cooling, the drying edge); in a mold the crop sets in its shape (`castInto`), its material the shape's (`productAs`), and grows; the spent liquor runs off by the one drip (`BulkableApi.drip`), over `drainDays` when the row says ([thermal.md](./thermal.md) § Crystallizing) |
| **release at once** | a plastic charge with green strength in a `pressKind: press` mould | `MouldMixin.pressRelease()`, when the load beats its stiffness |

### ⭐⭐ When a set advances — banked at its own changes, finished on its own wake (crowded-rooms D8)

A set's progress is integrated only at moments that do not depend on who
is reading: **the mould's own changes and its own wake.** Reading a mould —
`look`, `look in`, `get … from`, a card, the room's light or smell walk —
writes nothing (`setState()` is pure; there is no `getContents` override).

- **The bank** (`reconcileSetting`) integrates from `setStamp` to now at
  the **stored** `setLoadN` — the load the stretch has been under — then
  stores the load as it is now and re-arms the wake. It is called by the
  mould's own changes: a pour (`receivedPour`), something set on or taken
  off it (`placedChanged`), being set on or off a host (`placementChanged`),
  a move (`onMoved`), something in or out (`onContainableAdded` /
  `onContainableRemoved`), a press turned on it (`ScrewPress.turnBy`), and
  the wake. Because the load is stored, banking AFTER a change is exact: a
  weight set on the follower halfway counts from that moment, whoever reads.
- **The wake** (`wakeAtS`, persisted; `setWakeAtS` re-arms
  `WorldClockApi.at(… , { tag: 'mould-set' })` — the `Comminuting.
  pendingGrind` shape, so a restart re-arms and a wake already past fires at
  once) is armed for when the set will be through at the current rate,
  never more than **one game hour** away (`MOULD_SET_STEP_S`, owner ruling:
  a constant, not a dial — it bounds the error from the continuous inputs,
  a mould drying and the cure's temperature). A cure's finish is found off
  its own kinetic.
- **Completion happens only in the wake** (`finishSet`): it checks
  `assertCanRelease` first, so a one-use mould standing nowhere throws
  `DetachedError` synchronously to the clock's guarded root with the pool
  untouched, and is retried at its next change (being set down). The cast's
  async failure is reported (console + `DiagnosticApi`).
- **A freezing pan** is the thermal reconcile's; the bank keeps its
  temperature read, so a pan below its melting point freezes at a wake.

Two hoops poured together, one looked at every game minute and one never,
hold identical state and cast in the same minute (`Mould.test.ts` §
*a set finishes on the clock*).

### The drain set

Progress runs at `(1 + min(1, ratio)) / days` per game day, where `ratio =
loadN / (poolKg × g × loadRef)`. The set thing's moisture is `unloaded +
(loaded − unloaded) × ∫ratio` up to the reference load, and ⭐ **past it,
`loaded × ratio^(−mould.overpressExponent)`** — a screw press makes a
firmer cheese than a stone can (the 12 kg stone on a 2 L hoop is ~2.9× its
reference, so it too presses a little past `loaded`). Its mass is the
pool's less the water driven out (the material's `water` nutrient, else a
`/water` composition share); its material is `becomes`.

⭐ **A sugar strike is NOT a drain set** (matter-model-overlaps D12): it
crystallizes, and its molasses is the mother liquor the crop leaves — the
phase engine's arm, which drains it out of the cone over four days. The
drain set's `becomesAs` (the strike's shape → material map) moved to
`crystallizesTo.productAs`, and `drainPerBand` (a poorer strike gives up
more) is gone: a poorer strike gives up more because there is less sucrose
in it to set, by the purity mass balance.

⭐ **The absorbed drain — slip casting (D24).** A drain marked
`absorbedByMould` sets only in an **`absorbent`** mould, and the MOULD is
the sink: the water driven off goes into the mould's own wetness
(`Wet.wet`), and the pace scales by `(1 − wetness)` — a wet plaster mould
drinks slower until it dries. In anything that does not drink, slip just
sits there. The cast is the cavity's kind in clay (`slip`'s `becomes`), at
leather-hard moisture.

### Where the drainings go — the one drip (W8; matter-model-overlaps D10)

What runs out of a mold goes where anything that drips goes: to what is
under it, by `BulkableApi.drip` over `Containable.dripSlots()` — the vessel
the mold stands IN, then the vessel it stands ON (its placement host), when
either holds liquid in an interior, then the surface of the floor
([spatial.md](./spatial.md) § What is under it — the drip). A vessel
already holding a different liquid is skipped. ⭐ Nothing runs away any
more: a hoop on a bare board pools its whey on the floor, a cone lifted off
its pot pools its molasses there, and a floor already wet with something
else lets the rest soak into the ground (said on the drip's result). A mold
standing over nothing at all — no vessel and no floor — lets its drainings
run away, reported `lost` on the drip's result (never a throw: a drip is a
read's consequence). Shipped consumers: a sugar cone on its curing pot (the
crystallize arm's timed drain, below), and the salt pan on its **bittern
tub**.

⭐ **A mold's foot is open** (`passesDripsThrough() → true`): what drips
off a thing set IN a mold — molasses off a clayed loaf in its cone — runs
on to what the mold drips to, never into the mold's own interior.

The crystallize arm ([thermal.md](./thermal.md) § Crystallizing) is the
first caller: a pan whose brine has set its salt runs the spent liquor —
swapped to `crystallizesTo.residue`, bittern — down into the tub, up to the
tub's room; a tub already holding a different liquor is skipped, and what
the tub cannot take — or all of it, with no tub — runs to the ground. A pan
still concentrating keeps its brine. ⭐ A sugar cone keeps its molasses in
the pool and lets it run over the row's `drainDays` (four), a share each
read and the rest on one wake at the due time (D13); its grade is the one
read off its purity, not a band drop.
⭐ The `/trade/quarrying/thing/SaltPan` (`MaturingMixin` over the kernel
`Mould`, the evaporative `brine` profile taking the water) stands on a
`/trade/quarrying/thing/Tub` (`PlacingMixin` over `Vat`) at the Rejection
flat and the Terminus salt house — a prop `{ template: …salt-pan, on:
…bittern-tub }`. ⚠ The trade-sugar pack ships the same `PlacingMixin(Vat)`
shape as its `CuringPot`: two packs with no common ancestor wanting one
substrate is the signal to promote it to the kernel at the merge.

### The cure (S17)

A pool whose material authors a kinetic set — the first `kinetics:` entry
that is a point curve with a ladder (`Kinetics.poolSet()`; a dose law is
heat-work, the kiln's) — advances it at the mould's temperature, whatever
presses it, and sets into its rung's material (review R2-D3: `cureSetting`
was a second copy of the one heat-and-time evaluator). Plaster slurry (`stir` burnt
gypsum and water in the plaster tub) cures in a quarter of an hour into
**cast plaster** — and, poured into the **plaster case** with a model
pressed in before it sets, comes out as a **plaster mould** carrying the
model's shape: ⭐ *a mould that casts a mould hands it the cavity*
(`castInto` sets the impression on a cast that is itself a mould, and the
case is ready for the next).

Gypsum burns to plaster by the ONE heat rule (`rock/gypsum`'s
`kinetics` → `mineral/plaster`, ~420 K for an hour): the foundry's
**core oven** holds that gentle heat.

## ⭐⭐ The envelope (S12)

| property of the mould | effect |
|---|---|
| **conductivity** | the **chill**: `Material.chillsTo [{mouldConductivityAbove, becomes}]`, first match by the MOULD's thermal conductivity — cast iron sets WHITE in an iron chill mould (80 W/mK), GREY in sand |
| **absorbency** (`absorbent`) | drinks a slip's water; drinks plain water poured into it (`receivedPour`) — so a watered flask is a damp flask |
| **wetness** | a hot pour (> 373 K) into a mould wetter than `mould.blowWetness` (0.3) **BLOWS**: the cast is a *blown lump* carrying a visible `blowhole` flaw (its look says so; the flaw caps the grade at poor, B104 — it was a baked grade and baked prose), and the pourer is burned — `afterPouredInBy` (a `Bulkable` `@hook` the pour verb calls) inflicts `heat` on the hands through `ConditionApi`, so a glove takes the edge off |
| **melting / ignition point** | **survival** (S13, below) |
| **effusivity** √(kρc) | how fast it draws heat — the misrun's scale (below) |

A freshly rammed sand mould holds `mould.rammedWetness` (green sand, 0.15)
— damp enough to hold a shape, dry enough to take a pour.

## ⭐⭐ Every pour obeys its receiver (S13)

`BulkableLogic.transfer` asks the receiver **before anything moves**, for
EVERY vessel: `BulkableMixin.canReceiveAtK(pourK)` → `holds` · `melts` ·
`burns` (its `meltingPoint`, then its `autoignitionTemperature`). On
`melts` or `burns` the pour goes to the floor (the shipped puddle path) and
the receiver is gone — melted into its material's `brokenTemplate` where it
stood, or burning (through its own `light` when it is fuel; else it chars
away). Status `melted` / `burned`, and the pour verb says what happened to
the vessel. Lead melts a tin cup; bronze burns the chandler's pine soap
frame; pewter into a bronze cup holds. Interior pours only — a floor is
never a receiver.

## ⭐⭐ The pour fails by Chvorinov's rule (S14)

`receivedPour({temperatureK, litres, material})` — called by the transfer
after the thermal coupling — records the metal's **superheat** over its
melting point. At the set, for a metal in a **closed cavity**
(`closedCavity`, or any mould holding an impression — an open ingot trough
is its own riser and is judged on neither):

- **misrun** iff `superheat < mould.misrunKcm × chill ÷ modulus`, the
  modulus `V/A` in cm (a shape whose impression carries `lengthM` reads as a
  prism of it, a blade — a `Weapon`'s does, B104; until then a sword's
  `Quantity` length was read as `NaN` and every blade judged as a cube;
  anything else as a cube) and `chill = effusivity ÷
  mould.sandEffusivity`. An arming sword in sand wants ~35 K in hand; a
  bell, a few. A pour that ran short of the cavity is a misrun too. The
  cast carries a visible `misrun` flaw (its look says *it froze before the
  mould was full*) and grade poor.
- **shrinkage** iff no `riser`: porous where it froze last — a visible
  `shrinkage` flaw (which buzzes when `ring`ed) and one grade down. ⭐ The
  flaw is the fact (review R2-D7): its words are the flaw augmenter's,
  never baked into the cast's description. `carve riser
  on <flask>` (any edge — `MouldMixin` is a `CarvableMixin`) cuts one
  into a rammed mould; the bell pit authors `riser: true`.

⭐ Nothing is rolled. A competent founder reads it in advance — the melt's
heat through `analyze temperature <crucible>`, the flask's damp through
`analyze wetness`.

## Pressing — one load from three sources (S16, D25)

`Mould.loadN()` = what is placed **on** the mould × g, plus the
**screw press** it stands on: what its placement host bears down on it
(`Placing.bearingLoadN(item)`, B104 — `0` for a shelf, the platen for a
press; it was a duck-probed `pressForceN()`), so any press a pack ships
loads a mould the same way. `ScrewPress`
(`platform/thing`, `PlacingMixin(Good)`, `newtonsPerTurn`, `maxTurns`,
`turns`) is a **`TurnableMixin`** (`lib/craft/Turnable.ts`, which also affords the verb) that
the platform verb **`turn`** (`cmd/device/turn.yaml`;
`turn back <press>` slackens it) asks — so a windlass can take the word
later. A turn first banks every mould's set at the OLD load, then adds
force, then asks each mould on the bed to `pressRelease()`.

**Shape-and-release** (`pressKind: press`): a Moist solid charge (not
the mould's own cast) whose moisture is ≥ 0.05 resists with `mass ×
mould.pressNPerKg × (1 − moisture)²` — wet clay gives, stiff clay wants
turns. When the load beats it, the charge (up to the cavity's volume — the
rest stays a lump) is cast in the mould's shape, carrying its composition
and its water, and **leaves the mould at once** to set outside it. The hoop's
curd stays: it is bulk and drains.

**The blow** is the die's, not the mould's: `hammer <blank> on <die>` —
`trade-smelting`'s `Die` is a `Tool` offering `anvil` (so the hammer's
binder takes it) and a `StrikingDieMixin` (the kernel's, `lib/craft/StrikingDie.ts` — smithing's
controller cannot name smelting's class) that answers `strikeWith(blank, blowJ, by)`: the blow's
force (its energy over a 2 mm stroke) against the blank's hardness over a
token's face, eased as it nears its melting point. The token carries whose
die struck it as `castFrom` provenance.

## Consumed moulds

`reusable: false`: the cast is moved to the mould's scope, the mould is
destructed, and ⭐ it leaves **`breaksTo`** — the sand flask its heap of
moulding sand, the wax block the lump of beeswax caught from its vent. ⚠
The wax block's MATERIAL is its clay jacket's: what receives the pour is
the jacket, and a bare block of wax would melt under the metal (S13).

`reusable: true` wears by `wearPerCast`; a **cracked** mould casts the
material's own lump.

## Slow pours (D26)

`pourDurationS > 0` (the **bell pit**, six game hours): the metal still goes
in at once — it would freeze in the crucible otherwise — and the founder's
day is the WATCH: `MouldMixin.watchThePour` holds the pourer's attention on a
sealed-down `ManualBuildStep` for that long, and the mould's discipline is
credited (`hard`) when the watch is kept. Every other pour is unchanged.

## What rides onto the cast

The pool's payload is carried onto a cast that can hold it: ⭐ the batch's
`maker` and `craftedAt` (the payload's — the fallback to the mould's own
maker is gone: who made the mould did not make what set in it), the grade
**`min(payload.grade, mould grade)`** (matter-model-overlaps D3 — a fine
pour into a poor mould casts poor; being a Mould is the claim that passes
its own grade, and either side ungraded leaves the other's), the mould's
`discipline`
on the one cast record (`castFrom.discipline`, beside any impression it was
taken from — repair D1, review R2-D6), freshness, pathogens, a set's moisture,
`alloying` (ceramics ruling C), `madeFrom`.

**Which material the cast is** (`SetPool.stampMaterial`): a drain set
stamps `becomes`; a crop stamps the shape's material
(`crystallizesTo.productAs` — one strike sets as loaf sugar in a cone and as
raw sugar in a hogshead's block — else `product`); a freeze stamps the pool's
material when the pool is not `liquid`-tagged (one ingot row serves
copper, tin and iron) and otherwise keeps the shape row's own (an ice
block is ice, never water).

## What composing it claims

*A made, wearing vessel that holds what set in it.* Requires `Bulkable`,
`Thermal`, `Container`. ⛔ Not every vessel is a mould. `MouldMixin`
is an `ImpressibleMixin` and a `CarvableMixin`, which afford `impress` and
`carve` (environment + peers) — a mould cut to its
own shape answers both with its own refusal.

⛔ **Forming over a form is not moulding** (the moulding slate's § 2
ruling, 2026-10-16): a shoe on a last, a hat on a block, boiled leather,
steam-bent wood, slumped glass are material worked OVER a shape, not
matter setting IN a cavity — there is no container, so nothing composes
`Mould`, and each is its own trade's act (cobbling, millinery, cooperage's
bending). Pressing IS moulding, because the cavity is the shared thing (M2).

| host | composition | rows |
|---|---|---|
| `platform/thing/Mould` | `Mould(Assembled(Durable(Crafted(Thermal(Bulkable(Container(Good)))))))` | `ice-pan`, `ingot-mould`, `chill-mould`, `flask`, `wax-block`, `bell-pit` (smelting), `plaster-case` (two-piece), `plaster-mould`, `tile-mould` (ceramics), the rubber moulds, the soap frame |
| `trade-dairying` `Hoop` | `Placing(Mould(Durable(CraftVessel)))` | `hoop` |
| `platform/thing/ScrewPress` | `Turnable(Placing(Good))` | `tile-press` (ceramics), the creamery's `cheese-press` |

## Where it stands

- **Rejection's foundry** (east of the smelter) — flasks, wax blocks, the
  chill mould, the die and token blanks, the core oven, the plaster and slip
  tubs, the plaster case, the tile press; **the casting yard** north of it
  with the bell pit under a plank roof (rain on a loam mould is a pit that
  blows); **the founder** (`founding` proficient), the foundry business.
- **The curfew bell** hangs on the Terminus market square; anyone rings it
  (`ring`, the `Bell`'s own `ringSound`).
- **The creamery's cheese press**, beside the stone.
- ⭐ **Founding is `trade-smelting`'s discipline**
  (`/trade/smelting/idea/Discipline/founding`), credited by the flask, the
  chill mould, the wax block and the bell pit, which ship in that pack with
  the die and the moulding sand; the plaster case, the plaster mould and the
  tile mould credit `pottery` ([ceramics.md](./ceramics.md)). Not its own
  pack: a foundry is where the smelter's metal goes next, so the smelter's
  pack owns it, and a second foundry is rows. (Closes the moulding slate's Q4.)

## ⭐ The mould keeps its drainings (sugar, 2026-10)

A drain set no longer throws away what it drives out. `DrainSetting`
gains `drains` (what the drained matter IS, e.g. whey off a curd). (The
sugar strike that first used it crystallizes now — matter-model-overlaps
D12 — and `drainPerBand` went with it.) The drainings run to what is
under the mould (the one drip, above) **as the set progresses**: each step
drains `total × progress`, tracked in `setDrainedKg`, and the remainder goes
at the set.

⭐ **The drainings are a transfer through the set** (re-plan R1-D8):
`BulkableApi.drip({ from: pool, litres, via: { through: 'set', arrivesAs,
grade } })`, which is a `transfer` per slot under it, the one move
([bulk.md](./bulk.md) § `TransferVia`). The population and the
dissolved go with the liquid, the maker goes with it, the solid's identity
does not, and the drainings are their own material at its own strength. The
grade — on the drainings' PAYLOAD (D2) — is one band below what the pool
sets at (`min(payload.grade, mould grade)`, the cast's own), never above
what the vessel below already held (the grade rides `TransferVia.grade`,
and a top-up takes the lower). What
that vessel has no room for — or all of it, when nothing is beneath or the
vessel holds a different liquor — runs on to the floor. ⚠ The pool is
therefore DEBITED as it sets: the set reads the
charge it started from as what is in it plus `setDrainedKg` at the
drainings' density, and the cast's mass is that charge less what drained.
Consumers: a cheese hoop standing on a vessel (both curds author `drains:
whey`); a mould on a board pools its drainings on the floor. (A sugar cone's
molasses is the crystallize arm's mother liquor, above.)

## ⭐ Percolation — a liquid through a porous solid (sugar, 2026-10)

`PercolatingMixin` (`lib/material/Percolating.ts`) composes on a porous
SOLID. Liquid is put on it two ways. ⭐ It is a **pour target**
(`PourTargetMixin`; matter-model-overlaps D10b): `pour pail onto heap`
takes the liquid out of the vessel and the heap keeps what it is, its
`percolant` — `{ material, payload, litres }`, persistent, its material
loaded by the restoring setter `setPercolant`. One liquid at a time: another
kind poured on while one is working through answers `mismatch`. Or
`receivePercolant(litres)` puts on it the liquid it already lies in (a
hopper's liquor, a lump of wet clay's water) — `material: null`, carried out
in place by the host. The **reach** (`0..1`) advances over the host's
`percolationDays()` while liquid is working — banked at the heap's own
changes (a pour, `receivePercolant`, `onMoved`) and finished on a game-clock
wake at its due time (`percolationWakeAtS`, re-armed by
`setPercolationWakeAtS`; crowded-rooms D8) — and each advance calls the
host's `@hook onPercolated(from, to)`. ⭐ A POURED liquid comes out of the
bottom as the reach advances — the share of what is left that this stretch
lets through — as the host says it left (`@hook percolateOut`), by the one
drip (`BulkableApi.drip({ off: heap, … })`): into the vessel the heap is
in, else the one it stands on, else onto the floor. A clayed sugar loaf
(set by the crystallize arm — the claying is percolation, not the set) is
the first consumer: its head washes white past a third of the reach,
its middle past two-thirds, its foot when the water is through, and
the molasses it carries out drips — through its cone's open foot into the
pot below, else onto the floor. ⭐ Ash is the second (cordage W7, potash):
in a hopper the hopper's own liquor is put on the heaps by amount and each
heap enriches the liquor it lies in (`dripSlots()[0]`) by reach — unchanged;
watered where it lies, a heap answers **lye** (`percolateOut`: the row's
`leachesTo`, carrying the alkali that stretch gave, its `LyeStrength` band as
`payload.grade`, whose it is decided at the pour by the communal-oven rule —
`percolantPoured`), and on a bare floor that is a puddle of lye. Either way
it is spent ash at the bottom, where it lay. A vessel's seeper reconcile
brings current whatever drips into it — the same chain read from the
receiver's side — when that vessel is deliberately read; a heap comes
through at its due time on its own wake, read or not.

## Deferred

- **A mould that is a place** — a pit you climb into (S19; a Location
  composing `Mould` needs `Bulkable + Thermal` on a place). Recorded under
  the plan's *needs the owner*.
- **Dead-burnt plaster** — gypsum burnt past ~700 K never sets; the mixing
  vessel names the blend whatever went in, so the honest version needs the
  cure to read the blend's composition. Not shipped (no false claim).
- **Bake and dry** as set processes on `reconcileSetting`.
- **The sand's re-tempering** (`knead` the heap back to green sand).
- **Glass press- and blow-moulding** — rows plus a set process.

## ⭐ Failure policy at the set (B104)

The set path follows *refuse, skip or throw*
([spatial.md](./spatial.md) § Writes on a detached subject):

- A **one-use mould standing nowhere** cannot break to release its cast:
  `castInto` throws `DetachedError` before anything is minted, and the
  setting reconcile checks the same thing before it empties the pool.
  (It used to destroy itself and leave the cast in no place at all.)
- A **runner with nowhere to go** is the same throw.
- A **cast row that will not mint**, or is not a tangible, visible thing,
  throws a `TypeError`; so does a `breaksTo` row that cannot be left. (A
  failed mint was a console warning, and what set was lost.)
- A drain or cure set completes on its WAKE (crowded-rooms D8), never
  inside a read; the release check throws to the clock's root before the
  pool is touched, and the cast's async failure is reported. The thermal
  freeze still completes inside its reconcile and logs a failed cast.
- `pressRelease` makes the cast before it spends the charge.

## History

- **2026-11 (crowded rooms, MR !598)** — a drain or cure set was integrated
  by whoever READ the mould (`getContents` was overridden to reconcile), so
  a mould looked at every minute set at a different moment from one left
  alone. It now banks at its own changes at a stored load and completes
  only on its own game-clock wake (§ When a set advances). Found on the
  way: the shipped dairy `Hoop` (`PlacingMixin(MouldMixin(…))`) hid the
  mould's own placement hook, so a weight set on a hoop never counted
  until the next read; the placement hooks now chain.

- **2026-11-20 — matter-model overlaps (MR !644).** Drainings leave by the
  one drip (in → on → the floor; over nothing they run away, reported
  `lost`), one band under the pool's grade on the PAYLOAD; `drainsInto` and
  `bandsBelowFine` went. A cast reads `min(batch, mould)` (`castGradeOf`),
  so a poor mould lowers it. Sugar's drain set (`becomesAs`, `drainPerBand`)
  retired to `crystallizesTo.productAs`/`drainDays`.
