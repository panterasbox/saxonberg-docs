# Spoilage

Food goes off, and the reason it goes off is a **population**, not a
timer.

⭐⭐ **There are TWO populations, and the difference between them is the
whole subject.**

> **Spoilage is a CLOCK. Contamination is an EVENT.**

- The **spoilage flora** grows on its own, everywhere, forever. It is
  what makes a ration go off in a pack, and it *tells you*: the band, the
  smell, the ptomaine. By the time it can hurt you it has been warning
  you for a while. This is `FreshnessMixin`.
- The **pathogens** never appear on their own. Something has to have
  *happened* — an animal opened, a dirty knife, a board — and once one is
  there, **no sense reports it at all**: no reading, no smell, no taste,
  no tell. This is `ContaminableMixin`.

And a third fact, which is about the matter rather than about anything
living in it: how much water is actually available to grow in. That is
`CuredMixin`, and it is what drying and salting change.

Every `Provision` carries all three, and they are three different
questions about one cut of meat: *what is growing in it on its own*,
*what somebody put in it*, and *how much water either of them has to work
with*.

⚠⚠ **The doc used to say the gauge "teaches before it punishes"** — that
the dose onset sits inside the tainted band on purpose, so you get a
smell warning before you get a dose. **That is still exactly true of the
spoilage flora, and it is false of the system.** The second population
punishes without teaching, by design, and what keeps *that* fair is a
different thing entirely: the *risk* is legible even though the *hazard*
is not. You can see that the meat is raw, that the board was used for
gutting, that the stew has been out since morning. **Invisible to the
senses, knowable by procedure.**

Source: `lib/material/Freshness.ts` (the spoilage mixin + the shared
arithmetic), `lib/material/Contaminable.ts` (the silent population),
`lib/material/Cured.ts` (the water state), `lib/material/Material.ts`
(the two tabulated constants), the `freshness.*` / `cure.*` dials in
`lib/config/AppSettings.ts` and their seeds in the platform pack's
`content/settings/`. The pathogen roster is authored as `Condition` rows
under `/platform/idea/Condition/pathogen/`. Read alongside
[thermal.md](./thermal.md) (the gauge asks its host what temperature it
is), [metabolism.md](./metabolism.md) (where the dose lands and where an
infection starts), [vitals.md](./vitals.md) (where an infection grows)
and [bulk.md](./bulk.md) (the blend half of every gauge here).

## The rate law

```
μ = μ_max · f_T(T) · f_aw(a_w)          growth, logistic toward capacity
```

Two terms, and each is the honest one:

- **`f_T`** is **Arrhenius** over the food's own tabulated activation
  energy (`Material.spoilActivationEnergy`, J/mol; real foods sit around
  50–130 kJ/mol). Warm food spoils faster, and *how much* faster is a
  property of that food rather than a global multiplier.
- **`f_aw`** is the **water-activity** term. `Material.waterActivity` is
  how much of the material's water a microbe can actually use — fresh
  meat ≈ 0.99, a hard ration ≈ 0.64, honey ≈ 0.60, dried salt ≈ 0.15.
  Below the floor (`freshness.awFloor`, 0.60) nothing grows at all.

⭐ **That floor is the whole preservation curriculum in one number.**
Salt, sugar, drying and candying are not four mechanisms; they are one —
take the water away and the population cannot start. Nothing in the
library carries a `shelfStable: true`, and the reason a sack of salt
keeps forever is legible from its own row.

Three regimes fall out of the same law, with no branches beyond the two
temperature bounds:

| regime | what happens | why |
|---|---|---|
| below `freshness.freezingK` (273 K) | growth **pauses** | the water is ice; a thawed thing resumes where it left off — freezing is not a reset |
| between | logistic growth at `μ` | the ordinary case |
| at/above `freshness.killK` (333 K) | the population **dies** exponentially | 60 °C is the pasteurization floor: this is what cooking does |

Growth is closed-form logistic from an inoculum
(`L(t) = L₀e^{μt} / (1 − L₀ + L₀e^{μt})`) and death closed-form
exponential, so a week-long gap costs exactly what a minute does.

## The gauge

`FreshnessMixin` composes onto exactly **one** class: `Provision` — "the
one class in the library of which [being food] is true by name". Bulk
holders (`Bottle`, `Receptacle`, `UnboundedReceptacle`) do **not** compose
it; their gauge is the payload field, read through
`Freshness.loadOf(slot)`.

⚠ **It took two rounds to land there, and both wrong answers had the same
shape.** First `ThingBase` — all 152 `Thing` classes — which put five
spoilage methods on the documented author surface of a rock, a lantern and
a pair of socks. Then `Prop`, the deliberately-empty generic `Thing`,
which taxed the anvil and the toilet instead. Both were argued from *"a
prop is often food"*, which is a claim about ROWS; the four rows in
question (`stew-meat`, `root-vegetables`, `ration-stock`, `rations`) were
simply on the wrong class, and `prime-cut` — in the same pantry chest —
was already a `Provision`. **The fix for a mixin on too wide a host is
usually to move the rows, not to widen the class further.**

⭐⭐ **What makes the narrowing safe is a GATE, not a judgement.**
Perishability is a property of the **Material**, not the class — the same
class is an anvil or a cut of stew meat depending on its `_materialPath`,
and only 30 of the library's 107 materials tabulate an activation energy.
Read too far, that argues for composing as wide as possible (it is what
produced both wrong answers above). So narrowing by hand risks the worst
failure this codebase knows: food authored onto an inert class would
**silently never rot**, exactly as `eat` shipped with no affordance and
`feel`/`taste` never ran.

`pnpm lint:perishable` closes that: every shipped row whose
`_materialPath` names a material that rots must resolve to a class
composing `FreshnessMixin`. CI-gating, no exemption list — a row that
legitimately holds perishable matter on an inert class is a design
conversation, not a list edit.

⚠ **What the gate cannot see, stated rather than implied:** a RUNTIME
`setMaterial(perishable)` onto a non-`Fresh` host. A gate reads authored
rows; it cannot read a craft's output assignment. The gap is narrow — the
craft paths flow material onto outputs of known classes — and it is the
price of the narrowing.

**Inert by default**, therefore, is what carries the design: a Material
tabulating no activation energy never advances past zero. An anvil does
not rot, and no flag says so.

⚠⚠ **The sparse-storage guarantee is an ORDERING, and it is easy to lose.**
`reconcileFreshness` checks perishability **before** it reads the clock,
so inert matter reads and writes nothing at all. Get that order wrong —
stamp first, ask later — and the first `look` at an anvil writes a
non-default `freshnessClockStamp` into its snapshot forever, for matter
that can never rot. It shipped that way for one review round: "two scalar
fields at their `0` defaults" was true only until somebody looked at the
thing. Two tests now pin it.

Reconcile-on-read over game-time, with **two deliberate divergences from
`WetMixin`**:

1. ⚠ **No far-past guard.** Wetness drops a long gap because real absence
   never dries you. Food rots over the *whole* absence — coming back to a
   week-old stew is the point, and it is the difference between a
   spoilage gauge and a wetness one.
2. ⚠ **No linkdead freeze.** An item has no `Interactive`, and a carried
   ration does not stop rotting because its owner dropped link.

The temperature the gauge reads is the host's own (`Thermal`), which is
why `Provision` — the class every food row is over — composes
`ThermalMixin` alongside `FreshnessMixin`; the two travel together. A cold larder and a warm windowsill are then different
answers to the same question, for free.

## The blend half

Bulk matter carries the same gauge on `BulkPayload.freshness`
(`{ load, stamp }`), reconciled through its **holder** — the vessel is the
Thermal host. Both halves call the same `Freshness` statics, so a bowl of
stew and the roast it was made from can never age by different rules.

The gauge is **seeded lazily**: a slot holding perishable matter gets one
the first time anybody asks, and nothing else does. The shadow payload
written at that moment mirrors the Material field for field
(`Freshness.materialShadow`), so every `payload ?? material` reader is
unaffected by its arrival.

⭐ **Transfers blend loads by mass on every pour** — unlike the payload's
identity, which rides into an empty vessel only. That asymmetry is
load-bearing: it is what closes the pour-to-reset exploit. Decanting a
spoiled pot into a clean bowl moves the spoilage with the matter, and
tipping half of it into a fresh pot raises the fresh one rather than
laundering the spoiled one.

## The ingest reach

`Freshness.doseFor(load)` maps a load to a `ptomaine` dose — a **curve,
not a step**: nothing at all below `freshness.dose.onsetLoad` (0.3), then
rising superlinearly, so a barely-turned ration is a bad afternoon and a
rotten one is a real poisoning.

⭐ **The dose is folded at the READ and never stored.** The food does not
*contain* ptomaine the way a nightshade contains its alkaloid; it
contains a population, and the dose is what that population has produced
by the moment you swallow it. Storing it would let a refrigerated pot
keep a dose it no longer deserves.

Two folds, one function (`Freshness.withDose`):

- **bulk** — `Freshness.ingestPayloadOf(slot)`, read by `drink`, `sip`
  and `eat`'s dish arm.
- **discrete** — `EatController` builds a transient payload from the
  target's own mixin gauge.

Downstream is the shipped metabolism path, untouched: the dose lands in
the digestion pool, absorbs into the `ptomaine` burden, bands into the
`food-poisoning` Condition, and `vomit` inside the window dumps what has
not absorbed yet.

⚠ When there is no payload, the fold **synthesizes the whole material
shadow**, because metabolism reads `payload?.nutrients ?? material…`: a
payload carrying only the ptomaine would silently drop the food's real
nutrition.

## What cooking does to it

⭐⭐ **"Hot enough" is a RATE, not a line you cross.** The death rate is
Arrhenius over `freshness.killK`: at the kill temperature it is the
tabulated base rate and it climbs steeply from there. So a long hold at a
lower heat and a brief moment at a higher one achieve the same kill —
which makes a **sear**, a **simmer** and a **lazy warm-through** three
genuinely different acts rather than one boolean.

A recipe expresses that as the pair `(requiresHeatK, holdS)`.

⭐ **`holdS` absent means "the working was as long as it needed"** —
byte-identical to the threshold this replaced, which is what keeps every
recipe authored before holds existed cooking exactly as it did. A number
is a claim that the hold was *not* sufficient, and is therefore always a
deliberate authoring act. `seared-cut` (500 K / 10 s) and
`warmed-through` (335 K / 120 s) are the shipped pair; the second is
authored as a trap, and it is the row that makes the lesson a thing a
player can be wrong about.

Then, two different facts, and keeping them apart is the design:

- **The load** falls at that rate over the hold — to nothing when the
  recipe authors no hold; below `killK`, the inputs' loads blend through
  by mass. A lazy warm-through launders nothing.
- ⭐⭐ **…and the dose that load had already earned is deposited into the
  dish as a real, FORMED toxin** — authoring no `labileAtK`, so nothing
  later destroys it either. Heat kills the population; it does not
  destroy what the population made. Cook rotten meat and you get a
  sterile dish that will keep, and poison yourself with it.
- **The rate afterward** comes from the OUTPUT material's own constants.
  `/platform/idea/material/cooked` tabulates the fastest rate in the
  library — warm, wet, nutrient-rich, with nothing left to compete — so a
  cooked dish goes off *faster* than the raw stock it was made from.
  ⚠ That is what leftovers do, and it is the hazard the whole gauge
  exists to make real.

⚠⚠ **Cooking spoiled food does not un-poison it**, and that is true of
both halves: an *authored* dose survives unless it declares
`labileAtK` (see [metabolism.md](./metabolism.md) — a raw bean's lectin
does, ptomaine does not), and a *derived* one is deposited as formed
toxin by the kill itself.

⚠ **The second half was missing until a live drive found it.** The reset
took the derived dose with it, so cooking rotten meat produced a clean
dinner — a free lunch the design explicitly denies, invisible to a suite
that only ever cooked sound stock. Standing the kitchen up is what caught
it.

⭐ **The working heat is not the room's heat.** The craft's heat *gate*
asks whether the setup can supply the recipe's demand; what the food was
held at is the recipe's own demand. A stew simmered beside a roaring
forge was simmered, not forged, and conflating the two would have every
dish in a kitchen cooked at the hottest thing in the room.

## The water state — what drying and curing actually change

`CuredMixin` (`lib/material/Cured.ts`) carries two scalars that describe
the **matter**, not anything living in it:

- **`moisture`** `[0, 1]` — how much of the material's own water is still
  in it. `1` is as-harvested. Drying lowers it.
- **`solute`** `[0, 1]` — how much of the remaining water is bound up by
  dissolved salt or sugar. `0` is untreated. Curing raises it.

Water activity then derives, **multiplicatively**:

```
a_w = a_w(material) · moisture · (1 − solute)
```

⭐ **The multiplication is the whole design.** It is real hurdle
technology: drying and salting are the same lever seen twice — take the
water away, or bind what is left — so they **stack** rather than compete,
and partial treatment earns partial benefit, with nobody enumerating
"salt cod" anywhere.

⭐ `moisture: 1, solute: 0` is the **identity**, which is why every row
that shipped before the axis existed keeps exactly as well as its
Material always said it would. It is pinned by a test, not assumed.

**The asymmetry, and it is the lesson.** Curing does not reverse: salt
that went in stays in, and `solute` has no passive arm at all. Drying
does — a dried thing left somewhere damp climbs back toward the ambient
equilibrium, read synchronously through `BiomeApi.localHumidityFor`. So a
dry store is worth building and a steamy kitchen is the worst place to
hang a ham.

⚠ **The passive arm only ever RAISES moisture.** Nothing dries on its
own: drying is an *act*, and a gauge that quietly dried everything in the
pantry would both undo that and change how every shipped row behaves. An
untreated instance therefore reads and writes **nothing** — the reconcile
returns before it touches the clock.

⚠ Composed **beside** `FreshnessMixin`, never folded into it. Leather,
timber and grain are all dried and none of them rot on a microbial curve;
folding water activity into the spoilage gauge would make a tannery
compose a microbial load in order to express drying.

The acts are `trade-cooking`'s: `cure` (salt, consumed from a sack),
`dry` (time only), `smoke` (a fire, deliberately **13 K under the kill**
so that smoking preserves without sterilising). Each is a recipe row with
a `cure: { moisture?, solute? }` block, applied as the **stronger** of
each axis — so a weaker second treatment never un-cures, and two separate
acts stack.

## The silent population

`ContaminableMixin` (`lib/material/Contaminable.ts`) carries
`Record<pathogenKey, load>` and it is **`{}` until something put a load
there**. There is no seeding path in the file and there is not meant to
be one: the reconcile returns immediately on an empty map, so *no food a
player owns ever becomes dangerous on its own*, at any temperature, over
any span. That invariant is arithmetic rather than policy.

**Hosts — food equipment, and nothing else.** Two kernel classes and three
of the cooking trade's own:

| host | why it can hold a load | what puts one there |
|---|---|---|
| `Provision` | it IS the food | the gut spill; a craft's tangible output |
| `CraftVessel` | a pot, a bowl, a platter — what food sits in | a fill from a dirty vessel |
| `ButcherBlock` | the board you open a carcass on | the gut spill |
| `BoningKnife` | a cook's blade — food kit, not a weapon | the gut spill |
| `KitchenTool` | the sieve, the press — tools that work food | a craft dirties the tools it used |

⚠ Not `Cutlery`, which touches a mouth rather than a carcass.

⚠⚠ **And deliberately NOT the two kernel classes those last two descend
from.** `ContaminableMixin` was composed on `Weapon` and on `ToolItem` for
one build, on the argument that *"this can carry pathogens between things"*
is true of a clasp knife and a kitchen sieve. It is — and it was the wrong
question, because it is false of most of both host sets: a mace, a flail, a
warhammer, a whip and a fire poker; a felling axe, a sledge, a pick, a
pick-haft, a pinch bar, a smith's hammer, an assay kit and a shovel.
`callable == visible == cared-about` settles it.

⭐ **The answer was not to delete but to NARROW.** `BoningKnife` and
`KitchenTool` are the food-work subclasses, and the split they express is
the real one: **cutting and carrying are different facts.** `butcher` gates
on an EDGE, so the store's clasp knife opens a carcass exactly as the
kitchen's boning knife does — it just does not *remember* what it cut,
because a pocket knife is a general tool and a cook's knife is food kit.
Likewise a shovel is offered the same contamination a sieve is, at the same
seam, and is structurally unable to take it. **The narrowing does the work,
never a guard.**

⚠⚠ **And not `Weapon`.** It was composed there for one build so the clasp
knife a player buys could carry pathogens off a carcass — which put
`getPathogenLoad()` on the documented author surface of a mace, a flail, a
warhammer and a whip. *Most weapons are never used on food*, so the claim
the mixin makes is false of most of that host set, and
`callable == visible == cared-about` settles it.

⭐ **Cutting and carrying are different facts, and only one of them is a
weapon's.** `butcher` still gates on an **edge**
(`constructionForm: bladed`), so any blade opens a carcass — the store's
clasp knife included. What the blade does not do is remember what it found
there. The carrying lives on the **block**, which is food equipment and is
the canonical cross-contamination vector besides: *do not prep vegetables
on the board you cut raw meat on.*

⚠⚠ **A vessel carries TWO loads and they are different facts.** Its
contents carry their own on the bulk payload; the mixin gives the vessel
a *surface* load as well. A dirty pot and a bad stew are genuinely
different things, and only one of them survives emptying the pot. All
three consequences are live and pinned by
`__tests__/VesselContamination.test.ts`:

1. **Filling a contaminated vessel contaminates its contents** — folded in
   at the transfer, at full strength rather than mass-weighted, because
   what is on the pot is on everything the pot touches however little you
   poured.
2. **Emptying does not clean.** The surface load rides on the mixin, which
   a transfer never touches, so one unwashed pot is a chain of poisonings.
3. **Washing clears the surface, never the contents.** ⚠ `CraftVessel.wash()`
   does this itself. It lived in `WashController` alone for one build,
   which meant `wash()` claimed to make a pot clean while leaving
   salmonella on it — every other caller (a dishwasher, an NPC cleanup
   brain, a `wash all`) got a lie. A method that says "washed" has to mean
   it.

Every constant comes from the organism's own `Condition` row rather than
a dial, because these differ from each other in exactly the ways that
matter:

| what it says | why it is not a dial |
|---|---|
| its own temperature floor, ceiling and Arrhenius steepness | a cold cellar stops one and not another |
| its own **water-activity floor** | *Staph aureus* grows at 0.86, which is why it is the cured-ham organism and why salting alone is not an answer to it |
| a **survival fraction** | the spore-formers: boiling REDUCES them and never removes them |
| a **germination ceiling** | as a cooked dish cools past it, the survivors' rate turns positive again |
| a **reach** — `infect` or `intoxicate` | one grows in you; the other already made a poison, and killing it does not unmake that |

⭐⭐ **`reach` and spore-forming are not alternatives**, and a build that
treats them as a two-way choice cannot author the shipped roster:
*C. botulinum* is a spore-former whose survivors germinate as food cools
and *then* produce a toxin. Both mechanisms, one row.

**The roster** (five rows, and between them every answer is on the table):

| row | reach | what it teaches |
|---|---|---|
| `salmonella` | infect | cooking removes it **entirely** — so cook it and eat it promptly, and curing preserves every cell that was already there |
| `e-coli` | infect | the lowest dose in the roster: a knife that touched a carcass and then a salad is a real route |
| `perfringens` | infect | spores survive the pot and **wake as the dish cools** — the most common real food poisoning there is, and the lesson nobody believes until it happens to them |
| `staph-aureus` | intoxicate | a heat-**stable** toxin: kill the population, keep the poison. Boiling does not save you |
| `botulinum` | intoxicate | spore-former **and** toxin-maker, and its toxin IS heat-labile (358 K). Boiling fixes this one and not the other |

⚠⚠ **`lint:pathogens` is what keeps the roster honest**, and it catches
the silent case: an `intoxicate` row whose `toxin.type` resolves to no
`Condition` deposits a dose that `resolveToxinBehavior` returns null for
and the caller skips — green suite, poisoned food, healthy eater. It also
refuses a non-empty `channels`.

**Where a load comes from:** `butcher`, and nothing else in this build.
Gut spillage is the dominant real contamination route and it is exactly
what an unskilled hand does, so one band read of the `butchery`
Discipline decides both the yield and the mess. ⚠ An expert still
deposits a floor of it: the answer to this hazard has to be cooking and
cold, never a good enough butcher.

**Where it goes:** onto the board and the blade at the butchering, onto
the **tools a working used** (before the inputs are consumed — a destroyed
Stuff is an inert proxy that answers `undefined`), onto whatever a dirty
surface touches next (`transferContaminationTo`), through every pour by
mass, through both craft paths, and across **both arms of `eat`** into the
body.
⚠ That last bridge is the one that fails silently and completely — a
pathogen that does not cross it leaves the suite green, the food
contaminated and the eater fine.

**Where it ends:** `wash` clears a surface outright. ⚠ Which needed the
verb widened: it was `instanceof CraftVessel`, so nothing but glassware
could be washed — not a block, not a pot's surface, not a tool.

## In the body

An ingested `infect` population at or above its `infectiousDose` becomes
an `AfflictionRecord` carrying a live `pathogenLoad`, seeded in
`Metabolic.ingest` (the one place food becomes body) and grown by
`VitalsMixin.reconcileConditions`. See [vitals.md](./vitals.md).

⭐ **Nothing shows until the incubation is up.** Illness arrives hours
after the meal, not at the table — which is the entire pedagogy: the
information is in what you *did*, so a player has to reason backwards
rather than forwards from what they feel.

⭐ **Nothing new kills anyone.** A severe infection drains hydration —
what dysentery actually does — and dehydration already has a lethal
cascade with a rescuable dying window at the end of it.

⭐ **The record names the cook.** A dish reaches a body as
`(material, litres, payload)` and the eater never sees the bowl, so the
maker rides on the **payload** (`lib/craft/Crafted.ts` declares it) and an
accountability row is written when the maker is somebody else. Eating
your own risky food is a private gamble; putting it in front of a paying
customer is a choice about another person, and the ledger is what makes
the two different acts.

## What a player sees

A band phrase on `look` and `smell`, never a number — the
banding-is-presentation rule. A fresh (or inert) thing says nothing at
all. The augmenter is channel-filtered: you *see* that something has
turned and you *smell* it, so a `taste` gets the palate line
([crafting.md](./crafting.md)) rather than a duplicate of the smell.

A **treated** thing gets its own line on the same two channels — *"It has
been thoroughly dried and heavily salted."* Two axes, band words, no
number. ⚠ It is deliberately a **separate line and not a fifth freshness
band**: the population and the water state are different facts, and one
gauge reporting both is how the split gets quietly undone at render time.

⭐⭐ **And a contaminated thing says NOTHING**, on any channel, ever.
`ContaminableMixin` ships no `markupAugmenters` at all, and the pathogen
rows author `channels: []` — which `lint:pathogens` enforces, so the
silence is a gate rather than a habit. The test that pins it asserts that
a contaminated item's rendering **equals** a clean one's, with a control
alongside proving the same seam does report a cure; asserting the words
twice would have passed a build whose renderer said nothing at all.

## Dials

All in `AppSettingKeys` with seeded-literal fallbacks, seeded by the
platform pack. ⚠ The per-material half of the law (activation energy,
water activity) is tabulated on the `Material` and is deliberately NOT
dialled here — a global "meat spoils faster" knob would erase the point.

| key | default | what it is |
|---|---|---|
| `freshness.muMaxPerHour` | 0.35 | max specific growth rate at the reference temperature |
| `freshness.referenceK` | 303 | the Arrhenius reference, where `f_T = 1` |
| `freshness.freezingK` | 273 | at/below, growth pauses |
| `freshness.killK` | 333 | at/above, the flora dies |
| `freshness.killRatePerHour` | 6 | the death rate AT the kill temperature |
| `freshness.killActivationEnergy` | 200000 | the steepness of the death curve — what makes the kill a rate |
| `freshness.awFloor` | 0.60 | the water-activity growth floor |
| `freshness.awDefault` | 0.97 | assumed a_w for a perishable that tabulates none |
| `freshness.inoculum` | 0.002 | the seed population growth starts from |
| `freshness.ambientK` | 293 | what a gauge on a non-Thermal host reads |
| `freshness.band.{tainted,spoiled,rotten}At` | 0.25 / 0.6 / 0.85 | the band thresholds |
| `freshness.dose.onsetLoad` | 0.3 | below this an ingest carries no dose at all |
| `freshness.dose.scaleMg` | 900 | the dose a fully rotten serving carries |
| `cure.rehydrationPerHour` | 0.02 | fraction of the moisture gap a dried thing closes per game-hour |
| `cure.ambientHumidity` | 60 | the relative humidity (%) assumed where nothing authors one |
| `cure.band.{dried,drying,cured,curing}At` | 0.5 / 0.85 / 0.35 / 0.05 | presentation cutoffs for the cured-state line |

## Calibration (what a player actually feels)

The dose curve was fitted against the **shipped** `ptomaine` seed's bands
(2 / 6 / 12) and its `clearanceRate: 0.02` per game-minute — the seed was
not moved to suit the curve. Burden ≈ `dose × potency / bodyMass`, at the
70 kg reference body:

| load | band | dose (mg) | peak burden | severity | clears in |
|---|---|---|---|---|---|
| 0.30 | tainted | 0 | — | none | — |
| 0.50 | tainted | 73 | 1.0 | none | ~1 h |
| 0.60 | spoiled | 165 | 2.4 | 1 | ~2 h |
| 0.80 | spoiled | 459 | 6.6 | 2 | ~5.5 h |
| 0.90 | rotten | 661 | 9.5 | 2 | ~8 h |
| 1.00 | rotten | 900 | 12.9 | 3 | ~11 h |

Which reads: **tainted food is unpleasant and harmless**, spoiled food
costs you an afternoon, and rotten food is a real poisoning that takes
most of a day to clear. The onset sits *inside* the tainted band on
purpose — you get a smell warning before you get a dose, so **this gauge**
teaches before it punishes.

⚠⚠ **Which is true of the flora and NOT of the system.** The second
population gives no warning at all, and never will; see the header. Read
the two claims together or the doc actively misleads.

⭐ The authored `spoiled-ration` (700 mg) lands at severity 2 and ~8 h,
which is a load of ≈0.92 — "rotten", exactly what that row's name and
appearance already claimed. The hand-authored row and the derived curve
agree without either being tuned to the other, which is the check worth
having.

## ⚠⚠ Fermentation is the one collision to watch

`Vat` composes **both** `FermentingMixin` and `BulkableMixin`, so a
fermenting vessel's slot is also a spoilage-gauge slot. Fermentation *is*
deliberate microbial growth; spoilage is the undeliberate kind. Two
microbial models, one slot.

**Nothing collides today**, and only because of a data fact: every
fermenting material — `wort`, `lager-wort`, `distillers-wort`,
`red-must`, `white-must`, `wash`, `ale` — tabulates **no**
`spoilActivationEnergy`, so `Freshness.loadOf` returns 0 and never seeds.

⚠ That is luck resting on a decision, not a guard. Wort is a sugary
liquid that genuinely spoils, so the day somebody tabulates an `Ea` on it
(entirely reasonable — a ruined batch is real, and the fermentation build
already models cultures and viability) a vat will accrue a *spoilage*
load while deliberately fermenting, and the ferment's own product will
read as rotten.

**Before adding a spoilage constant to any fermentable, decide which
model owns the vessel.** The likely answer is that `FermentingMixin`
suppresses the gauge for as long as a ferment is live — a working culture
IS the flora, and it out-competes what would otherwise grow — and hands
back over when the ferment completes, which is exactly when a finished
ale starts to be spoilable. That is a fermentation build's decision, not
this one's, and it is written down here so it is a decision rather than a
surprise.

## Deliberate deferrals

- **Wetness does not feed water activity.** A rain-soaked biscuit really
  does spoil faster, and the conversion from `WetMixin` saturation to an
  effective a_w is a follow-on. v1 reads the authored value only.
- **Alcohol and acidity are not modelled as preservatives.** Spirits,
  vinegar and bitters are inert by authoring no activation energy, with
  the reason recorded here rather than pretended at in the rate law.
  Staling by oxidation (coffee, oil going rancid) is not a microbial
  story and the gauge says nothing about it.
- **Dish-as-ingredient** ("stock into soup") is out of scope: the cooked
  blend base is excluded from the craft gather's intermediate test, and
  there is a negative test that says so.
