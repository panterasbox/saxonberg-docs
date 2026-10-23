# Sericulture — the RGO whose take ends the animal

> **Pack:** `trade-sericulture` at `/trade/sericulture` · **Discipline:**
> `sericulture` · **Verbs:** none of its own (`put`, `get`, `harvest`,
> `reel`, `spin`, `weave`, `heat`) · **Design:**
> silk-slate.md

A tray of silkworms eats cut mulberry leaf for about a month, ages by the
warmth of the room it stands in, moults four times, wanders, and spins.
The cocoons must then be **stifled** — held hot enough, long enough, to
kill the pupa — within about ten days, or the moths break the thread on
their way out and lay the next lot where they stood. That deadline is the
trade's one decision: what you stifle becomes reelable silk, what you keep
becomes seed, and the heaviest you keep raise the next generation by the
breeder's equation.

Silk is the **fourth row of the taps feedback law** ([taps.md](taps.md)):
milk feeds back on the rate directly, eggs through a state the act
prevents, wool and honey not at all — and silk's take *ends the animal*.

## The tray is the host

`RearingMixin` (pack `lib/`) on `RearingTray` =
`RearingMixin(ThermalMixin(ContainerMixin(Good)))`. The worm is below the
resolution anything acts at; the population is the tray's state, exactly
as a colony is the hive's ([apiculture.md](apiculture.md)). Nothing mints a
"rearing".

- **Hatching is adoption.** `put egg lot in tray` copies the lot's facts
  onto the tray (`seedFromLot`) and consumes the card after the move (the
  hive's microtask lesson). A one-brood lot that has not wintered is
  refused in words; the winter clock (`chillS` below 5 °C) lives on the
  `EggLot`.
- **The clock is degree-days** off the tray's own temperature trajectory
  (base 10 °C): hatch at 150, five ages ending at 52 · 97 · 157 · 232 ·
  352 after hatch, the last tenth of ages I–IV a moult (asleep, eating
  nothing). Reconcile-on-read, no far-past guard.
- **Feeding is a soft limit.** Leaf is contents (`put leaf in tray`; the
  tag `silkworm-fodder`). Each window takes the age-weighted demand from
  the leaf on the tray, weighted by freshness band, and the worms age at
  `0.5 + 0.5 × satisfaction` of the warm rate. ⚠ **Hunger never kills** —
  it slows the clock and lightens the cocoons (the fifth age's mean
  satisfaction sets the shell).
- **The end:** a mounting brush (tag `mounting`) gives wandering worms
  something to climb; without one they spin a day late, on the floor,
  ×0.8. Three game days later the tray holds three `Cocoons` stacks —
  **heavy (top 20 %), middling, light** — and a sack of frass (compost).
- **A tray is named by what is on it** — *a tray of silkworms*, *of cocoons*, *of eggs*, *of wandering/spinning/sleeping/dead silkworms* (`RearingTray.getShortDescription`), so a room of trays can be told apart and `look silkworms` finds the working one.
- **Reads are words.** `rearingLine()` via a markup augmenter: *eggs,
  waiting for warmth* · *in their third age, eating* · *restless, heads
  raised* · *asleep for the moult* · *the sound of rain on leaves* ·
  *wandering* · *spinning*. Never a number.

## The disease nobody can see

`pebrineLoad` is **seeded** on an egg lot (authored on the bought rows:
imported 0.02, the house's valley lot 0.25, the cheap card 0.8) and is a
`spoiler` field — no `look`, `handle` or wiki line reports it. It shows
itself as it did in 1850: a heavy tray eats poorly, is **peppered with
dark specks by its fourth age** (≥ 0.4), and **dies at the start of its
fifth** (≥ 0.6); a light load spins `× (1 − load)` cocoons. A lay
multiplies the mothers' load by 1.6. Telling a clean lot before rearing it
wants a microscope — optics-slate's seam is `EggLot.pebrineLoad`.
It is NOT `ContaminableMixin`: that gauge is a food host's, with every
constant on a pathogen row and *no seeding path* as its invariant.

## The cocoon's deadline

`Cocoons` = `ThermalMixin(StackableMixin(CraftedMixin(Good)))`; every fact
about a stack (strain, heft, means, load, spunAt, stifled/scorched/
pierced) is **stack identity**, so a split keeps them and unlike stacks
never merge. Each read integrates the stack's own trajectory:

- **stifled** after 10 min at ≥ 333 K; **scorched** after 30 min at
  ≥ 423 K (two bands off). ⭐ A cocoon has almost no thermal mass, so the
  window is the difference of the hold times, calibrated on a real lit
  oven: a 500 K kitchen range stifles at ~11 min and scorches at ~32; the
  345 K **stifling oven** never scorches.
- **unstifled**, it opens `emergeDays` (10) after the spin: pierced (one
  band off, still spinnable — the material is named `spun silk`) and ONE
  `EggLot` laid where the stack stands.

## Selection — R = h²S

The lot carries a **genetic** mean (`lotGeneticMg`); the tray's
**phenotype** mean is that × feeding × brush. The heft stacks sit at
`p ± 1.4 sd`. At a lay, the new lot's genetic mean is
`lotGeneticMg + h² × (stack mean − p)`: keeping the heavy stack raises the
next generation, keeping the light lowers it, the middling moves nothing —
and an underfed tray does NOT breed lighter worms (only the heritable part
moves).

## The reel — fishing's verb, the instrument decides

`reel` is ONE view, fishing's (the collision ladder's first rung: unify
behind an interface). ⚠⚠ **Known debt, shipped knowingly:** at review
this proved a FALSE unification — the controller narrows on the target to
pick the mechanic, which is command-spec's own tell for two verbs. The
undo (fishing's view restored; silk's own `unwind` in this pack) is
`command-taxonomy-slate.md` § The work, item 7, deferred to the global
command sweep. With nothing bound it is the fishing act; bound to a
target answering `Reelable` (`trade-fishing/src/lib/Reelable.ts`, the
kernel's `WorkPrognosis`/`WorkResult`), the target plans, the controller
engages, the target completes and names the Discipline to credit.
`ReelingTool` (hand reel rate 1; the mill's basin rate 2, `control: fine`)
affords it; the stack affords nothing. `reel cocoons --count N`: 40
cocoons per reeling, wants a lit fire in reach ≥ 330 K, makes ONE skein of
`RawSilk` (a `TextileStock`, `yarnCount` 0, so `spin` throws it) with the
filament count and an **evenness** from the reeler's competence and the
reel's control, the reeler's mark, and a handful of pupae (food).

## Downstream

Raw silk (`gummed`, `sericin`) carries its material through `spin` and
`weave` ([textiles.md](textiles.md) — the chain keeps a fibre's material).
A bolt boils off its gum in a **copper** (the textiles pack's
`FinishingMixin` + the degumming profile here, through the kernel's
immersion rung — [thermal.md](thermal.md)) and comes out **silk**, white
and a quarter lighter. Gummed cloth takes a dye thinly.

## The silk house and the mill

Hearts Delight's silk house (`/world/terminus/hearts-delight/silk-house/`)
is a loft of three persistable `HouseTray`s (fixed in its racks, and named *house tray* so a bought *rearing tray* set down beside them is still the one `put … in rearing tray` finds), a stifling oven, a counter
and a mulberry walk — an orchard: an ordinary place holding one kernel
`PermanentBed` (a bed that is a fixture of its place and remembers being
cut) of four mulberries and a kermes oak; not a forestry `Panel`, which is
a timber stand's bed — run by a keeper (the judgment seat) and a paid hand (`tends`: feed,
mount, harvest, stifle, set out). ⚠ **It runs while LOADED** — by a
visitor, or the Wharfside reeler's buying trip — never booted (the boot
ceiling is full; an unobserved house is the unobserved-economy slate's).
Its first tray is authored mid-spin, so a crop is there in any season.

The Wharfside mill's **reeler** (wage 3 — the bottleneck seat paid least)
runs `stocks` (walks to the house and buys cocoons for the crib), `reels`
(reel → throw → weave → the silk shelf) and `consigns` (silk bolts to the
general store at the labour bench's ask).

## The labour bench — a luxury is hours

`trade-sericulture/src/__tests__/labour.bench.test.ts` reads the shipped
dials and rows and prints attended hours per web and per kg of fibre —
linen ~2.6 h/kg, wool ~2.2, **silk ~170** — with a rearing priced as
ATTENDANCE (two 15-minute visits a day for the four feeding weeks), and
asserts the silk prices are those hours at the seats' wages (the cocoon
floored at one coin; the floor is markets-slate open Q5's).

## Deferred

⚠ **The drive's one recorded gap** (silk drives 26–27, 14/17): the
mill-to-store chain — the reeler's bolt reaching the general store, then
bought and degummed. The house sells only the crop its hand has set out
(~170 cocoons in the run), the drive's player buys 120 of it first, and
the reeler ends with ~50 of the 160 a bolt needs. The economy is right
(one buyer took the crop); the drive's ORDER sets the player against the
mill. Fix: run the mill section before the player's buy. The reeler
already buys a bolt's worth (160) a trip.

The house's brazier and oven burn bounded fuel props — the fuel trade's
supplied line is the attach point (fuel/markets slate) ·
Bait from pupae (fishing's closed `BAIT_KINDS`) · `FinishingMixin` on a
garment, wool scouring, piece-bleaching (textiles-slate) · a booted house
(unobserved-economy-slate) · microscope certification of a lot
(optics-slate) · a fibre's own base colour in the dye mix (textiles-slate)
· the conditioning house, status demand, the finite labour pool
(markets-slate).
