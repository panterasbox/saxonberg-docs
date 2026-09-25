# Maturation — the durative transform

**The liquid leg's kernel** (fermentation build, 2026-09-01): a batch
converts over game-time under conditions. *Growth accretes, maturation
converts* — the husbandry SHAPE (reconcile lazily on read, staged, no
far-past guard, no linkdead freeze) with a different equation: the mass
is already present and what changes is what it IS.

## ⭐⭐ Which failure is it? (`stalled` / `killed`)

A ferment fails in three ways that feel completely different to whoever
owns it — and until the grain chain the augmenter said **the same
sentence for all three**:

| what happened | what you should do | what it used to say |
|---|---|---|
| too cold — **held** | carry it somewhere warmer | *"A first few beads track up through it."* |
| scalded — **killed** | throw it away | *"A first few beads track up through it."* |
| genuinely just started | wait | *"A first few beads track up through it."* |

That is not cosmetic. It makes the lesson **unlearnable**: you cannot
tell *move it* from *bin it* from *wait*, so the mistake teaches nothing
and the same person makes it again next week.

`MATURATION_LINES` now carries `stalled` and `killed` for all three
mechanisms, and the augmenter branches on the host's temperature **now**
against the profile's own `stallBelowK` / `stallAboveK` / `killK`.

⭐ **Read at look-time, never stored** — so it is a fact about where the
vessel is standing, and carrying the trough to the hearth changes what
the next `look` says. That is what makes the recoverable failure
*actually* recoverable rather than merely described as such.

⚠ **Killed is checked before stalled**, deliberately: the recoverable
reading must never shadow the unrecoverable one, or somebody carries a
dead vat to the fire and waits.

⚠ **Known limit, recorded rather than claimed correct:** a batch is
"killed" by *being hot now*, so a scalded batch that has since cooled
reads `stalled` again. A culture carries real `viability` and does not
have this problem; a batch has no stored dead state. The honest fix is a
`killed` flag on the batch, and it is a deferred seam rather than a thing
this build pretended to do.


## ⭐⭐ It is MATURATION, and fermentation is one mechanism of it

It shipped as `FermentingMixin` and was renamed once the substrate
outgrew the name. The shape is *put it somewhere, time and conditions
transform it, judge when to stop* — and fermentation is one mechanism
with that shape, not the only one. Flax **retting** joined it (genuinely
fermentation: pectinolytic bacteria digesting the pectin that glues bast
fibre to the woody core) and so did **grass-bleaching linen**, which is
not fermentation at all but photochemistry.

⚠⚠ **A data model that asserts a false mechanism is a defect on a
platform that teaches**, and this one asserted it in prose too: the
sensory augmenter was written for a wine cellar and never asked who else
composed the mixin, so a bleaching green — an acre of grass with linen
pegged out in the sun — rendered *"It bubbles steadily, a yeasty breath
rising off it."* A retting pit, whose entire authored character is that
the smell arrives a second before you do, reported when finished that
*"the air over it is clean."*

⭐ So a profile now declares its **`mechanism`** — `microbial ·
photochemical · chemical`, a closed vocabulary — and the prose reads it.
Same move `Dyestuff.chemistry` already makes: one substrate, two
chemistries, the DATA says which. **Nothing in the clock branches on
it**; it steers only what the thing looks like while it works.

⚠ Names that stayed: `getFractionConverted`, `batchStrain`, the `turned`
phase. They are mechanism-neutral or belong to the microbial case, and
renaming them would have been churn without a claim behind it.

Kernel: `lib/maturation/` (`Maturing.ts`, `MaturationProfile.ts`),
`platform/idea/maturation/MaturationProfile` (the instanceable concrete —
the Material split), `platform/thing/Vat` (the one concrete every
trade's vessel rows name), `platform/idea/MaturationProfileCatalogue`
(the self-warming roster — **no MaturationApi**: the queries are statics
on the owning class, and an operator-shaped warm does not belong on a
consumer Api). Zero new verbs: crush/mash/distil are
recipes `order`ed off unpriced work boards (the kitchen-menu shape);
rack = `pour`, seal = `close`, bottle = `fill`, pitch/feed = `pour`.

## The transform rides the VESSEL (D2)

`MaturingMixin` composes on a Bulkable host (composition-validated)
— bulk matter has no identity, vessels do (the pot-as-bed precedent).
`Vat` = `Fermenting(Crafted(Sealable(Thermal(Bulkable(Detailed(Thing))))))`;
a carboy, a conditioning bottle and the cask are ROWS over it (sizes
are data). The vat's `category` defaults `'vat'` (rows depart).

**The batch is detected, not hooked**: the mixin keys its batch to the
interior material path and notices a change on reconcile — a new
material is a fresh fill (a fresh batch), an emptied interior resets
to idle, and the mixin's own product/turn/lees swaps re-key so they
never read as fills. A same-material top-up continues the batch (the
payload rule).

## Profiles are the whole authoring surface (D1/D4)

`MaturationProfile` rows (any root's `idea/maturation/` subtree; matched by
the must's material TAGS against `inputCategory`; a double match is a
warned authoring error resolved by lowest key — never a roll):
temperature band (`stallBelowK`/`happyK`/`damageAboveK`), `ratePerDay`,
`productMaterial`, `turnedMaterial` + `turnDays` (D3 — an OPEN finished
batch becomes vinegar; sealed holds), `sealedOnly` (bottle/cask
conditioning — sparkling and real ale), `killK` (hot pitch dies; a
batch run past it goes sterile — the stuck ferment), `stallAboveK`
(authored dormancy — lager's warm refusal), `requiresStrain`,
`spontaneousLagDays` (a sterile must in an OPEN vessel catches wild
flora after the lag; sealed never starts), `wildStrain`,
`leesFraction`/`leesMaterial`, culture kind: `strain` + `starveDays`,
and the INERT `foreshotCharacter` (P10 — the deferred cuts rung's
seam). ⭐⭐ **That rung has a SECOND consumer and it is worth building for
the first one anyway:** fractionating crude is the same act as making the
cut — *separate a mixture by boiling point, and the operator decides
where the boundaries fall*. So the whiskey cut and the refinery are one
mechanism with different feedstocks and a different number of cuts, and
building the rung for whiskey (a medieval-epoch build that pays for
itself in a shipped trade) pre-builds refining for free. See
drilling-slate § *Refining is the
distiller's deferred cuts rung*. **A new drink is rows alone** — proven by the cider test
(`world/__tests__/fermentation-cider.test.ts`).

`MaturationProfileCatalogue.postRegister` stands the roster up, eager via
the platform pack's `boot:` manifest (`sync-read`) — the
reference-Ideas-inert-at-boot rule, self-warming (the
Discipline/Recipe catalogue shape rather than an `Api.boot()`
sequencer line). `MaturationProfile.forMaterial`/`.byKey`/`.all` read the
live population (the `everyMaterial` shape, no cache — statics on the
owning class, no Api).

## The numbers are derived and discoverable (D4)

Starting sugar reads off the input material (`nutrientAmounts.sugar`,
g/L). ABV% = converted g/L ÷ 17; specific gravity = 1 + remaining ×
0.0004 (both constants exported). Windows are segmented at EVENTS
(seal toggles and moves reconcile first — the `Vat` overrides), each
credited at the vessel's own reconciled `ThermalMixin` temperature —
the cold cellar is a PLACE (`_temperature` on the room), never a rule.
Grade: `_worstStretch` is the monotone-min damage satisfaction (1
at/below `damageAboveK`, 0 fifteen K past it) mapped through the
husbandry harvest thresholds and written to the host's Graded face
every reconcile. No `Math.random` in the tree (source-scanned).

## The transfer seam carries the batch's identity (D6/P3, W0)

`BulkableApi.transfer` into an EMPTY destination carries the source
holder's grade band (Graded→Graded) and the maker's mark
(Crafted→Crafted: maker/recipe/craftedAt) — the payload rule applied
to identity. `GradedReceptacle` (⇒ every stock bottle) is Crafted now,
and a MARKED bottle appends its `renderVerdict()` to look (W7). The
mismatch branch offers a cross-material pour to a Fermenting interior
as an INOCULATION before declining: a strain-bearing pour into a
sterile batch is a PITCH (the jar's `batchStrain`, or a material's
`strain:<x>` tag), sugar into a culture is a FEED.

## Yeast, wild and kept (D13/D14)

A microbe is a MATERIAL (D13): the living thing is the lees
(`biologicalSource` → the fungi species rows — S. cerevisiae,
S. pastorianus in base-library). **The lees split at the rack** (P12):
a finished batch holds `leesFraction` back as a rack floor
(`getBulkAvailable` shadow); racked down to it, the residual swaps to
`leesMaterial` and re-keys AT ONCE as a culture batch — strain from
the lees' culture profile, the Crafted mark untouched (the trace back
to the harvested batch). A culture's "conversion" is viability:
starves over `starveDays` (×0.25 below the band — the cellar; ×3 past
damage — the shelf), killed above `killK`, fed by a pour of
sugar-bearing must; dead is terminal and pitches nothing.

## Byproducts and the cellar's air (P11/D12)

Recipes carry `outputResidue` (template + count), cloned beside the
maker at resolve — pomace at the press, spent grain at the tun; both
are `compost`-tagged feed (the reverse B2B: the growers buy nitrogen
back), pomace distils to grappa, and unsold residue stands where it
was left (the ambient-burden rule). A CONVERTING batch drains its
room's authored `air` reserve; an unventilated cellar flips to a
`carbon-dioxide` atmosphere (a fourth row in every biome table —
unbreathable, no contaminant: asphyxia) and an open door recovers it.

## The producer beats (P7)

`lib/behavior/cellars.ts` — kernel commons beside `consigns` (both
sibling trades bind it; a cross-trade dep is what the `distribution`
cut removed, D10). One concern per beat off the home floor's vats:
bottle-and-consign a finished vat (the W0 seam stamps every bottle);
`order` a crush/mash off the board, pour the bucket, pitch the jar
(the lager leg carries the cold mash down and pitches the cold
strain); the distills leg ignites the still (its own furnace — the
351 K recipes), runs the charge, compounds, and consigns SPIRIT as
well as gin (the vintner's fortification buys it — the B2B leg);
compound legs for the vermouths; a buys LIST on the house card.
Bounded, literal player verbs, home in `finally` — the farms rules,
asserted by the bounded test.

## Reads (D5) and credits (D9)

Senses first: the ferment augmenter derives the vat's face from state
(bubbling / sweet-and-silent / still / the vinegar edge / the creaming
or dead culture) — never a number. Numbers cost the **hydrometer**
(`measure density <vessel>`, a measure SUBCOMMAND; instrument class +
generic-objects row): a Fermenting vessel answers its batch's specific
gravity — the two-vat experiment recovers the authored slopes.
Credits: recipes carry `discipline: fermenting|distilling` (craft-
resolve credits at the act); the rack and the bottling credit
`fermenting` in Pour/FillController when the source batch is finished.

## The switchover (W8) and the home brewer (D15)

The winemaking/brewing floor-product faucets are RETIRED — the vessel
faucets (`wine-bottle`, `keg`, `spirit-bottle` empties at target)
supply the glass, and every drop on the distribution counter is a
batch a beat made. Bottling and farming faucets are untouched; malt
is the honestly-labelled imported-input faucet (base-library material,
the sack + par at the distribution counter). The homebrew kit rides
the general store (carboy + culture jar in the commons, small
press/tun/still with their trades, all stipend-priced): verbs ride the
instrument, never the venue — the same profiles, physics being fair.

**The venue archetypes** (`content/archetypes/` — winery, brewhouse,
still-house): each trade's needs as capability slots, the hospitality
shape, over a new `vesselKind` need (the vat is capital that is
neither a tool nor a bulk source; the need reads the kernel's own
vessel-kind `category`, empty or full — `vesselKind`, not `vessel`,
since bare `Vessel` is the enterable-container class). `industry` stays null — the
shared `fermenting` discipline would cross-derive the sibling trade's
tools, so the floors are authored residue; the work boards (Menus) are
deliberately not needs (the bathroom rule). Reported by `survey`,
never enforced; a second winery anywhere satisfies from any rows.

## Generalization notes — bread, yogurt, cheese (parked 2026-09-01)

Assessed against the shipped model so the coming food trades inherit
the analysis instead of rediscovering it:

- **Yogurt — rows only.** Milk's lactose IS `nutrientAmounts.sugar`
  (~50 g/L, honest); a profile with `requiresStrain` + a ~316 K happy
  band (its culture authors the higher `killK`); pitched by `pour`.
  Backslopping falls out: a culture-kind profile over the yogurt
  MATERIAL itself makes a kept jar the culture. Needs one taxonomy row
  D13 pre-answered: a `bacteria` kingdom clade beside `fungi` for the
  Lactobacillus/Streptococcus species rows.
- **Bread — designed for (D14's baking seam).** The starter is a kept
  culture over a levain material, fed by a flour-water pour — works
  verbatim. The rise is a proofing-bowl row (Vat family) + a dough
  profile, and D3's machinery hands over-proofing over for free
  (finished + open + fractional `turnDays` → collapsed dough as the
  turned material). The exit from bulk is CRAFTING's: the bake is a
  recipe (dough bulk + oven heat → a tangible loaf), grade flowing
  weakest-link. The lactic family will eventually want an ACIDITY read
  (the hydrometer pattern, a different number) — the SG/ABV reads mean
  little for dough.
- **Cheese — fresh yes, aged is the gap.** ⭐ The lees split IS a
  curd/whey split: cultured milk sets, the whey pours off down to a
  residual fraction (`leesFraction`, proportions inverted) and the
  residual swaps to curd — no new mechanism; whey rides the P11
  byproduct economy (ricotta over it, feed); pressing a wheel is a
  recipe. **The gap: ripening a DISCRETE thing.** A wheel is a Crafted
  Thing, and `MaturingMixin` requires a Bulkable interior — months in
  a cave has no mechanism (note whiskey does NOT share this gap: it
  ages as bulk in a Vat-family cask, a future aging profile). Aged
  cheese + cured meats are the follow-on build's question.
- **Two coarsenesses to design with, not around:** (1) one profile per
  input material (the double-match rule) — strain-dependent FORKS from
  one input (raw milk → yogurt vs clabber) take process-differentiated
  input materials, the lager-wort move, usually honest anyway; (2) the
  transfer seam's domain branches (thermal blend · grade/mark carry ·
  inoculation) are explicit, narrow-gated steps in `BulkableLogic` —
  fine at three; if a FOURTH domain wants in (whey handling, dyeing,
  poisons), that is the moment to generalize to a host-side participant
  hook rather than a fourth branch.

## File map

- `lib/maturation/Fermenting.ts` — the mixin: reconcile, batch detection,
  strain/culture/lees, the cellar air, the sensory augmenter
- `lib/maturation/MaturationProfile.ts` (+ the query statics) →
  `platform/idea/maturation/MaturationProfile`
- `platform/idea/MaturationProfileCatalogue.ts` (the roster's
  self-warming boot home; platform pack row + `boot:` entry)
- `platform/thing/Vat.ts` — the concrete; seal/move window events
- `platform/idea/api/BulkableLogic.ts` — `carryBatchIdentity` +
  `tryInoculate` on the transfer
- `lib/behavior/cellars.ts` (+ `__tests__/`) — the producer beats
- `platform/thing/instrument/Hydrometer.ts` +
  `platform/idea/cmd/perception/MeasureDensityController.ts`
- Tests: `lib/maturation/__tests__/` (the transform, yeast),
  `world/__tests__/fermentation-{brewing,distilling,cider}.test.ts`
  (real rows), `lib/bulk/__tests__/GradeCarry.test.ts` (the seam)

Cross-references: [husbandry](./husbandry.md) (the shape) ·
[thermal](./thermal.md) · [bulk](./bulk.md) · [crafting](./crafting.md)
· [fire](./fire.md) · [content-packs](./content-packs.md) ·
[behavior](./behavior.md) · [advancement](./advancement.md).

---

## ⭐ `evaporative` — the fourth mechanism (extraction W1)

The closed mechanism vocabulary gained a fourth member beside `microbial`,
`photochemical` and `chemical`:

> **`evaporative`** — ⭐ **the air does it, by taking water away.** A salt pan,
> a brine hearth, a drying green for a solution rather than a solid.

Two things distinguish it from the other three, and both are load-bearing:

- **the batch gets SMALLER as it converts** (the water leaves), which is what
  `productFraction` is for — the fraction of the original volume the finished
  good occupies; and
- ⭐⭐ **rain puts it back.** It is the only mechanism with a **setback that is
  not a failure**: `reconcileEvaporativeWindow` walks the window segment by
  segment, the rate is the air's measured against `BRINE_EQUILIBRIUM_RH_PCT`,
  and rain adds litres through a derived aperture so `f` goes *backwards*. A
  pan left out in a wet week is not spoiled; it is behind.

⚠⚠ **It authors NO strain, and the absence is deliberate** — see the
`requiresFlora` finding below.

### ⚠⚠ The strain gate froze two shipped features

`requiresFlora(profile)` is new, and it exists because a flora-less mechanism
turned a latent bug into a rule. The strain gate arrived with the fermentation
build and asks *has it been pitched?* — and **`retting.yaml` and
`bleaching.yaml` author no strain at all**, so `strainOk` was false forever and
**a retting pit and a bleaching green never converted**, while the room read
*"It sits sweet and silent."*

The rule is now: **a profile demands flora only when it names a required strain
or declares it catches one.** Two related gates moved with it — the augmenter's
*sweet and silent* line fires on the same predicate as the conversion, and
`reconcileCellarAir` is gated to `microbial` (an unguarded bleaching green was
draining its room's air reserve toward unbreathable).

⭐ The lesson generalizes past maturation: **a gate added for one mechanism
silently freezes every mechanism that does not answer it.**
