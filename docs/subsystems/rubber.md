# Rubber — the material that keeps water and current out

> Shipped by the rubber build (2026-10). Pack: `trade-rubber` at
> `/trade/rubber`. Kernel: `lib/material/Crosslinking.ts`, `fire … for`,
> the mould seam in `FireController`, `Mould.holdsCast`. Places: the rubber
> works off the Terminus goods yards. Model of what is over a place:
> [construction.md](./construction.md).

## Supply — three inputs, nothing granted

- **Latex** — a Hevea under glass. The species row
  (`…/euphorbiaceae/hevea/brasiliensis`) taps latex on a `weather` window
  open between 291 and 309 K **all year**: a stove house has no season but
  its temperature. ⭐ The window reads the **air at the tree's place**
  (`Producing.windowAirK`), not the stem's body — a 290 kg stem lags its
  house by hours. The works' three trees stand in tubs (`hevea-tub`, a
  `PlantPot`) that somebody has to water.
- **Latex sets** by the dairy's drain set: the latex row authors
  `drainSetting` (a day, → raw rubber) and its water share; a
  **coagulating pan** (a `Mould`) holds the pool a game day and keeps the
  sheet. ⚠ A cup of latex never sets — the set runs only in a mould.
- **Dandelion rubber** — the russian dandelion (`trade-farming`): seed on
  the general store's counter, dug in its first autumn. The roots are
  **wet-milled**: `mill` (or `grind`) them at any millstones — the
  Delight's grist mill, a hand quern — with **5 L of water** and an
  **empty pail** in reach (the byre has both), and the crushed root gives
  up its latex into the pail (`dandelion-latex`). It is the Hevea's own
  latex from there: poured into a coagulating pan, a game day later it is
  a sheet of raw rubber. ONE set process for both supplies.
  ⚠ It shipped as pulp-then-`make` and the drive found that unreachable
  three ways (`make` runs a player's script, not a catalogue recipe; no
  verb washes a pulp; a `Casting` is not a craft output). The root is
  edible (a grown, marked thing is craft STOCK only as food), and the
  recipe has no ladder placement (rubber has no knowledge gate).
- **Sulfur** — the Hill Quarry's caprock band (−10 … −11 m), won by `dig`
  as a 50 g nodule (a `Firewood`; `stoke` takes it).

## The cure — data on the feedstock

Raw rubber's row carries a `crosslinking` spec; `Crosslinking.outcome()`
evaluates it. `dose = holdS · 10^((T − 418)/12)`, `degree = min(1,
dose/1800) · min(1, sulfur/0.03)`:

| outcome | when | material |
|---|---|---|
| under-cured | degree < 0.6 | raw rubber (tacky, weeping) |
| ebonite | sulfur ≥ 20 % | ebonite (hard, black, rings) |
| burnt | held above 455 K | burnt rubber (blistered, brittle) |
| cured | otherwise | rubber, graded by its miss (the cure reaches *fine*) |

⭐ **There is no knowledge gate**: nothing reads who fires or what they
know. The sulfur fraction is the whole matched charge's (what you weighed —
nodules counted against rubber); the hold is `fire … for <duration>` (the
world clock's grammar), else the recipe's `holdS`, else a minute. The works
oven holds ≈ 430 K, ≈ 482 K with the bellows — past the scorch line, which
is how you scorch. A second polymer is a spec on a second feedstock.

## The mould decides the shape

One `vulcanize` recipe; the shape is the MOULD's. A charge loaded into a
mould standing in the chamber comes out through `mould.castInto(...)` — the
dairy's one cast path (shape, material, mass, wear) — and the cast takes
the weaker of the mould's grade and the cure's. Loose, a cure is its
material's own lump; a cracked mould casts a lump. `holdsCast()` is true
only for a CAST (a thing of the mould's shape, or a lump), so a charged
mould is not "set". ⭐ The seam is moulding-slate § 5's: pressing, failed
pours and consumed moulds extend it without moving it.

## The goods, and the four corrections

Rubber boots, gloves and an apron are **one-piece `Garment` casts**
(`constructionForm: hide`, no bill; mending is the mould's `chemistry`);
packing and hose are a `Tool` and a `Thing`; the rubberized coat `extends:`
the tailor's coat in rubberized cloth. The general store stocks boots,
gloves and the coat. What the goods prove is four corrections to shipped
readers:

1. **Wet matter conducts** — the wet law (`Material.permeability()` /
   `wetProperty()`, owned by storage-and-ice) on each worn layer as soaked;
   wading the flooded cell (down from its catwalk) soaks the feet. Leather
   boots soak through and shock; rubber takes in nothing.
2. **The outer layer sheds rain** — `Attired.exposeToRain`: each distinct
   outermost garment is rained on once; the body takes permeability ×
   saturation of it.
3. **Wear reads the material** — the pump packing wears by
   `(10/toughness)·(1 − min(0.8, elasticity/500·0.5))`: rubber 4× leather.
4. **Rubber shrugs acid** — vitriol and lime name `once-living`, not the
   coarse `organic`; rubber keeps its `organic` tag.

## The works

`/world/terminus/goods-yards/rubber-works` — six rooms, each saying what
it is made of (brick and slate; the stove house glass all round; the roof
terrace `none`/`none`), and a `Structure` that says only what stands on
what (the loft on the works floor, the tower on the store, the terrace on
the loft). A second works is rows.

## Deferred

Synthetic rubber and tyres (destructive-distillation-slate); compression
moulding, a pressed pan and consumed moulds (moulding-slate); the stove
house's stoker brain (standing-instructions-slate); a published cure curve
(inquiry-slate).

## Stretch

The fourth reader of `elasticity`: a garment gives round the body by its
material (the waist's, when one is set). The general store stocks plain
and elastic-waisted trousers side by side, and `wear` says which one is
tight on you. Arithmetic: [textiles.md](./textiles.md) § *Fit*.
