# Apiculture

⭐⭐ **The first RGO whose reservoir is somebody else's land.**

A colony is the organism. Not a herd of insects you draft one from and
not a box with a number on it: a single living thing with a strength, a
queen, stores of honey and pollen, and a temper you find out by putting
your hands on it. The individual bee is below the resolution anything in
this game acts at — you never name one, never handle one, never treat one
— so **the population is the body and the arithmetic is the metabolism.**

What it eats is the bloom within about an hour's flight of where it
stands, which means two keepers on one valley find each other out with no
ledger telling them. What it leaves behind is a fruit set on trees it does
not own. Both of those are the subsystem's reason to exist.

- **Pack:** `trade-apiculture`, root `/trade/apiculture`.
- **Verbs it ships:** one — `rob`.
- **Verbs it borrows:** `open`, `put`, `split`, `handle`, `ignite`, `make`,
  `consign`. All the platform's or ranching's.
- **Discipline:** `apiculture` (ISCED-F 0811).

## ⭐ What the taps build changed here (2026-10-01)

Nothing a player can see, which was the point — `rob` is the exemplar
the tap substrate was generalized FROM, and AC 15 was *the hive behaves
exactly as it shipped.* What moved is where the code lives:

- `ProducingMixin` is the **kernel's** now (`lib/husbandry/Producing`),
  so `Hive`'s import is a specifier. ⚠ The pack still depends on
  `trade-ranching` — for `HandledMixin`, which carries the `handle` verb
  — so the dependency did not go away, only the mixin import.
- The honey tap declares `window: {kind: biome}`, which states in DATA
  the thing this doc already claimed: ⭐ the first RGO whose reservoir is
  somebody else's land. `Hive.biomeWindowOpen()` reads its own forage
  census, so *"the refusal is the season's rather than the colony's"* is
  now a window answer rather than an empty phrase.
- ⭐⭐ **`RobController.mint` moved onto `Hive.mintTake`.** The
  controller used to stash its target on `this` in an `execute` override
  just to reach the hive's forage — *a controller holding state about
  its subject is the tell that the behaviour belongs on the subject* —
  and the comb composition, the per-frame minting and the forage blend
  all live on the hive now. `RobController` is twelve lines.
- `rob` gains a short **duration** (it is an engagement like every other
  take). ⚠ That is the one player-visible change, and it was the plan's
  Risks §1: AC 4 (*every take takes observable time*) against AC 15
  (*the hive is untouched*). Resolved toward AC 4; it is one number on
  `mellifera.yaml` if it reads wrong.
- `Hive.takeFrom` keeps its one-box-worth override and returns the new
  `TapTake {units, worst}` shape. `worst: 1` means *no quality record*,
  not *best* — `worst` is a `continuous` tap's field and a hive has none.

The mechanism is [taps.md](./taps.md).


## The two hosts, and why there are exactly two

`ColonyMixin` (`src/lib/Colony.ts`) carries the population. It is composed
by:

| host | what it is | what it has | what it has NOT |
|---|---|---|---|
| **`Hive`** | a colony in a box | an interior volume, a wall, capped comb, a lid, taps | — |
| **`Colony`** | bees with no box — a swarm, a nucleus, a split | strength, a queen, a temper | **no stores**, because stores are comb and comb is in a box |

That second class is what makes AC 1 work. Buying a nucleus, catching a
swarm and splitting a strong hive are three different acquisitions with
three different costs, and **all three mint the same Thing**; installing
it is the platform's `put`. Without it each acquisition needed either a
bespoke verb or a state transfer between two Vessels, and `put nuc in
hive` is a better sentence than any verb anybody would have invented.

⭐ The absence of stores on a boxless colony is expressed as three
protected hooks — `storesKg`, `storesCapacityKg`, `consumeStores` —
answering zero, rather than as a guard. A guard that re-narrows the host
set is the tell that the host is wrong.

## ⭐⭐ The hive is a `Vessel`, and `AtmosphericMixin` is composed on it

What a winter costs a colony is decided by the wall it is living behind,
so the hive needs a volume, an exposed area and a conductivity — which is
precisely the envelope arithmetic the buildings use.

⚠ `AtmosphericMixin` used to live on `Vessel` and the base-class narrowing
moved it to `ExitableVessel`, on the argument that *inside is something
you can only BE for a vessel you can go into* — thirty-seven rows had
claimed their own weather and not one authored an atmospheric field. **A
hive is the honest counter-case:** you cannot go inside it, and the
interior climate is the entire mechanism. So the hive composes the mixin
itself. It overrides `getVolume()` (the brood box plus every `HiveBox` in
it) and `enclosureDefaults()` (its own material at a box's wall
thickness), and inherits `openExteriorOpenings()` as zero because a Vessel
is not Exitable.

⚠ `envelopeApplies()` is **not** overridden and is false for a hive under
the sky. That is deliberate: the build reads
`envelopeCoefficients().uWperK` directly, which needs only a volume and an
area. The hive's integrated interior temperature is not a product
requirement — a `feel hive` reading is a thermal-slate follow-on.

## The winter equation

```
U        = hive.envelopeCoefficients().uWperK                 (W/K)
clusterK = brood season ? 308 : 293   (brood season ⇔ outsideK ≥ 283)
P        = U · max(0, clusterK − outsideK) · coupling          (W)
burnKg   = (maintenanceKgPerDay · strength + P·86400/honeyJPerKg) · days
```

⭐ **Nothing here is authored per hive.** A thick-walled box needs less
honey than a thin one because a thick wall has a lower `U`, which is the
same arithmetic a shed uses — so the epoch ladder (skep → thin box →
thick box) is **two numbers on a row and no code at all**, and a player
who works out that a warmer box costs less honey has worked out something
physically true. `colony.test.ts` asserts the ORDERING, never a figure.

`coupling` (0.35) is below 1 because a cluster is not the whole interior:
the bees heat themselves, not the woodwork, and the entrance's leak is
folded in here rather than modelled as an opening.

⚠ **No far-past guard**, exactly as the taps have none. A kept colony's
clock runs while its keeper is away, and what an absence costs is a
season, a swarm, or — if it was already starving in the cold — the
colony. A guard would refund the thing the build exists to make real.

## ⭐⭐ Out of stores has two endings, and the weather picks

| weather | after | what happens | `lifecycleState` |
|---|---|---|---|
| flying (≥ 283 K) | `abscondDays` (3) | **they leave.** The box is empty. | untouched — nothing died |
| cold (< 283 K) | `starveDays` (5) | **they starve.** | `'dead'` |

That is one branch answering AC 8 and AC 14, and the difference between
them is legible in what the keeper did: taking all the honey in autumn is
a decision with a spring in it.

## Swarming

A colony swarms when it **filled its room in a flow** — strong, queened,
stores near the top of the comb it has, and forage on. Every term is
something the keeper can act on, and giving them another box is what
stops it, which makes supering a decision rather than a chore.

Half go with the old queen; the parent raises one after `requeenDays`
(21) if enough of them are left to get her mated, and dwindles if not. The
departed swarm is given a body — a `Colony` Thing in the hive's own
location — and **hangs for a day**: a keeper who is present catches their
own swarm and one who is not loses half a colony to the woods. Either way
the reading says it swarmed.

⚠ It is **not brain-shaped**. A brain is conduct emitted on channels by a
Behaved host with cadence timers; this is arithmetic on a record, which is
the growth model's shape and reconciles correctly across an absence.

## The range, and what it gives back

`Hive.forageCensus()` is a bounded breadth-first walk **over exits**, not
containers: bees fly out of a hive and across the ground, so the walk is
the one a traveller makes, priced in minutes, stopping at about an hour's
flight with a hard hop cap. Every read on it is sync, because it runs
inside a reconcile.

It counts, in clover-equivalent m²:

- a **sward**'s own `inFlowerFraction()` × its area. ⭐ The sward knows
  when it blooms; the hive asks. A pack-local helper reaching into the
  Field's cached sky would break the methods-only inter-Stuff contract.
- every **flowering plant**, including the plants seated in a bed one
  level down;
- and **every colony on the range, itself included.**

`forageFactor = totalM2 / (forageM2PerHive × hives)` — so two hives on one
valley's bloom each get half a living, and **nothing tells either keeper
that.** The second beekeeper finds out by looking at their own bees
(AC 13), and a gauge reading *"forage 47 %"* would replace the lesson with
a number.

⭐⭐ **The same walk pushes pollination back.** Every flowering plant in
range gets `plant.pollinate(share)`, and the plant does not know bees
exist — it exposes the method and the hive is the thing that pushes. So a
grower whose trees set a full crop benefits from a keeper who put a box
over the wall, and neither of them had to agree to anything. **That
asymmetry is the trade.**

The kernel side is `GrowingMixin`: `GrowthProfileData.pollinationBaseline`
(the share a plant reaches with no pollinator, `1` when absent and
therefore byte-identical for every row that shipped before it),
`_fruitSetCount` / `_pollination` latched at the set moment, and
`getFruitSetCount()` read by `HarvestController` at the pick. Authors
author the CAUSE — what pollinates this plant — never the effect.

## The take

⭐ **Honey needs no tap-window rule**, because the stores ARE the tap
state. There is no second honey number: what is standing in the `honey`
tap is what the colony has to eat, which is why robbing in autumn is the
same act as taking the winter away.

`rob` is a `TapController` subclass — taking honey really is the same act
as milking a cow — with one override that is the whole product decision:
**a rob takes one box-worth, not everything.** You rob again to take more
and you stop to leave some, and **nothing warns you either way.**

What comes out is **comb**, by the frame, not honey: you take the comb
away and get the honey out of it afterwards, two ways.

| recipe | tool | honey | residue |
|---|---|---|---|
| `crush-comb` | none | 0.55 L | a **cake of beeswax** |
| `spin-comb` | `extracting` | 0.72 L | the **comb back** (sets `combDrawn = 1`) |

Neither is better. Which you want depends on what you are selling, and
nothing in the code branches on the tool: two recipes over one input.
⚠ The slot's `category` is a Material **tag**, so both name `honey` (what
the comb is made of) and `kind: item` vs `kind: bulk` is what keeps a jar
of honey out of a comb slot.

⭐ **Honey's character rides the comb's `composition`.** `Comb` is a
`Provision`, so it composes `ComposedMixin`; `RobController` stamps the
forage census onto every frame it mints, and the crafting core already
sums an item input's composition into a bulk output's payload. `taste`,
the label and the tags derive from it on read — so clover honey and
cherry-blossom honey are different honey with **no row written for
either**.

## The sting

`ColonyMixin.disturb(actor)` fires from `Hive.open()`, from `workedOver`
and from `rob`. Not a die and not combat:

```
attempts = round(stingBase × (0.25 + handlingRisk()) × (smoked ? 0.15 : 1) × (cold ? 1.5 : 1))
```

Each attempt is a `point`-channel `HazardDelivery` at a site walked from a
**seeded** gap ladder — the host's identity plus its disturbance count —
so the same hive worked the same way twice gets you in the same places,
which is how you learn to cover them. Each goes through the same
`ConditionApi.inflict` path as a poisoned dart, and **the dose is gated on
`afflicted`**: a covering the channel cannot breach turns the wound and
the venom with it.

⭐ So a veil works because it is cloth over the place bees go for, not
because it is a bee veil, and **nobody authored a mitigation table**. One
sting is half a unit of venom, well under the shipped condition's first
band (3); thirty crosses its third (15) and is a medical problem. The
bands do that arithmetic themselves.

⚠⚠ `stingJ` is **0.26** and the window is narrow (0.25–0.28), because the
shipped covering grid says NO cloth resists a `point` and content may not
author a resist profile — that is combat mitigation and deliberately
kernel-only. What a layer of cloth does to a sting is take a hair's worth
of energy off something that was only just enough to break skin, **which
is what a sting is.** `stings.test.ts` pins both ends of the window; if a
veil stops working, that dial moved.

⭐ A bee dies when it stings, so every landed sting costs the colony a
little strength. Working a hive roughly is not free.

## AC 2 — the reading is bands, all the way down

`handle` reaches the hive through **ranching's** view
(`lint:verb-collisions` refuses a second one, correctly), so the body of
the act moved onto the animal: `HandledMixin.workedOver(actor)` returns a
report and its DEFAULT is the livestock body verbatim — the rail-slam
hazard, `handle(1)`, the flesh score out of a hundred, the spine prose.

A hive overrides it. What you get is what a beekeeper actually gets: the
traffic at the door, the heft of the box when you tip it, whether there is
brood in a tight pattern, the temper, the last event, and the crowding
sentence. **No number appears anywhere in it** — the drive asserts that
with a regex — because the precise flesh score belongs to palpating a
mammal and is livestock's sentence, not the mixin's.

The alternative was a `typeof target.colonyReading === 'function'` branch
in the controller, which is the guard that tells you the host is wrong.

## AC 15 — one mechanism, twice

| row | input | lag | product | turns to |
|---|---|---|---|---|
| `honey-wild` (apiculture) | `honey` | 6 days | fermented honey | nothing |
| `mead` (winemaking) | `honey-must` | 4 days | mead | wine vinegar |

Both are **spontaneous** (`spontaneousLagDays > 0`, the shipped
`requiresFlora` predicate's third clause): a vessel left **open** catches
wild yeast out of the air, a pitched one starts at once, and a sealed one
never starts. Honey left open ferments by itself and that is a defect;
honey diluted three-to-one and fermented is mead and that is a product;
**what tells them apart is that somebody meant one of them.**

⚠⚠ Honest limit: the microbial clock reads temperature and openness, not
ambient humidity. The criterion wanted *"somewhere it can take up damp"*,
and honey left open **is** how honey takes up damp — so *open* is the
honest form of the same fact rather than a weaker one. A humidity term on
`spontaneousLagDays` is a maturation follow-on.

⚠ Tag hygiene against the double-match rule: `honey` carries `honey`;
`honey-must` carries `honey-must` and **not** `honey`.

## Where the content lives

| what | where | why |
|---|---|---|
| the honeybee species | `/stuff/idea/species/…/apis/mellifera`, file in **trade-apiculture** | a honeybee is a fact about the world, so the row is the commons'; the file moved so the pack's content is not pointing ranching at apiculture |
| beeswax | **base-library** | the candle is a generic object and a generic object must not depend on a trade to know what it is made of |
| the candle, the veil, the gloves | **generic-objects** | ordinary clothing and an ordinary light |
| honey must, mead | **trade-winemaking** | the process that makes it |
| the kit and the nucleus | **Terminus's general store** | ⭐ a general store's job IS importing, so a faucet belongs there |
| the close | **hearts-delight** | Quist's clover ley under his cherries, titled to his business |

## ⭐⭐ The valley keeps a vacancy, deliberately

Nothing in Heart's Delight keeps bees, sells bees, or sells a hive. That
is the build's own participation finding, and it is two findings:

1. A par faucet on Quist's farm would have made one NPC the farmer, the
   landowner, the pollination beneficiary, the honey buyer **and** the
   woodenware seller — five roles, in the build whose thesis is that a
   beekeeper is a **second party** on land they do not own. You cannot be
   a second party to a farm that keeps its own hives.
2. Hives, supers and frames are **carpentry**, so the trade creates a
   woodenware-maker-shaped hole. The store's par is standing in for a
   trade that does not exist yet; the drive's dirty reason says so.

## Not built

- **Varroa**, as the setting rather than as a deferral: this is a world
  where the mite has not arrived.
- **Foulbrood** — the seam is `ContaminableMixin`'s spore floor, which
  `Comb` already composes because it is a `Provision`.
- **Robbing between colonies**, persistent-disturbance absconding.
- **Pollination as a contract** (N hives present over the bloom window).
  The census already counts hives per location.
- **A `face` body part.** The gap ladder is authored so `body.head.face`
  slots in first the day the body plan grows one; until then the head is
  the honest approximation and the veil covers the head slot.
- **Allergy.** The sting's `introduceToxin('venom', …)` is the exposure
  event a pharma build will read. It is out of scope here because no
  remedy exists, and an anaphylaxis with no counterplay is a death
  sentence rather than a mechanic.
- **The hive interior as an applying envelope** — a `feel hive` reading.

## Cross-references

- [ranching.md](./ranching.md) — the taps, `HandledMixin`, `workedOver`
- [husbandry.md](./husbandry.md) — the growth model and the set latch
- [hazard.md](./hazard.md) — venom rides the wound
- [maturation.md](./maturation.md) — the spontaneous ferment
- [spoilage.md](./spoilage.md) — water activity and the microbial floor
- [materials-response.md](./materials-response.md) — why cloth resists a
  point poorly, and why that is right
- [smallholding.md](./smallholding.md), [soil.md](./soil.md) — the Field
  and its sward
