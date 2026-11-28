# Faces — a good has an outside, a vessel an inside, and a face carries coats

Every good has an **outside**, and a good that holds bulk also has an
**inside**. Each face carries a list of **coats**, outermost first. Shipped by
the ceramics build (W1, 2026-10, plan D5–D7, requirements M6–M8), with three
consumers on day one: the **glaze** on a pot, the **lead leach** from an
inside coat into the contents, and the **weeping jar** that cools its water.

Source: `packages/server/src/mud/lib/material/Faced.ts` (`FacedMixin`, the
`Faced` interface, `CoatLayer`, `FaceName`, `CoatState`), composed on
`lib/stuff/Good`. The bulk reads (the face exchange: leach, weep, memory) live on
`BulkableMixin`; the cooling read (`evaporativeDepressionK`) on `ThermalMixin`.

## ⭐⭐ One shape for a wall and a pot

A coat is construction's layer: `EnclosureSpec` (`{material, thicknessM}`),
the shape a wall or a roof is authored in ([construction.md](./construction.md)),
plus the `state` a firing gave it. So a building's face and a pot's face are
one idea. ⚠ But `Faced` composes on **`Good`, not `Thing`**: a wall, a floor
and a fixture take paint through construction's layers. Composing it on
`Thing` would claim a coat on every floor.

State-free until a coat is laid (`coats: {}`). Every good can answer the claim
it makes, *"it has an outside, and something may be laid on it"*, which is the
test for composing on the base — glaze, paint, tar, varnish and wax go on pots,
carts, signs, tools, hulls and cheeses alike.

⚠ **The claim is about the GOOD; the verb is the BATH's.** Being dippable is not
a fact about a good, so `Faced` affords nothing: `dip` is afforded by the vessel
you dip into (`GlazeTub` — the glaze and slip tubs; chandlery's dip pot;
glass's melt). ⚠ Not every compounding vessel: the temper pit, the batch pot,
the rubber tub and the plaster tub mix and are not baths, so they do not
afford `dip` (B101). It was on every good for a while, and every good in reach offered `dip`
(ceramics review, 2026-10-28).

⭐ **Coats are a per-piece record**, so a coated stack does not merge with a
bare one: `StackableMixin.canMergeWith` compares a coat signature (material and
state, by face) beside Thermal's fired signature — identity fields are scalars
compared by `===`, and a coat list is not one.

| method | answers |
|---|---|
| `faces()` | `['outside']`, plus `'inside'` when the good holds interior bulk |
| `getCoats(face)` | the coats on a face, outermost first |
| `outermost(face)` | the material that meets the world: the top coat's (unless crawled), else the body's |
| `layCoat(face, material, thicknessM, dirtyAtLay?)` | a WET coat, outermost (`dip` calls it) |
| `heatCoats(integrate, peakK)` | advances every wet coat by HEAT-WORK over a window and fuses the ones that get there (B100) |
| `fuseCoats(peakK)` | fuses every wet coat at once — only for a coat the heat itself lays (a vapor glaze) |
| `holdsWater(face)` | a sound coat of a tight material, or a tight body — the FACE only; a crack is not its question (an open crack or hole leaks by the one hole rule, `Flawed.flawLeakLPerHour`, summed in `Bulkable.leakLPerHour`) |
| `isPermeable()` | the wall's half of a weep: NEITHER face holds water — a sealing coat on either face stops the pores (matter-model-overlaps D4); the bulk exchange asks this from inside its own reconcile |
| `porosity()` | the body's open porosity, `0..1` |
| `cleansAs()` | how hard it is to get clean, `≥ 1` |

## A coat is laid wet and set — a glaze, by heat

`dip` lays a coat **wet** from a bath whose material `laysCoat`. The coat is
**set** — a glaze by heat: the coat is not a Stuff, but it has a PROGRESS of its own
(`CoatLayer.progress`), its material's **first kinetics change** advanced by
HEAT-WORK — the same integral the body under it fires by (`heatCoats`, called
from the host's thermal reconcile with ∫ rate(T) dt over the window). It fuses
when that reaches the change's first rung. ⭐ So a glaze that only touches its
heat for a moment does not fuse, and the glaze, the body and the cones in the
spy-hole all read one heat-work (B100 — it used to fuse on the window's peak
alone). The window's peak decides only HOW it came out: past `scorchAboveK`
it runs. A coat whose material has no kinetic law fuses at its melting
point. At the fuse a coat becomes:

| state | when |
|---|---|
| `set` | sound: it wipes clean, and holds water if its material is tight |
| `run` | fired past the glaze's `scorchAboveK`: it runs to the foot and pools |
| `crawled` | laid over dirt (`dirtyAtLay` > `faces.crawlDirt`, 0.3): it pulls back off bare patches, and a crawled coat does not count as the outermost |
| `crazed` | the coat's `thermalExpansion` exceeds the body's by more than `faces.crazeMismatch` (1.5 × 10⁻⁶ /K): it shrinks more on cooling, cracks, weeps a little and holds dirt |

⭐ **The words are the coat material's, not the mixin's.** A set coat reads
*"It is `coatWord` on the outside with …"* — the material row's `coatWord`
(`glazed`; absent reads `coated`), so a paint row says `painted` and no
look line changes. A wet coat says *not yet fired* when its material sets by
heat, *not yet set* otherwise. The faults (`crazed`, `run`, `crawled`) are
every coat's: paint crazes, crawls over grease and runs too.

⭐⭐ **Paint extends this, not beside it** (paint-slate).
What the paint build adds to `Faced`: setting by the coat material's BINDER
(carbonation, oxidation, evaporation — heat is the glaze's case), WEAR (a coat
is the sacrificial layer that weathers so the body does not), a thing's COLOR
read from its opaque outermost coat over its dyed fiber over its body, and one
coat stack shared with construction's wall layers so a wall and a cart weather
by one rule.

The glazes are rows: `/stuff/idea/material/glaze/{lead,ash,salt,tin}` (commons rows in base-library — R4: the crocks that wear them live in packs that do not depend on trade-ceramics).

| glaze | fuses | runs above | expansion /K | notes |
|---|---|---|---|---|
| lead | 1250 K | 1400 K | 7 × 10⁻⁶ | crazes on earthenware; leaches lead |
| ash | 1500 K | 1600 K | 5 × 10⁻⁶ | a stoneware glaze; gives nothing |
| tin | 1250 K | 1400 K | 6 × 10⁻⁶ | opaque white; a lead glaze, so it leaches too (more slowly) |
| salt | 1350 K | — | 5.5 × 10⁻⁶ | never dipped (`laysCoat: false`): vapor in the kiln lays it (ruling B) |

## Holding water, and porosity

`porosity()` is **derived, never authored**: it is the fired body material's
water absorption (`waterAbsorptionCapacity` / 100) for a `fired` or `porous`
material. The firing swaps the material (terracotta 15 %, earthenware 8 %,
stoneware 1.5 %), so porosity follows the heat-work without a field of its own.

`isSealed(face)` is true for a set (or run) top coat whose MATERIAL is tight
(its own water absorption at or under `faces.holdsWaterBelow`, 0.03 — every
glaze is 0; a limewash would not be). `holdsWater(face)` is `isSealed(face)`
or a body at or under the same dial. So stoneware holds water bare,
terracotta weeps, and a glazed crock holds. `isPermeable()` — the pores'
half of a weep — is a porous body with NEITHER face sealed: a coat on
either face stops the flow through the wall.

⭐⭐ **One leak** (matter-model-overlaps D4). ⚠ A crack is no longer a face's
question: it is a FLAW (`FlawedMixin`, the record split out of
`IntegrityMixin`), and a cracked glazed crock leaks at the CRACK's rate
while its faces still hold. What a vessel loses is one read on the thing
that holds, `Bulkable.leakLPerHour()`, summing three terms:

- **pores** — a Faced vessel whose wall `isPermeable`: `porosity ×
  thermal.evaporation.seepLPerHourPerL × held` (the weep, below);
- **flaws** — `Flawed.flawLeakLPerHour(∛(held/1000), 1)`, the ONE hole rule
  for every crack and hole: `flaws.leak.<kind>LPerHour × √head × scale`
  (crack 4, holed 640 L/h — a hull passes her draft and her length,
  [ship.md](./ship.md));
- **joints** — `Assembled.jointLeakShare() × held` (a sprung stave,
  [assembly.md](./assembly.md)).

`Bulkable.isLiquidTight()` asks the same three at capacity, as a share of
capacity an hour, under `faces.holdsLiquidBelowSharePerHour` (0.05);
`getClosure()` is `open` when it is not, so EVERY vessel — not only a cask —
spills a pour it cannot hold ([bulk.md](./bulk.md) § Closure scale). What a
vessel already holds drains on read in the wall's exchange, moved by the one
drip (the pores' share `through: 'pores'`, the rest plainly), and a vessel
losing liquid arms ONE game-clock wake (`leakWakeAtS`, persisted and
re-armed by its setter) at the computable moment it runs out
(`flaws.leak.dregsL`, 0.05 L) or the vessel under it brims.

## ⭐⭐ The wall's exchange — one model, two sources (M7; matter-model-overlaps D11)

What a vessel's inside wall does to what it holds is ONE call,
`Faced.exchangeWith({ material, litres, payload }, dtS)`, which returns the
contents' new payload. `BulkableMixin.reconcileFaceExchange` keeps only the
clock (on an interior slot's read, at most once a game-minute, stamped
`faceCheckedS` on the payload), the call and the writes — plus the leak,
which is the slot's ([bulk.md](./bulk.md)). It first asks
`wallExchanges(contents)`, so a wall with nothing to give or take writes
nothing: a glazed pot of water carries no payload.

Inside the exchange, in order:

1. **The leach** — always, below. The inside's OUTERMOST material (a coat,
   else the body) leaches by the contents' acid, sealed or not: a lead glaze
   is a seal that poisons.
2. ⭐ **A sealed inside stops the rest** (`isSealed('inside')` — a set
   glaze, a pitch). A pitched ale cask gives its ale nothing.
3. **The wall's own material.** The GIVING LAYER is the inside's top coat
   when there is an unsealed one, else the body. It gives its material's
   `imparts` (`ExtractionMixin`, mg/L at full give) × `1 − spent`, toward
   that target, linearly over `faces.wallGiveDays` (90) of contact — linear
   and capped, so a read every minute and a read a month apart land the
   same. Only a liquid takes it (extraction wants a solvent). `spent` is per
   layer: `wallSpent[face]` for the body, `CoatLayer.spent` for a coat.
   `noteFill(face)` — called by `BulkableLogic.transfer` on a transfer into
   an EMPTY interior, the fill DECLARED by the act — writes `0` on a layer's
   first fill and adds `faces.wallSpentPerFill` (0.34) on each after, so a
   fourth fill gives nothing. `wallGiveLeft(face)` is `1 − spent` (`null`
   when the layer gives nothing), and `look` words it: *"has less left to
   give"* at half, *"has given its contents everything it had"* at none.
4. **What the wall has absorbed** — the crock that remembers the pickles, the
   cask that makes water taste of whiskey. A body tagged `absorbent` (oak),
   or porous past holding water bare (terracotta), unsealed, first GIVES
   BACK toward what it holds — counting the held material's own smell as
   already there — and LOSES what it gives (the two meet halfway, at a
   game-day's rate), then LEARNS `faces.aromaRetained` (a tenth) of what the
   contents give a nose: their dissolved aromatics AND the held material's
   own `smells`, converted at each aroma's threshold (`strength × threshold`
   = mg/L). A tight stoneware body and a glaze keep nothing. A wash spends
   it (`washAbsorbed`, below); it never touches `wallSpent`.

**The re-fire** (`refireFace(face, coatPath)`, gated to `CraftingLogic`):
the face is shaved back to clean wood — its coats, its spent and what it had
soaked up go — and a fresh SET coat of `coatPath` is laid. The path is the
kind's bill's `refire:` (the cooper's char, `charred-oak`); there is no
settings path and no kernel default. A plain cask re-fired comes back a
charred one.

### The lead leach

A material may `leach` into what is held against it: `leaches:
[{toxin, mgPerLPerDay}]`, the rate against VINEGAR (pH 2.4, the acid a
lead-release test uses). `BulkableMixin`'s face exchange runs on an interior
slot's read (`getBulkAmount`/`getBulkPayload`, at most once a game-minute)
and is stamped `faceCheckedS` on the payload. It reads the holder's inside
`outermost` material, multiplies by the contents' **acid attack** and the
days elapsed, and adds the result to `payload.dissolvedToxins`. That is
metabolism's own per-litre field, so drinking it doses the drinker with no
new code.

⭐ **The attack is the contents' pH** (`Material.ph`, `acidAttack()`, B109 —
it was `acidity`, a 0..1 dial nothing could be checked against, which had
sea water faintly acid and lemon milder than vinegar). `10^((2.4 − pH)/2)`
below neutral — tenfold per two pH units — and nothing at or above 7: lead
is an acid's to take. Vitriol (pH 0) ≈ 16×, lemon (2.2) ≈ 1.3×, wine (3.3)
≈ 0.35×, ale (4.2) ≈ 0.13×, milk (6.7) ≈ 0.007×, sea water (8.1) and plain
water (7, unauthored) 0 — which is why lead glaze was trusted for so long.
`ph` is the same unit the ground's `nativePh` and the soil kit read.

⚠⚠ **No sense reports it.** `look`, `smell` and `taste` say nothing. You learn
it from knowledge, from an assay, or from the sickness. It is the honest shape
of lead glaze, and of the vinegar crock that poisoned a household.

⚠ A crock of milk in Hearts Delight's creamery and a cask in Crowsfoot
run the same exchange: what differs is the wall's material and its coats.

## The weeping jar (M8)

The pores' term of the one leak: a vessel holding a LIQUID whose wall
`isPermeable` (neither face sealed, a porous body) loses `porosity × thermal.evaporation.seepLPerHourPerL` (0.02)
litres per hour per litre held into its own `wet()`. So its outside stays wet.
`ThermalMixin.evaporativeDepressionK(airK)` = wetness × (T_air − T_wetbulb) ×
`thermal.evaporation.efficiency` (0.8) — and a vessel that is WEEPING counts its
surface fully wet, because the film is there as long as it weeps. It is taken
off the air on both sides (the trajectory the pull side reads, the ambient the
push side resolves), so the wet jar and the contents it IS read cooler than the
room: ≈ 6 K under a 300 K room at half humidity, ≈ 9 K in dry air. A
glazed jar seeps nothing and reads at room heat. ⚠ **Air only**: a thing
immersed in water evaporates nothing (underwater's immersed branch wins). One
wet-bulb formula serves the body and the jar
(`Evaporation.wetBulbK()`).

⭐ **The film has a limit** (matter-model-overlaps D10). The weep wets the
vessel's own outside up to what that wall can hold (`waterCapacityKg`), and
what the film cannot hold drips — by the one drip, `BulkableApi.drip`
([spatial.md](./spatial.md) § What is under it — the drip): into the pack
it is carried in (wetted up to the pack's own capacity), then onto the
floor. A jar left long enough on a bare floor stands in its own damp ring.

Porous ware also **keeps** what it held. Grease lodges in a bare porous face,
and a share of a fill's aromas (`faces.aromaRetained`, 0.1) stays on the
inside face and goes into the next fill: the crock that remembers the pickles.
Glazed ware and stoneware keep nothing, which is what *hygienic* means here.

## ⭐⭐ The water filter — through, when there is somewhere to go (S9, W8)

The same weep, with one question first: **is there a vessel to run into?**
The answer is what is under the body (`Containable.dripSlots()`,
matter-model-overlaps D10): the vessel it is set IN (its container), then
the vessel it stands ON (its placement host), when either holds liquid in an
interior and holds nothing but what would come through. With such a vessel
the water does not weep onto the body's outside; it runs THROUGH the wall
and on by the one drip (`BulkableApi.drip`, `via: { through: 'pores' }`),
at `porosity × faces.filterLPerHourPerL` (5) per hour per liter held — a
free flow, not an evaporation, so about 250× the weep. What the vessel has
no room for runs on to the floor. No such vessel, and the body weeps as
above.

⭐ **What comes through is what the pores let through** — `Material.filteredTo`
on the liquid's own row, else the liquid itself. The pores of a filter body
(`/trade/ceramics/idea/material/filter-stoneware`, 12 % open porosity: a
stoneware fired short of vitrifying) are about a micron, smaller than a
bacterium and far larger than an ion, so:

- **fouled water** (`filteredTo: water`) comes through as water — the
  dysentery is IN something, and it stays in the wall;
- **tainted water** (no arrow) comes through tainted — lead is dissolved;
- anything DISSOLVED rides along: the payload's `dissolvedToxins`, dissolved
  solids and hardness go through (folded by volume into what is there); the
  microbial load (`freshness`) stays behind.

A vessel already holding a different liquid is skipped: the filter does not
flow into it, and weeps slowly onto its own outside instead. The order —
**IN, then ON, then the floor** — is because a filter candle sits in the
mouth of a crock, and `ClayVessel` offers no placement; the ON rule is the
one sugar's cone and the salt pan use, so one chain serves every drip. ⚠ A weeping jar set in a basin now drips into the basin rather than
wetting its own outside, which is the physics.

The row: `/trade/ceramics/thing/water-filter` (a `ClayVessel`, 2 L, unglazed —
a glaze would close the pores) and the `filter` form, sold at the Terminus
potter's stall. Doulton's filter of the cholera years — an instrument
against [watershed.md](./watershed.md)'s contamination by kind, and its
limit is that kind.

## Cleaning

`cleansAs()` multiplies how hard a good is to wash ([cleanliness.md](./cleanliness.md)).
It is 2 with a crazed coat, 1.5 for a bare porous body, and 1 for a sound
glaze or a tight body.

## Seams

- **Paint** (paint-slate) rides `Faced` for
  goods: a coat on a cart or a sign is a coat on its outside face, weathering
  by decomposition's exposure clock. Buildings take paint through
  construction's layers.
- **Casks and lined vessels** — ✅ migrated (matter-model-overlaps W4): a
  charred cask is a cask with a `charred-oak` coat inside, an ale cask one
  with a pitch coat (which seals). A tin lining would be a coat row the same
  way; none ships.
- **Wet clothes and sweat** reading `evaporativeDepressionK`: exertion and
  textiles' deferred lists.


## Setting by the air, weathering, color (paint build, 2026-10-31)

A coat need not set by heat: `KineticSpec.sets` names carbonation,
oxidation, drying or solvent, and `Faced.airCoats` walks the air (rain runs
a wet coat, frost crazes it). A coat may remember what it was made of
(`CoatLayer.paint`, the pot's makeup — its color), what was lettered in it
(`marks`) and how much sun it has had (`faded`). `coatPlanFor` is the
`paint` verb's protocol on the thing painted; `coatAreaM2` is its face's
area; `persistsCoats` gates the coat slice. ⭐ **The weather works on every
outside** (`reconcileWeather`, folded in from the paint build's
`WeatheringMixin` at review): coats thin at their material's rate where the
weather reaches them, set by the air, fade in the sun, and a durable host's
condition wears at `bareWearPerDay` once they are gone — none for a good,
a building's dial for a building and for a ship's hull (harbors: `Ship`
composes `Faced` over `Durable`, her seams the fabric, and they leak as
they open — [ship.md](./ship.md)). ⭐ Transport's `Boat` composes `Faced`
over `Durable` too (matter-model-overlaps D7): under her tar her fabric is
her seams, worn at the house's rate once it has gone and read by the ship's
seam bands at her length (`Bilge.hullLeakLps`). The shipped boats are
tarred, so they are tight until the pitch goes. See
[painting.md](./painting.md).

## History — matter-model overlaps (MR !644, 2026-11-20)

The wall's exchange became ONE model with two sources — the wall material's
`imparts` (`ExtractionMixin`, spent per fill, re-fired as a coat) and what
the wall has absorbed (spent by `wash`); the lead leach rides it.
`isSealed(face)` is the one coat question; `holdsWater` lost its crack
clause (a crack is a flaw that leaks by the one hole rule). A boat has a
fabric now (her seams), and rain sheds off a sealed outside.
