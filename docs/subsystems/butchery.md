# Butchery — taking an animal apart, and what the pot does next

**The subject: a carcass becomes named joints, and the joints know what
they want.** This doc owns the share model that makes part mass
scale-free, muscles as materials, cuts that claim them, the two forms of
`butcher`, the tool and skill gates, the cooking law, and the nose-to-tail
products.

⭐⭐⭐ **One derivable law runs the whole chain.** A muscle that works
carries connective tissue; collagen gelatinizes only under long, moist
heat. So a shoulder braises and a loin sears, and getting it the wrong way
round costs you. That is real food science and it is **predictable without
a table** — a player who knows a leg works harder than a loin can say
which wants the pot and be right, which is the Andy Weir property
`design-lenses.md` lens 1 asks for.

⚠⚠ **It was missing for a build, and the tree said so the whole time.**
The carcass chain shipped the plumbing and skipped butchery: every land
animal yielded twelve units of one generic `stew-meat`, so a cow, a ewe
and a sow differed only in kilograms-per-unit. Meanwhile
`Discipline/butchery.yaml` had always described the skill as *"where the
joints are, **which cut is which**, and how to open a carcass without
opening its gut"* — two of those three had shipped.

## The share model

A `TissueComposition` on a `BodyPlan` states the **share of the whole
body's mass** a tissue of a part carries, `(0, 1]`. Part mass is
`share × the instance's own mass`.

⭐⭐ **A share, because ONE plan serves every size of animal.**
`quadruped` is named by sheep, cattle, dogs, cats and horses; `avian` by a
canary and a hen. ⚠ Absolute kilograms made that a lie and nothing caught
it: the quadruped plan authored a 28 kg torso, so **a bullock claimed the
same torso as a ewe**, and `avian`'s canary-sized 4 g of torso bone is why
nothing else could reuse it. The masses fed only `partArea`, whose readers
are ratio-only, so the fiction was invisible until butchery needed to know
what a cut weighed.

- **The invariant:** a plan's shares sum to **1**. `lint:anatomy`
  clause (a) holds it over shipped rows. ⚠ An EMPTY plan is exempt
  (`sessile` is a plant with nothing to weigh); a PARTIAL one is not.
- `setBodyParts` validates each share in `(0,1]` and **refuses a `mass`
  key by name** — the `governs`-typo precedent, because a row left on
  kilograms would read as a share of 12, silently enormous.
- `partArea` is Meeh's law over shares, `(Σ share)^(2/3)`. Both readers
  are ratio-only, so every shipped answer is unchanged.
- ⚠ **Whole-body mass is untouched.** `Creature.bodyMassIndex()` is
  `getMass() / stature²` and `getMass()` is the species frame plus the
  flesh/lean reserve deltas. Fitness and BMI never read part masses.
- **A species whose proportions differ gets its own plan**, by `extends:`.
  `BodyPlan/swine` extends `quadruped` — same topology, a pig has the same
  muscles in the same places — and restates the shares: 28 % separable fat
  against a sheep's 6, and twice the belly. ⚠ A `Species.tissueShares`
  override field was built, tested and **taken back out twice** by
  `lint:unconsumed-seams`; a row saying *a pig's body is a different body*
  turned out truer than scaling a sheep's at read time.

## Muscles are materials

`MuscleMixin` (`lib/butchery/Muscle.ts`) + the twin
`platform/idea/material/Muscle` — the `RadioactiveMaterial` pattern, one
field: **`work`**, `0..1`, *how hard this muscle worked in life*.

The shipped quadruped ladder: `shank .95 · neck .85 · shoulder .80 ·
leg .60 · belly .50 · rib .35 · loin .25 · tenderloin .05`.

- ⚠ **Not a field on `Material`** — that would claim granite worked, and
  every reader would guard on a `muscle` tag (the wrong-host tell).
- ⚠ **Not a reuse of `toughness`**, which is the blunt channel's
  MJ·m⁻³: reusing it would make a shank harder to **bruise** than a loin
  in a fight.
- Intramuscular fat is **`composition`** naming `animal-fat` — an existing
  field with existing readers, so a belly reads fattier than a loin on the
  nutrition label for free.
- ⚠ `work` defaults to `0`, the **tenderest** end rather than a plausible
  middle: an unauthored muscle is noticed the first time a cook braises
  it.

⭐⭐ **White meat is white by the same law, expressed structurally.** A
chicken's breast is the flight muscle of a bird that does not fly: pale,
almost no myoglobin, almost no connective tissue. So `avian` (which flies)
names `breast-red` and **`fowl`** (which does not) names `breast-white`,
and nothing special-cases poultry. A duck or goose later is `avian` plus a
species row, which is why the two plans are kept apart. ⚠ A fowl's wings
deliberately do **not** `serve: [locomotion]` — a hen's wings get it onto
a perch, so a broken wing must not cost it a quarter of its mobility the
way it costs a canary.

⚠⚠ **And the hen named `biped` — the HUMAN plan — until 2026-10-06.** A
chicken had `body.arm.left.hand`, a liver, an upper and lower spine, and
no wings. Nothing errored, because nothing checked.

## A cut claims muscles

`CutMixin` + `platform/thing/Cut` (on `Provision`, inheriting Freshness,
WaterActivity, Contaminable, Crafted).

- `tissues: string[]` — the muscles it takes. ⭐ **A porterhouse is two
  entries**, and nothing special-cases it.
- `cutting: boneless | bone-in | chop` — the depth the tools must reach.
- `difficulty` — the hand it takes to get the joint rather than trim.
- `_speciesPath`, `damaged` — stamped at the mint, never authored.
- **Toughness derives**: the share-weighted mean of its muscles' `work`,
  derive-on-read. ⚠ `null` when nothing claimed is a muscle, because a
  bone heap has no texture and answering `0` would make it the most
  tender thing in the game. **Nobody can author a tender shank.**
- **Mass derives**: `Σ(claimed shares) × the carcass's own mass`, so a
  yield line with a claim authors **no `fraction`** and the same shoulder
  row off a ewe and an ox weighs what each animal's shoulder weighs.
  `lint:anatomy` clause (c) refuses a line that authors both.

## The two forms, and the three gates

`butcher <body>` takes every line the tools allow;
`butcher <body> for <cut>` takes one, named by a **word** matched against
the cut row's keywords.

- **The tool is the depth.** A knife: boneless cuts, offal, hide, fat. A
  **saw**: bone-in joints. A **cleaver and a block**: chops — you cannot
  chop through a backbone against nothing. ⭐ A line the tools cannot
  reach **stays on the carcass** and the sentence names the tool it wants:
  the refusal is the progression UI.
- **The hand decides joint or trim.** A good butcher lifts a whole loin
  where a poor one leaves trim, which puts skill in the **output** where a
  player can see it rather than in a mass multiplier nobody can read.
- **The gut spills by band** (`spillGut`), which is the contamination
  route — and ⚠ **gut itself is contaminated at full severity whatever the
  hand**, because the gut is where the contamination lives.

⭐⭐⭐ **The carcass REDUCES.** `Corpse.takenTissues` / `takenLines` mean
what you took is gone and what you left is still there, so a side can be
worked **to order** — lift the loin to sell and come back for the rest.
The body is destructed only when every line is off it. That gives partial
breakdown and trimming with **no second verb**, and makes the lens-4
choice real: prime to the inn, or keep it. ⚠ Not durable: `Corpse`
composes no `PersistableMixin`, and a body on a hook mid-breakdown is not
something a reboot owes anybody.

⭐⭐ **A wound damages the cut it came from**, off the Trauma slice the
corpse already adopts: wounds site on anatomy parts and a muscle names the
part it sits in, so the join is a lookup and not new plumbing.

## The cooking law

`applyMethodFit`, at the one place a build's grade is derived.

- The method is read off fields a recipe **already had**: `medium: water`
  plus a hold of two game hours is `long-moist`, anything else is
  `fast-dry` (`CookingAttempt`). That vocabulary shipped to model a phase
  ceiling, and it describes the method exactly — **no recipe schema
  change**.
- ⭐⭐ It reads the **material**, not the cut object, which is the engine's
  own law (`response = f(mechanism, material, construction)`) and covers
  both routes: the one-shot craft and the by-hand build, whose
  contributions are snapshots carrying only a `materialPath`. ⚠ Reading
  the cut object would have left the by-hand route as a laundering path
  where a stewed loin came out ungraded because it went through a pot.
- **One band of grade**, either way. Not a refusal and not a destroyed
  dish: a stewed loin is still dinner, just a worse one.
- ⭐ The fit is **asymmetric** because the mistakes are not: a tough cut
  cooked fast is **inedible** (all that collagen, none of it dissolved)
  where a tender cut braised is merely **wasted**. A cook learns the worse
  lesson first.
- ⚠ `hearty-stew` had to start **authoring** `holdS: 7200`: it stated no
  hold, so it rode the universe default, which reads as a quick simmer. **A
  braise has to say it is a braise.**

## Nose to tail

⭐⭐⭐ **The chain closes rather than extending.** A sausage takes the
**trim** a good hand's offcuts leave, the **fat** nobody wants on a plate
and a **casing** scraped out of the gut — three things that were waste a
build ago.

- `scrape-casing` → `sausage`; `black-pudding` from blood, fat, grain and
  a casing. ⚠ Scraping makes gut **usable, never safe**: the loads ride
  straight through.
- ⚠⚠ The sausage deliberately does **not** take `offal`, so the bakery's
  dog loaf keeps its cheapest line. The two goods want different things
  off the same animal.
- ⚠ **Blood SPILLS** when nobody brought a vessel — no refusal, no
  warning, because blood on the floor is what happens. A pudding is
  something you only get if you came prepared. Deliberately unlike the
  milk rule: milk is a tap you can return to, and a carcass is once.
- **Lard is not suet.** `leaf-fat` is tagged `cooking-fat` and **not**
  `tallow`, so the chandler's dip refuses a pig's fat by tag arithmetic
  with nothing anywhere checking for a pig.
- The general store prices a **loin at 12 against trim at 3** — the same
  animal is worth four times as much cut well, which is why a butcher is
  worth paying.

## Fish

**Already done, and the precedent the land chain should have copied.**
`trade-fishing`'s species have authored real per-species cuts (`fillet`,
`roe`) since the fishing build, and `food/fish-flesh` carries
`spoilActivationEnergy: 60000` against flesh's `80000`, so a fillet goes
off sooner with nothing special-cased.

⭐⭐ **The law covers fish with no fish branch**: nearly all white
fast-twitch muscle in myomeres, almost no connective tissue, collagen that
gelatinizes far lower — which is *why* a fillet cooks in minutes and flakes
where a shoulder braises for hours. Fish are the **tender end** of the same
scale the ox's worn shoulder anchors.

⚠ **Cleaning (gutting and scaling) is deliberately not an act.** Prompt
gutting really does slow spoilage, but a second verb earns nothing a player
would care about, and a fish needs no tool beyond a knife.

## The fidelity rule this subsystem is built on

⭐⭐⭐ **Consequence lives at the limb; tissue detail must earn its way by
changing an outcome somebody cares about.** A bullet sites on a muscle —
cheap, since the wound seam already reads tissues — and the consequence
aggregates to the **limb** through `serves`'s mean. One limp, no menu of
limps. Nothing prohibits modelling more; nobody would care which limp they
have, and getting them to care about a limp at all is a different
exercise (→ `recovery-slate`).

⚠ So there is **no per-muscle exertion consumer**, deliberately. Inventing
one would manufacture a declared capability with no reader — a failure
this build met three times in one day (`getDecayStage`, `partArea`'s
callers, `tissueShares`).

## Gates

`lint:anatomy` (`scripts/check-anatomy.ts`), four clauses, all over
shipped rows:

- **(a)** every body plan's shares sum to `1 ± 1e-3`.
- **(b)** every species that yields anything names a body plan that
  **resolves** — a species naming a missing plan has no anatomy, so every
  claiming cut derives zero and the animal butchers to nothing.
- **(c)** no yield line authors a `fraction` beside a claiming cut — two
  sources for one number, where the derivation silently wins.
- **(d/e)** every claimed tissue resolves to a Material row and is carried
  by the plan of every species whose yield claims it.

⚠⚠ **Not a ratchet.** The sum is an invariant: there is no honest census
of bodies that are 40 % nothing, so there is no ceiling to lower.

`lint:census` reads a cut's `tissues` as identity refs — a misspelt muscle
would otherwise be **a cut with no texture and no weight**, minted happily
with nothing saying a word.

## Cross-references

- [vitals.md](./vitals.md) — anatomy, the share bullet, the Agent/Creature
  split
- [mortality.md](./mortality.md) — the corpse, and why it is a `Thing`
- [race.md](./race.md) — Material substrate and Species
- [crafting.md](./crafting.md) — recipes, Grade, `applyMethodFit`
- [spoilage.md](./spoilage.md) — the clock a cut runs on
- [harm.md](./harm.md) — the wound that damages a cut
- [materials-response.md](./materials-response.md) —
  `response = f(mechanism, material, construction)`
- [ranching.md](./ranching.md) · [taps.md](./taps.md) — upstream

---

## History — the pre-merge sweep (2026-10-06)

Three things the sweep's drive found, recorded because each failed closed
and **silent**, and two of them had been red on the branch with nothing
able to say so.

⭐⭐⭐ **The one working tannery's pit was DRY.** `tanpit.yaml` authored
`barkKg: 100` and no liquor, so `liquorLitres()` read an empty bulk slot,
`tanLiquorFor` returned `null`, and **every `tan` in the realm refused
`pit-dry`** — *"the pit is empty, bare boards and a smell that has soaked
into them."* The solute without the solvent: a hundred kilos of bark in
no water is not a charge, and the row's own comment said *"ships
CHARGED"*. Nothing threw; the pit described itself beautifully; the
refusal was a true sentence about a world nobody meant to author. ⚠ A
warm database that had been hand-filled once hid it, which is why it took
a freshly dropped one. Fixed by authoring `interiorMaterial` +
`interiorAmount` beside the bark, and pinned by a row assertion that
checks the two numbers satisfy `BARK_KG_PER_LITRE × litres` **together**
rather than either alone.

⚠⚠ **The carcass drive asserted the opposite of what this build decided.**
Its checkpoint 7 read *"and the carcass is gone — one body, taken apart
once"*, which was true when one `butcher` destructed the animal and
minted five goods. This build **deliberately reversed that** — a carcass
reduces a cut at a time and persists until spent, which is the whole of
AC6 and what lets a knife-only butcher come back with a saw. ⭐ The drive
was never re-run after that landed, so the file went red on the branch
and nothing said so. It asserts the new truth now.

⚠ **A cut comes off onto the GROUND, not into your hands**
(`ContainmentApi.move(cut, here)`), which is right — a carcass is on a
block and what comes off it lands there — but the drive had walked to the
tannery empty-handed and `tan hide` then bound the word to whatever else
was in reach, surfacing as a `TanningMixin needs TangibleMixin` refusal
three checkpoints downstream. A missing `get`, not a composition defect.
⭐ Worth keeping as a reading lesson: a mixin-composition refusal is what
an arg gate says when the *target* is wrong, so it points at the binding
rather than at the class.

⚠ **`analyze sky` stopped answering at all**, at the tannery after a
25-game-day jump: no dispatch-response in 30 s, which poisoned the
session and timed out the checkpoint a suite later. The carcass drive's
`ensureDaylight` used it; the butchery drive had already replaced the
same helper with one that asks the **room** — *what the file needs is to
be able to SEE, not to know the hour* — so that version is now in both.
⚠⚠ The instrument's own defect is NOT this build's: its known failure
mode is `details.keys is not a function`, because `DetailedMixin.details`
is a persistent Map and one restored host in reach breaks arg resolution
for every `analyze` in the room. ⭐ Worth noting *why the sweep found it
and nine build-time runs did not*: the dry pit had been blocking the
tanning leg, so the drive had never reached this far in a working state.
**Fixing one defect is what buys the next one.**

⭐⭐ **And the two drive files collide in one world.** Both draft from the
same persisted flock into the same yard, and the wire suite boots one
world for every file. Recorded with its three fixes in
butchery-slate; it cannot be patched
probe by probe, because with two ewes in the yard every noun the files
share is ambiguous.
