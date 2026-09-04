# Fermentation — the durative transform

**The liquid leg's kernel** (fermentation build, 2026-09-01): a batch
converts over game-time under conditions. *Growth accretes,
fermentation converts* — the husbandry SHAPE (reconcile lazily on
read, staged, no far-past guard, no linkdead freeze) with a different
equation: the mass is already present and what changes is what it IS.

Kernel: `lib/ferment/` (`Fermenting.ts`, `FermentProfile.ts`),
`platform/idea/ferment/FermentProfile` (the instanceable concrete —
the Material split), `platform/thing/Vat` (the one concrete every
trade's vessel rows name), `platform/idea/FermentProfileCatalogue`
(the self-warming roster — **no FermentApi**: the queries are statics
on the owning class, and an operator-shaped warm does not belong on a
consumer Api). Zero new verbs: crush/mash/distil are
recipes `order`ed off unpriced work boards (the kitchen-menu shape);
rack = `pour`, seal = `close`, bottle = `fill`, pitch/feed = `pour`.

## The transform rides the VESSEL (D2)

`FermentingMixin` composes on a Bulkable host (composition-validated)
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

`FermentProfile` rows (any root's `idea/ferment/` subtree; matched by
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
seam). **A new drink is rows alone** — proven by the cider test
(`world/__tests__/fermentation-cider.test.ts`).

`FermentProfileCatalogue.postRegister` stands the roster up, eager via
the platform pack's `boot:` manifest (`sync-read`) — the
reference-Ideas-inert-at-boot rule, self-warming (the
Discipline/Recipe catalogue shape rather than an `Api.boot()`
sequencer line). `FermentProfile.forMaterial`/`.byKey`/`.all` read the
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
  Thing, and `FermentingMixin` requires a Bulkable interior — months in
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

- `lib/ferment/Fermenting.ts` — the mixin: reconcile, batch detection,
  strain/culture/lees, the cellar air, the sensory augmenter
- `lib/ferment/FermentProfile.ts` (+ the query statics) →
  `platform/idea/ferment/FermentProfile`
- `platform/idea/FermentProfileCatalogue.ts` (the roster's
  self-warming boot home; platform pack row + `boot:` entry)
- `platform/thing/Vat.ts` — the concrete; seal/move window events
- `platform/idea/api/BulkableLogic.ts` — `carryBatchIdentity` +
  `tryInoculate` on the transfer
- `lib/behavior/cellars.ts` (+ `__tests__/`) — the producer beats
- `platform/thing/instrument/Hydrometer.ts` +
  `platform/idea/cmd/perception/MeasureDensityController.ts`
- Tests: `lib/ferment/__tests__/` (the transform, yeast),
  `world/__tests__/fermentation-{brewing,distilling,cider}.test.ts`
  (real rows), `lib/bulk/__tests__/GradeCarry.test.ts` (the seam)

Cross-references: [husbandry](./husbandry.md) (the shape) ·
[thermal](./thermal.md) · [bulk](./bulk.md) · [crafting](./crafting.md)
· [fire](./fire.md) · [content-packs](./content-packs.md) ·
[behavior](./behavior.md) · [advancement](./advancement.md).
