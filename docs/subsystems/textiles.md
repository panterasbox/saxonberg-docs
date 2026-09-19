# Textiles

Clothing used to be prose. Nine shipped rows authored a description and
a slot claim and nothing else — so worn clothing contributed exactly
zero insulation, weighed nothing to encumbrance, could not get wet, wear
out, be graded, be repaired or be made, and the one soft-goods recipe in
the tree squatted in `trade-smithing` consuming a `hide-stock` nothing
produced. Meanwhile the substrate to fix all of it already shipped and
was starving: `WetMixin` soaks off `Material.waterAbsorptionCapacity`,
`ThermalRegulation` sums insulation over worn slots, and every body-plan
slot declares a `covers:` relation **thermal never read**.

This doc is the kernel half — the object, the covering stack, and the
seams. The three trades that feed it live in their own packs.

---

## ⭐⭐ The governing idea: a garment's purpose is which channel it intercepts

Nobody authors *"this is a lab coat."* You author white, cheap, woven
cloth, covering torso and arms, sitting outermost, with low insulation —
and lab-coat-ness **emerges**. The codebase already stated the rule in
`lib/vitals/Dressing.ts`: a dressing *"is not a **kind** — it's a role an
item plays."*

An apron is soiling and only soiling: outermost and cheap, it takes the
stain so your shirt does not. A lab coat is two channels at once —
sacrificial soiling **plus** station signal, which is why it is white.

⭐ **Applied to the one class that was already an exception, this
retires `Armor`.** Armor-ness is not a class; it is **material +
construction form**. A steel breastplate is a `Garment` whose material
is steel and whose form is `plate`; a linen shirt is a `Garment` whose
material is linen and whose form is `woven`. The covering walk asks the
material and the form and **never asks what class they are**, so both
already resolved through one path — the subclass was asserting something
the model does not believe. See
[materials-response.md](./materials-response.md).

---

## The object

`platform/thing/equipment/Garment` =
`Wearable(Slottable(Crafted(Durable(Constructed(Dyed(Detailed(Thing)))))))`.

| composed | buys |
|---|---|
| `Tangible` (on `Thing`) | the material, and a real **mass** — a coat is felt by encumbrance |
| `Constructed` | the form word, and with it the resist profile, the layer band, and the fabric's loft and weave density |
| `Durable` | wear-on-use condition — clothes wear out and `repair` works on them. A durable good, **not** a crafting tool |
| `Crafted` (composes `Graded`) | the as-made grade and the maker's mark, so a garment can be a recipe output; a store-bought one simply has an empty mark |
| `Dyed` | the dye application stack and its fastness |
| `Detailed` | the parts a viewer can examine, and the surface a maker's authored prose rides on |

All fifteen shipped rows (nine `clothes/`, six `armor/`) carry
`_materialPath`, `constructionForm`, `gradeBand` and `mass`. ⚠ **None
authors `clo`, and a content test refuses one.**

---

## The covering vocabulary — two sources, one ladder

A padded gambeson *is* quilted cloth, so a shirt is not armor but it
**is** a covering — hence `armor` → `covering` throughout
`Construction`, and `quilted` as the fifth resist-bearing form.

| | where | may a pack add one? |
|---|---|---|
| **resist-bearing** — `plate` `mail` `padded` `quilted` `hide` | a closed kernel `as const` | **no** |
| **non-resisting textile** — `woven` `knit` `felted`, lace or netting later | `/stuff/idea/fabric/<key>` rows, class `/platform/idea/material/Fabric` | **yes** |

⭐ **Content never authors a resist profile.** One kernel constant —
`TEXTILE_RESIST_PROFILE`, `poor` on all three mechanical channels —
answers for every textile form at once. Content chooses drape, loft and
weave; **the kernel decides that cloth resists poorly.** A pack adding
`lace` changes nothing whatever about combat.

Full rationale, the four simultaneous constraints, and the
`FabricCatalogue` boot-ordering argument:
[materials-response.md](./materials-response.md) § *The covering domain
has TWO sources*.

---

## The covering stack — one walk, on the wearer

`SlottedMixin` answers about its own slots. ⚠ **There is no
covering-stack Api and there must not be**: a covering read is one host
answering about its own slots, which is none of the four mandates
`check-object-verbs` allows.

| method | answers |
|---|---|
| `wornStack()` | everything worn, outermost-first |
| `coveringAt(part, {includeHeld})` | the covering over one body part, outermost-first |
| `outermostAt(part)` | which layer takes a deposit |
| `insulationAt(part)` · `bodyInsulation()` | clo over a part · surface-weighted over the body |
| `windproofing()` | how well the outermost layer breaks a wind |
| `concealmentOffset()` · `attentionFactor()` | the two derived social/arcane reads |
| `wouldLayerViolate(candidate)` | the ladder refusal |

⭐⭐ **Three logic singletons hand-rolled the same outside-in walk** —
the trauma covering walk, the struck-site armor stack, and the
conduction walk — and all three now call `coveringAt`. Each already
holds the host, so the call **drops** a parameter rather than adding an
Api hop.

⚠ The shared method returns the **occupants** and leaves the narrowing
to each caller, because they want different things: the conduction walk
cares about a rubber sole's conductivity and not whether it declares a
form. Requiring `Constructed` in the shared walk would have silently
changed it.

**The ordering rule, one comparator, one place:** *form sets the band;
wear-order breaks ties inside a band.* Bands come from
`Construction.getLayerDepth()` (padded 0 · quilted 1 · hide 2 · mail 3 ·
plate 4, and a fabric's `layerBand` on the same ladder); anything with
no covering form sorts innermost. Wear order is slot insertion order,
and the persistence spine re-wears through `occupyAll` in the captured
order — **durable with no new field**.

⚠ `WearController` refuses only the **inversion** (a shirt over plate,
`layer-order` note). Shirt-vs-coat is the player's call and its
consequence is being cold.

⚠⚠ **Covering slots need `capacity` > 1 and did not have it.** Every
wear slot took the default 1, so a body held exactly one torso garment —
the shipped gambeson and hauberk could never be worn together, and the
entire outside-in model had nothing to walk. The biped's and quadruped's
covering slots carry `capacity: 4`. A cap rather than unbounded, because
"wear forty shirts" would otherwise be free insulation: **the ladder
decides the order, the capacity decides the depth.**

---

## Insulation — derived, per part, and wet

`clo` **derives and is never authored**; the thermal sum is
**surface-weighted per body part**, not body-wide. Both were the same
defect: a number that should have come from physics was a flat sum over
an authored field, and a body-wide sum *cannot* teach that bare
extremities cost you.

The arithmetic, the wet model, the organism carve-out on wet mass, and
the windproofing term: [thermal.md](./thermal.md) § *Worn insulation*.

---

## Fit — two numbers and one stamp, and the lineage seam

```
statureM   = species.getStature()
girthIndex = √(massKg / statureM)
```

⭐ `massKg` is `Creature.getMass()`, which already reflects composition —
**that is the lineage seam and it is one line.** The stamp is three
scalars (`cutToBodyPlan` / `cutToStature` / `cutToGirth`), and an absent
stamp means **stock**, resolving to the plan's average body, so all
fifteen shipped rows read as ill-fitting hand-me-downs with no content
edit. Consequences, the wrong-body refusal and the distance refusal:
[embodiment.md](./embodiment.md) § *Fit*.

⚠⚠ **`seamAllowance` is the stamp's twin and lives beside it on
`WearableMixin`** — both are *what the cutting did to this cloth*, and
both must survive `cut` → `sew` → `alter`. It briefly lived on the
tailoring pack's `CutPieces` alone, which meant `sew` wrote it through
an optional call onto a kernel `Garment` that had no such method: the
cloth folded in at `cut` vanished silently, and `alter` refused to let
out **every garment in the game** with *"was cut close. There is nothing
folded in the seams"* — a sentence that was never true of any of them.

⭐ **The unit test passed throughout**, because it exercised `CutPieces`
in isolation and the break was in the chain. The assertion that catches
it is that *both ends carry the field*, which is only true while it
lives on the shared mixin.

---

## Dye, wash and fade

`DyedMixin` (`lib/material/Dyed.ts`) stores the **application stack** —
`[{dyestuff, mordant, strength, transmitR/G/B}]` — and a `fastness`,
never a colour word. Overdyeing is arithmetic, fading is desaturation,
and authors author dyes rather than colours. It rides `ColorTag`, the
seam `lib/perception/Light.ts` already reserved.

### ⭐⭐ The colour is SUBTRACTIVE, and that is why it is not RGB

`lib/perception/Colour.ts` is the arithmetic. A colour is stored as
**transmittance** per channel — a yellow dye is a thing that absorbs
blue — and a stack **multiplies**, which is what layered filters do.

⚠⚠ **RGB gets the one case wrong that the model exists for.** RGB is
additive: blue over yellow added is white, multiplied as bytes is
black, and neither is green. Transmittance gives green, which is how
green was actually made. RGB appears only at the edge, as `toHex()`.

| | transmits | |
|---|---|---|
| weld + alum | `0.88, 0.78, 0.14` | passes red+green, eats blue |
| woad (vat) | `0.12, 0.40, 0.75` | passes blue, eats red |
| **product** | `0.42, 0.52, 0.26` | **a green no row contains** |

⭐ **`strength` is the lerp toward undyed**, so fading is desaturation
for free and cloth walks continuously back toward the fibre instead of
falling off the `legibleAt` cliff. Strength 0 is the identity — which
is exactly why the mordant's zero-strength marker can sit on the stack
without colouring anything, and why the triple is **optional**: a
mordant is genuinely colourless, and an entry without one contributes
nothing rather than multiplying toward black.

⭐⭐ **Measured, and better than the claim.** Weld over a *shallow* vat
is olive; green needs a vat built up over many dips. That is how
Lincoln green was actually made, and no lookup table could have
produced it — a table would have had to author "olive" and "green" as
two outcomes and pick one at authoring time. `Colour.test.ts` pins it.

⚠ **Four independent entries per dye, not one hue under a per-mordant
filter.** The metal ion is part of the chromophore, so alum-madder (a
clear red lake) and iron-madder (a purple-brown one) are different
pigments. "Iron saddens everything" would be tidier and false.

⚠ **The base is neutral white today.** The honest base is the fibre's
own colour — unbleached linen is fawn, wool cream to brown — which is
what makes *"linen was worn undyed and wool was the coloured cloth"*
true rather than asserted. That needs a colour on `Material` rows and
is the obvious next step, deliberately not stapled to this build.

### The word, the tint, and the dark

The palette in `Colour.ts` is **nearest-neighbour naming only** — the
arithmetic already happened. It is deliberately muted *and* carries a
**pale band** (`pale blue`, `sage`, `straw`, `oatmeal`), which is
load-bearing: a bath tops out around strength 0.8 and a vat builds from
0.35, so most real cloth lands light, and a palette of saturated words
only would answer "grey" for a perfectly good pale blue.

⭐ On the wire it is `<color value="…">` — the seam that already
existed, whose grammar says *"a thing's color is a real property of
what's perceived"* and **never a raw hex**. The client collapses the
words onto the theme's eight tints. That is not a loss: the sentence
already said the colour, so the tint is emphasis and never the channel.

⚠⚠ **`DYE_COLOR_TAGS` lives in `@saxonberg/types` because the two ends
must not drift.** An unlisted token falls through to `neutral`
*silently* in `MmlRenderer`, so a word added server-side and forgotten
client-side would render a whole textile economy grey with nothing
raised. Both ends assert against that array.

⚠ **The colour line is gated on light**, reusing
`PerceptionApi.canMakeOutMarks` rather than inventing a lux cutoff —
*"knowing that a scroll is there and being able to read it are
different questions"*, and colour is the same question. In the dark it
says **"too dim to tell"** rather than nothing, because an absent line
reads as *undyed*, which is a different fact and the wrong one.

`wash <garment>` takes the launder branch (`wash`'s target already
required `CraftedMixin` and a `Garment` composes it — no arg change, no
new verb). Each wash strips colour in proportion to `1 − fastness`, so
an **un-mordanted piece washes straight out on the first launder** and a
well-mordanted one survives many. ⭐ Hue comes from the dyestuff;
**durability comes from the craft** — which is why competence in dyeing
buys fastness and repeatability and never a brighter colour.

⚠⚠ `CraftVessel.soiled` is a **different concept sharing a word** — *is
this vessel claimable for a fill*, binary by necessity, owned by
crafting. The two are not folded together and a test says so.

---

## ⭐ The soiling seam — a METHOD, not a signal

Textiles owns the **layering model**, so it answers *which layer takes
the stain*: `wearer.outermostAt(partKey)`. That is the whole seam, and
the covering stack needed the method anyway.

**Out of scope, and asserted absent:** the gauge, its bands, its
attributed deposit log — those belong to
room-condition, and a
second gauge would be a third parallel representation of one idea.

⚠⚠ **And no `soil.*` event either.** The plan proposed one; it was wrong
three times over. Room-condition's *"attributed events"* are **ledger
records** while `EventApi` is a **broadcast bus**, so an emit would not
feed the log it was meant to. Nothing in this build soils anything, so
it would have **zero emitters**. And zero listeners, by design. A
soiling deposit has an actor, a target garment and a body part — **a
local interaction is a call, not a broadcast.**

⚠ Dirt is **act-deposited and freezes in absence**: a coat in a wardrobe
does not get dirty. Soiling is not time-integrated, and there is no
clock stamp anywhere in this build.

⭐ **The apron is designed and seamed now and lights up when
room-condition lands** — it is content only: cheap, outermost band, wide
`slotClaims`. Until then it is an honest, cheap, ugly garment.

---

## The presentation — the card enumerates, the prose summarizes

Two resolutions of one subject, not two subjects. Prose that enumerates
is just a worse card.

- **`worn`** is a distinct `DETAIL_FIELDS` projection and a **partition**
  of `contents` — the body half against the pack half. See
  [card-surface.md](./card-surface.md).
- ⭐ **Conspicuity reads SATURATION, not strength.** The comment always
  claimed *"a bright saturated one is loud"*, and reading strength could
  not tell a pale blue from a deep red at the same dip. The folded mix
  can, so a shallow vat is quiet and a madder red is not — the third
  claim the colour model turned from prose into behaviour.
- **The impression line** is a `markupAugmenter` on `SlottedMixin`: a
  one-sentence gestalt that **names no individual garment**, folded over
  whatever facts resolve, with phrasing **seeded rather than drawn**.
- **The authored half stays authored** — bearing, manner, the scar live
  on `PersonaMixin`'s claimed layer, untouched.
- **Getting dressed is one command**: `equip set <name>`, a **stanza**
  on a dressing view rather than a verb of its own. It shipped as `wear
  set` and moved when `equip` landed — see
  equip-slate.md *(slate retired 2026-09-06 — fully absorbed)* and
  [slot.md](./slot.md).

---

## The social and arcane seams

- **Concealment**: `getConcealment()` derives from the authored base plus
  the worn covering, and the scale gains `conspicuous` below `obvious`.
  ⚠ `perceives` saturates below `obvious`; the band earns its keep in
  `hideLevelFor`'s floor. See [concealment.md](./concealment.md).
- **The hood/veil interlock**: `attentionFactor()` multiplies a held
  binding's standby draw, so a **mundane** hood makes an **arcane** veil
  cheaper to hold. ⚠⚠ Faculty is capacity, never access. See
  [magic-items.md](./magic-items.md).
- ⚠ **No engine gauge from dress to regard.** *Engine measures; subject
  values.* NPC reaction to dress lives in a **brain**, in a pack.

---

## What this build deliberately does not ship

- **`SoilableMixin`, its bands and its event** — above.
- **Leatherwork and tanning** — nothing produces hide, so a tannery now
  would tan imports. Waits on ranching or a butchery faucet.
- **Wool and everything on its left edge** (`shear`, scouring) — waits on
  ranching. ⭐ The launch story is *flax gives you woven cloth; wool
  gives you everything else* — felt, fulling, knitting, saturated
  colour — which makes wool's arrival a real event rather than a second
  row.
- **`full`** — fulling works because wool scales interlock; **linen
  cannot be fulled**, and the same scales are why plant fibres do not
  felt at all. The `felted` fabric row ships **unreachable** (the
  `chalcopyrite` precedent) so the vocabulary is honest rather than a
  promise in prose.
- **Cotton and silk** — a second cellulose fibre is more content and no
  new lesson.
- **Synthetics, mills, mass production** — trades ship medieval and
  advance by exercised disciplines. ⚠⚠ **Do not add a `kevlar`
  construction form**: material scales the resist magnitude, so a
  high-performance textile is a `woven` fabric of a tough material.
- **Terrain-matched camouflage** and the viewer-side detection equipment
  term — the search slate's.
- **Hair dye** — lineage must ship appearance substrate first.
- **Individual body variance within a species** — lineage phase 2, and
  it feeds an existing seam.
- **A laundry vocation.** Water is a precondition, not a consumable, so
  the care loop is not an errand per wash.

---

## The chain, as it ships

```
        ┌─ trade-farming ─┐   ┌──── trade-textiles ────┐   ┌ trade-tailoring ┐
seed ─▶ plant ─▶ SHEAF ─▶ pit ─▶ line ─▶ yarn ─▶ BOLT ─▶ pieces ─▶ GARMENT
        (flax)          (ret)  scutch   spin   weave     cut       sew
                          │                      │                  ▲
                          └─ over-ret ▶ ruined    └── green ▶ bleached│
                                                                     │
        ┌──── trade-dyeing ────┐                                     │
        madder · weld · woad ─▶ mordant ─▶ dye ──────────────────────┘
```

**Three packs, four stages, eight verbs** — `scutch` `spin` `weave` ·
`mordant` `dye` · `cut` `sew` `alter` — plus `measure figure` and the
saved-set stanza, both **stanzas on shipped views**, so the chain itself
adds those eight and no more. (`equip` / `unequip` shipped on the same
branch and are not the chain's: they are the dressing surface, and the
slate for them is equip-slate.md *(slate retired 2026-09-06 — fully absorbed)*.)

⚠ The fitting stanza is `measure figure`, and it shipped for a day as
`measure customer`. Every other stanza on that view names the CHANNEL
being read (`light`, `density`, `strike`, `dip`); `customer` named the
subject's SOCIAL ROLE, which is the wrong axis — and a role that is not
even true off the shop floor, since a quartermaster measuring recruits
or a parent measuring a child is the same act with no customer in it.
⭐ The subject now **defaults to yourself**, which makes the un-shopped
case the default one. ⚠⚠ And the header's claim that *"the attendant
lease IS the consent"* was **prose over an empty check** — nothing ever
read a lease. The real gates are an organism and a `MeasureBook` in
reach, and the doc now says so. The honest consent model, when somebody
builds it, is that **the tape is the instrument and the instrument is
what needs the permission** — `measure strike` already needs a
surveyor's compass, and by eye you would get a rough figure at worse
resolution. That is a slate, not a rename.

### ⭐⭐ The tooling: every verb has two rungs, and the instrument affords it

| verb | rung 0 — portable, bad | rung 1 — fixed, good |
|---|---|---|
| `scutch` | scutching board | — |
| `spin` | drop-spindle | spinning-wheel (rate 3, fine) |
| `weave` | hand-loom | broad-loom |
| `dye` | household-vat | dye-vat · woad-vat |
| `cut` | **shears** (rate 1, coarse) | cutting-table (rate 2, fine) |
| `sew` · `alter` | **needle-case** (rate 1, coarse) | the store's sewing-machine (rate 3, fine) |

⚠⚠ **Tailoring shipped without any of this and had to be corrected.**
All three of its verbs were declared on `CuttingTable` — a room fixture
— against `crafting.md`'s own rule that *"the working verbs are
instrument-conferred… on the class of the thing that performs the act"*.
So `cut` was impossible outside a shop, there was no rung zero at all,
and `sew` was afforded by furniture while its capability lived in a kit
the affordance never mentioned. See
[crafting.md](./crafting.md) § *Split by what performs the act*.

⭐ **What the bench sells is cloth, not permission.** Shears cut the
same shirt anywhere; `cut` charges **one extra unit of cloth** for a
`coarse` instrument, and the table's `control: fine` buys it back. A
cloth cost rather than a time cost on purpose — a bolt is capital
(~20 unskilled days), `cut`'s whole tight/generous axis is already
denominated in cloth, and charging minutes would have made the bench a
convenience instead of an investment.

---

⭐ **Preparation and finishing ship with ZERO verbs.**
`MaturingMixin` runs the retting pit's clock and the bleaching green
is the same shape applied to weather. What you do is judge the moment.

### ⚠ The two stage names that are not stage names

| stage | flax's instance | wool's instance |
|---|---|---|
| `prepare` | **retting** | scouring |
| `finish` | **bleaching** | fulling |

Nothing in `trade-textiles` treats "retting" or "fulling" as the name of
a stage. Wool scours, cotton gins, silk reels, a synthetic extrudes —
each arrives later as its own `MaturationProfile` over its own material,
and the pack does not change.

### ⭐ Three seams cut so the ladder stays open

1. **The stage is `prepare`, not `ret`** (above).
2. **The chain begins at *fibre-exists-as-a-material***. `scutch` and
   `spin` gate on a MATERIAL tag, never on the flax row; `dye` takes a
   dyestuff material, never a crop. ⭐ That second one is the seam
   **Perkin walks through** — mauveine (1856) destroyed madder and
   indigo agriculture in a generation, and for a synthetic ever to plug
   in, the entry point has to be the material.
3. **The bottleneck assertion is scoped to the shipped tech level**, in
   the bench's own name.

⚠⚠ **Do not add a `kevlar` construction form.** `MaterialLogic` scales
the resist *magnitude* by the material while `responseFor()` supplies
only the *shape*, so a high-performance textile is a `woven` fabric of a
tough material. That is `response = f(mechanism, material,
construction)` doing its job.

## ⭐⭐ Throughput at bed scale — the measured finding

`mill-throughput.bench.test.ts` reads the shipped dials and measures one
bolt:

| | by hand | with the wheel |
|---|---|---|
| scutch | 0.33 h | 0.33 h |
| **spin** | **3 h** | **1 h** |
| weave | 0.5 h | 0.5 h |
| spin : weave | **6×** | **2×** |

Six is dead centre of the historical band — five to ten spinners kept
one hand-loom weaver supplied.

⚠⚠ **The wheel closes the gap threefold and does NOT flip it**, and the
plan expected otherwise. At any honest ratio it cannot: three does not
clear six. ⭐ **That is the history.** The wheel did not solve the
spinning shortage; the machine that did was the **jenny**, at eight
spindles — and eight is the first thing that clears a gap of six.
Fitting the durations until a wheel flipped it would have deleted the
most famous fact about this trade to satisfy a sentence. The bench
asserts what is measurable, pins the ratio inside the historical band,
and prints the rate that would flip it.

⭐ **And the bed-scale worry did not fire.** One Hinkley bed yields
~12 bolts per game-year (a stock shirt is ~1 bolt-unit), so the ratio is
feelable at smallholder throughput. The anticipated "document the
shortfall instead" outcome was not needed — the number is printed either
way, which is what made it answerable rather than arguable.

⚠ Retting is reported separately: ~14 game-days **elapsed**, zero
attended. **A wait is not labour**, and folding it in would make
preparation look like the bottleneck when it is the one step you can
walk away from.

## The three businesses

| | shape | wage |
|---|---|---|
| the Wharfside **mill** | produces + **consigns** — no counter | spinner **3** · weaver **5** |
| the Wharfside **dyehouse** | walk-in service, `discipline: line` | dyer **5** |
| the **tailor's shop** | by appointment, `discipline: appointment` | tailor **6** |

⭐ The mill does not retail. It consigns through the shipped `consigns`
brain, so cloth reaches players in the Terminus general store where they
already shop — less new content, and the good is where the buyer is.

⚠⚠ **Read the wage column: the bottleneck job is the worst-paid one.**
That is the uncomfortable historical fact, and precisely *why*
mechanising spinning was both so profitable and so socially disruptive.
**Do not "fix" it upward for fairness** — the wage table teaches the
same lesson as the bench and the price ladder, from a third independent
direction, and flattening it deletes the lesson rather than the
injustice.

### Prices, derived from wages rather than picked

`wageRate` is minor units per game-hour; an 8-hour unskilled day is 24.

| | minor |
|---|---|
| `measure` | **free** — the loss-leader that fills the book |
| alteration | 20 |
| stock shirt / trousers | 50 / 60 |
| stock coat | 200 |
| bespoke shirt / coat | 120 / 500 |
| plain linen, per unit | 40 |

A spinner on 3/hr earns 24 a day, so a stock coat is **~8 days of her
own labour** and a bespoke one is a month. ⭐ **A bolt is capital**
(~480 minor ≈ 20 unskilled days ≈ 4 coats), which is what makes `cut`'s
waste decision bite. ⚠⚠ **The numbers move together or not at all.**

## ⭐ Siting — the textile chain IS the nuisance-trade chain

Retting ponds stank badly enough to be banned upstream of towns;
dyehouses stank (urine was the classic woad vat); the tannery is Stage
C's and stank worst. That is a coherent industrial identity rather than
three coincidences, and it is what makes these read as **industry**
rather than as crafting stations.

So the mill and the dyehouse sit at Wharfside by the water, and the
tailor's shop sits off Mayfield Row where there are customers and good
light. **The answer differs at each end of the same chain**, which is
what a siting decision is.

## Magic — the craft wins on permanence

All three trades land on one shape, which is the sign the doctrine is
doing work rather than being recited.

- **Textiles**: `create·light` over a bleaching green is the obvious
  shortcut, and the refusal is physical rather than fiat — *bleaching is
  photochemistry; a glowlight is illumination*. It teaches
  light-as-energy vs light-as-seeing.
- **Dyeing** is where magic genuinely endangers a trade, and Kell's
  answer beats a refusal: **a magically-fixed colour is a binding**, so
  it fades when the charge runs out. ⭐⭐ *Magic gives you a brilliant
  colour that does not last; the craft gives you a modest one that
  does.* A player should take that bargain once and regret it — and it
  lands exactly on the muted palette.
- **Tailoring**: `alter`'s ceiling is the seam allowance **because
  matter is conserved**. A spell cannot conjure cloth, so magic might
  alter faster and never further. It needed no code.

⚠ Any later proposal that lets a working beat a craft *durably* is
contradicting Kell, not extending it.

---

## Cross-references

[materials-response.md](./materials-response.md) ·
[thermal.md](./thermal.md) · [embodiment.md](./embodiment.md) ·
[slot.md](./slot.md) · [card-surface.md](./card-surface.md) ·
[mql-subscription.md](./mql-subscription.md) ·
[concealment.md](./concealment.md) · [stealth.md](./stealth.md) ·
[magic-items.md](./magic-items.md) · [crafting.md](./crafting.md) ·
[race.md](./race.md) · [encumbrance.md](./encumbrance.md) ·
[maturation.md](./maturation.md) ·
[content-packs.md](./content-packs.md)
