# Arcane science — thaumology

**Status: lore + design constraint.** This document authors the
*science* of magic: the postulate, the laws, the units, the taxonomy,
the history, and the open problems. It is a peer of
story-bible.md (which holds the cosmology and the
mythic register) and a constraint on
[docs/subsystems/magic.md](./subsystems/magic.md) (which holds the
shipped mechanism). It changes no mechanic. It explains the ones we
have, and it tells content authors what they may and may not author
next.

It exists because the University has to teach something, and because
the fastest way to demonstrate that this world can carry a curriculum
is to build a subject from scratch and teach it honestly.

> **Audit log.** 2026-08-02 — every derived figure recomputed
> end-to-end, and every claim about real science history checked
> against sources. Two substantive errors found and corrected: the
> cooling model implied a second exemption (fixed — cooling is a heat
> pump, priced by Carnot), and momentum conservation was missing
> entirely (added — casts recoil). One historical claim ran backwards
> and was rewritten (see the Crowe dissociation). **Re-run this audit
> against any new numbers or historical claims before they land** —
> both errors were in the most confident-sounding passages, and the
> rule at the head of this document is only worth something if things
> occasionally fail it.
>
> **Revision, 2026-08-11 — the mana/energy split.** Two errors of the
> same kind, both in confident passages, both found by *using* the
> science rather than reading it. **(1)** Halloway Equivalence read a
> **coupling constant** as an **identity** (`a mana point IS a
> kilojoule`) — the same over-reading of Joule that real caloric theory
> made, and it is now stated as the exchange rate it always was.
> **(2)** The postulate was narrowed to *energy* when the exemption it
> actually makes is *locality*, which had the effect of appearing to
> forbid teleportation — while the TPA has been shipped content the
> whole time. ⚠ **No numbers changed** (k = 1 kJ/τ), and no second
> exemption was added. Design record:
> mana-economy-design-pack.

---

## The hard rule

> **Nothing in this science may contradict real science.**

The bar is *higher* for magic than for any other subsystem, not
lower, precisely because magic is invented. A student who learns
metallurgy here learns real metallurgy; a student who learns
thaumology here must learn real thermodynamics, real dimensional
analysis, real experimental method, and real scientific literacy —
carried by a fictional subject. If any part of this document teaches
a student something they would have to unlearn in a real classroom,
that part is a defect.

So the invented content is deliberately confined to **one postulate,
one quantity, and one taxonomy.** Everything a student would actually
be graded on is real.

| Invented | Real |
|---|---|
| The postulate (below) | Conservation of energy and momentum |
| **Mana as a second conserved quantity** | **Conserved quantities and their coupling constants** (charge; the mechanical equivalent of heat) |
| The verb × noun taxonomy | Thermodynamics, all of it |
| The names of the laws and their discoverers | Specific heat, latent heat, phase change |
| Which quantities happen to be measurable | Efficiency, exergy, Carnot limits |
| The historical narrative of the field | Dimensional analysis, unit conversion |
| | Exponential decay, curve fitting, error bars |
| | Landauer's principle |
| | Experimental design, controls, replication |
| | Reading a paper: claim, evidence, range, gap |

⚠ **The budget is now three items, and that is the ceiling.** A second
conserved quantity is the one addition since this document was first
written (2026-08-11). *Mana is not energy* — it exchanges with energy
at a fixed one-way rate, exactly as charge does through a potential,
and a student who understands that structure has learned something
true about charge and chemical potential too. **Anything that appears
to need a fourth invented item is a modelling error.**

---

## The method

The discipline is borrowed from Andy Weir, whose whole technique is
restraint:

1. **One impossible thing.** State it plainly, early, in a sentence.
   Never expand it.
2. **Everything downstream is real,** and computed honestly.
3. **Trust the reader with numbers and units.** Show the work.
4. **Constraints are the plot** — budgets, rates, deadlines.
5. **Being wrong is instructive.** Miscalculate and reality bills you.
6. **The impossible thing never solves a later problem.**

Rule 6 is the governing design rule for this project, and it is the
one fantasy always breaks. **Magic is the problem generator. Real
reasoning is the solver.** The moment magic gets a character out of a
jam that magic put them in, the course is dead and so is the game.

Rule 3 has an enforcement mechanism, and it is the most important
sentence in this document for anyone authoring content:

> **Every claim in this science is dimensionally checkable, because
> the thaum has a real dimension. A fudge shows up as a dimensional
> error.** If a proposed spell, effect, or law cannot be dimensionally
> analyzed, it does not ship.

---

## The Postulate

> **Locality fails for a caster acting deliberately. They may relocate
> energy, or themselves, between their own body and one chosen point,
> in either direction, in a form of their choosing, without a medium
> in between.**

That is the entire fictional physics. Stated precisely: **local
conservation fails; global conservation holds.** What vanishes at one
place appears at another, and the books balance.

**What it breaks:** locality, and *only* locality. The continuity
equation — the statement that a conserved thing cannot leave a region
without crossing its boundary — is false in this world, and it is
false only for a caster acting deliberately.

> ⚠ **Revised 2026-08-11 — the exemption is LOCALITY, not "energy
> transport."** This postulate was previously written as *"a caster can
> relocate **energy**…"* That narrowing was never load-bearing, and it
> had one bad consequence: it appeared to forbid **teleportation**,
> which the fiction has shipped all along (the TPA). Moving oneself is
> a locality violation and nothing else — **the same exemption applied
> consistently, not a second one.** See *Control·Body and the terminal
> network*, below.

**What it does not break:** everything else. The first law holds. The
second law holds — *this one is load-bearing and easy to lose*:
relocating energy is not a licence to move heat up a temperature
gradient for free, so a caster who cools something is running a heat
pump and pays a heat pump's price (below). **Momentum is conserved
too** — the channel carries momentum along with energy, so a caster
who pushes gets pushed. Chemistry, materials, biology, and every
downstream consequence are untouched.

The discipline to hold here: *every* time magic appears to need a
second exemption, that is a modelling error, not a discovery. The
postulate breaks locality. It has never needed to break anything
else, and the moment it seems to, the model is wrong.

This is better pedagogy than breaking nothing. You teach local
conservation properly, then show students exactly what its failure
would look like — and, more importantly, exactly what its failure
would *not* excuse. A student who can say "that's not covered by the
postulate" has understood conservation better than one who was never
made to test its edges.

### Four consequences worth naming immediately

**The caster is always one endpoint.** The postulate does not grant
transfer between two arbitrary points. One end is always the caster's
own body. This single clause is why the faculty is anatomical
(`CasterMixin` on a body, banded by species), why magic is limited by
a person's physiology, why cold magic cooks the caster, and why a
mage standing in a conductive pool is part of their own circuit.

⭐ **And it is why you can only teleport *yourself*.** A caster cannot
relocate a third party, because the third party is not an endpoint.
This closes an obvious exploit without anyone needing to rule on it —
nobody teleports a prisoner into a cell — and it is why a TPA terminal
is a booth you step *into* rather than a service performed on you.

**Perception at a distance is an active probe.** The postulate grants
transfer, not omniscience. Remote sensing works the way all remote
sensing works — you send a small amount of energy out and read what
comes back, the same nonlocal channel run in reverse. Which means
**scrying is detectable**: a probe is a signal, and signals can be
noticed by anyone with an instrument tuned for them.

**Magic never creates matter.** One litre of water is 9 × 10¹⁶ J by
E = mc². The postulate relocates; it does not manufacture mass. Every
"conjuration" in the roster is therefore a **collection**, and has to
be modelled as one.

⚠ **Do not over-read this into a ban on moving matter** — it is an
*affordability* argument about **creation**, and it is silent on
**transport**. Making a litre of water costs 9 × 10¹⁶ J and is
therefore out forever. Moving a litre of water that already exists
costs whatever its change in potential energy costs, which is usually
very little. Conjuration is collection; teleportation is relocation;
neither is creation.

**Every push is a shove in both directions.** Momentum rides the
channel, so Newton's third law is not suspended: give a 70 kg target
200 J of kinetic energy and it leaves at 2.4 m/s — and an unbraced
70 kg caster departs at 2.4 m/s the other way, carrying **its own
200 J.** A symmetric shove costs *double*, because you pay for both
ends of it.

Which is why a practised caster **braces** — plants, and lets friction
carry the recoil into the ground — exactly as a rifle is braced
against a shoulder. Bracing is not flavour text; it is the difference
between spending 200 τ and spending 400 τ, and between shoving
someone and shoving yourself. It also means **Control·Body is one of
the most expensive things in the roster**, which is the correct
result: moving mass around is genuinely costly, and a mage is not a
crane.

> ⚠ **That expense is about ACCELERATING mass, not relocating it.** The
> shove is dear because it buys kinetic energy and pays the recoil
> twice. A **teleport** accelerates nothing: no kinetic term, no
> momentum transfer, no recoil. Its only energetic cost is the change
> in potential energy, `mgh` — which is why arriving uphill costs more
> than arriving downhill, why a loaded traveller costs more than an
> empty one, and why **distance costs nothing at all**, locality being
> the thing that broke. What teleportation *is* expensive in is
> **specification**; see *The specification problem*.

### The second quantity — mana is not energy

The postulate is the only *impossible* thing here. Alongside it sits
one invented *quantity*, and the distinction matters:

> **Mana is a conserved quantity in its own right, which exchanges
> with energy at a fixed rate `k` in one direction only.**

`E_delivered = η · k · M_spent`, with **k = 1 kJ/τ**.

This is the relationship **electric charge** has with energy: charge is
conserved independently, couples to energy through a potential
(`E = qV`), can be stored in matter, and none of that makes energy
conservation false. Four consequences, all of which the field took an
embarrassingly long time to notice (see *Halloway Equivalence*):

- **The conversion is one-way.** Mana becomes energy; no amount of fuel
  becomes mana. There is no such thing as a mana generator.
- **Mana can be held by matter.** A substance has a *mana density* the
  way it has a specific heat, and a *mana conductivity* the way it has
  an electrical one. Both are ordinary material properties.
- ⚠ **Mana itself moves by contact, not nonlocally.** The postulate
  relocates energy and the caster; it says nothing about relocating
  mana, and extending it there would be the second exemption this
  document does not permit. Mana conducts, like charge — which is why a
  caster must *hold* a conduit to charge it, and cannot draw on a
  distant reserve.
- **Living things accumulate it slowly from their surroundings**, which
  is where a caster's reserve comes from.

⚠ **Note the two clauses point opposite ways and are easy to confuse.**
*Locality* is general: energy and the caster's own body both cross it.
*Mana transport* is the narrow case: contact only. Getting this
backwards produces a mage who can drain a vault from the street.

---

## Units and quantities

| Quantity | Symbol | Unit | What it is |
|---|---|---|---|
| Assertion / arcane charge | M | **thaum (τ)**, exchanging at k = 1 kJ/τ | mana committed to a transfer |
| Reserve capacity | D | τ | the caster's usable arcane budget ("Depth") |
| Recovery power | S | τ·min⁻¹ (≡ W) | rate of budget replenishment ("Serenity") |
| Delivery efficiency | η | dimensionless, 0 < η ≤ 1 | delivered ÷ committed |
| Leak rate | λ | τ·s⁻¹ | rate a bound state bleeds back toward equilibrium |
| Composure | C | dimensionless | resistance to being patterned by another |

**A thaum buys a kilojoule.** That is the whole of Halloway
Equivalence (below), and it is the reason this science can be graded.
The engine's mana `pt` and the thaum are the same number; naming it
costs nothing and buys dimensional analysis.

⚠ **A thaum is not *itself* a kilojoule** — it buys one, at the
coupling constant `k = 1 kJ/τ`, in one direction. The distinction is
worth the care: a calorie is not a joule either, and the field's
founding error (below) was exactly this conflation. **Every number in
this document is unaffected**, because `k = 1` numerically; what
changes is what the unit *means* and what may hold it.

*Calibration note.* World time runs at 12× real time. Rates quoted in
this document are **physical** (per real second). Any dial expressed
per game-minute must be divided through by the scale factor before it
is compared to a physical rate; getting this wrong is the most likely
source of a silently dishonest number in the whole system.

---

## The caster's budget

⚠ **Revised 2026-08-11.** The reserve was previously described as a
fast-access fraction of the body's glycogen. It cannot be: the coupling
is one-way, so **food does not become mana.** A caster's reserve is
**accumulated slowly from ambient mana**, the way any living thing
accumulates it — which means **recovery rate depends on where you are.**
Terminus, whose Confluence vents into the air, is a rich place to
recover; a sealed vault is a poor one.

For scale, a human body carries roughly **8.5 MJ** of glycogen (about
500 g at 17 kJ/g), and the arcane reserve is comparable to a small
fraction of it — a useful comparison even though the two are no longer
the same stock.

These are the **shipped** values (`lib/magic/Faculty.ts`,
`FACULTY_DEFAULTS`), read in kilojoules. Rates are quoted per
game-minute *and* per real minute, because world time runs at 12×
and the physics lives in real seconds:

| Band | Depth | Serenity | Recovery power | Time to full |
|---|---|---|---|---|
| low | 80 kJ | 0.9 τ/game-min | **180 W** | 89 game-min (7.4 real) |
| mid | 120 kJ | 1.5 τ/game-min | **300 W** | 80 game-min (6.7 real) |
| high | 180 kJ | 2.4 τ/game-min | **480 W** | 75 game-min (6.2 real) |

Three things a student should notice, because all three are real and
all three are commonly gotten wrong:

- **Capacity and rate are independent axes.** They are given
  independent bands on purpose. A large tank does not refill faster;
  a fast metabolism does not hold more. Confusing a stock with a flow
  is the most common novice error in this field and in every other
  field that has both.
- **A full arcane reserve buys about a quarter of a banana** — 120 kJ
  of delivered energy against a banana's 440. That is the correct order
  of magnitude and it should be taught, not hidden. A caster is,
  energetically, a snack.
- ⚠ **CASTING is exercise.** Compare the recovery column against the
  benchmarks: basal metabolism is ~97 W, a brisk walk is 300–400 W,
  hard cycling is ~1000 W. Pushing mana through the coupling costs the
  *caster* real joules — the losses go where losses go — so a mid
  caster working steadily is spending **about the metabolic cost of
  walking**, and a high-band caster rather more.

That second one is a derived prediction, not flavour, and it is
testable against the shipped metabolism subsystem: **working magic
should make you hungry and warm**, on the same order as a brisk walk of
the same duration.

> ⚠ **Revised 2026-08-11 — this used to read "recovering is exercise,"
> and the metabolic cost has moved.** It attaches to **spending** mana
> (the coupling is lossy and the caster is the resistor), not to
> **refilling** it (which is ambient absorption and costs the body
> nothing). So the old advice — *"a caster who wants to refill fast
> should eat, not meditate"* — inverts: **eating does not refill a
> reserve; being somewhere rich does.** What eating fixes is the
> hunger that *casting* caused.

**Overchannel** is metabolic overdraft. Completing a transfer past
empty draws on tissue the body needed, which is why it hurts, why the
penalty is a real condition, and why it clears slowly and by
hysteresis rather than at a threshold. Anyone who has bonked on a long
ride has done the experiment.

### The power level, and why it is canon

A few hundred kilojoules per full reserve is, at the top end, about
one hand grenade per day. A mage cannot level a city. This is not a
balance decision; it is the postulate plus human physiology.

It also proves a piece of existing canon. The bible says the realm
*networked its minds but never mechanized its matter*. **There are
three reasons, and they are independent — which is why the result is
robust rather than a balance decision.**

**1. You cannot manufacture the input.** Mana is found, never made: the
coupling runs one way, so no quantity of coal, water or labour produces
a single thaum. Every other industrial input can be scaled by building
another plant.

> ⭐ **You cannot build a mana plant. You can only own a mana deposit.**

**2. Magic is bad at everything that must be *held*.** By Kell's
Partition an act is an impulse or a binding, and a binding must be
continuously topped up. A concrete dam is a capital cost after which
gravity works for nothing; a magically-held dam is an operating cost
that never ends. **Structure beats sustained assertion, always** — so
magic loses every contest it enters against a built thing.

**3. A person is a poor conduit.** A caster's throughput is bounded by
their own physiology, and unlike a furnace you cannot build a bigger
one.

⚠ **The old argument on this line was "a kilogram of coal is 30 MJ; a
caster is 0.3."** It was retired 2026-08-11: once mana is not energy,
coal and casters are not bidding for the same job, and a comparison of
their energy content answers a question nobody was asking. The three
reasons above survive that change, and reasons 1 and 2 are structural
— they hold at any power level, which the old one did not.

**What magic is left good at** falls out of reason 2 and is worth
stating positively, because it is where the whole practice actually
lives: **impulse, portability, and places you cannot build.** A cave, a
battlefield, a moment of need. Magic stayed artisanal for the same
reason handweaving did — not because it was weak, but because it was
never the cheap way to do anything at scale.

---

## The price list — the second law sets the cost

The postulate lets you choose the *form* energy takes on arrival. The
second law charges you for the choice. Not all joules are equally
useful — this is **exergy**, it is real engineering, and here it shows
up as a measurable price list.

| Delivered as | Typical η | Why |
|---|---|---|
| Heat | 0.85 | thermalisation is the cheapest thing energy can do |
| Kinetic / mechanical | 0.55 | ordered motion from a disordered store — **and see recoil** |
| Electrical potential | 0.40 | high-quality, highly ordered |
| Coherent / directed light | 0.30 | the most ordered form we can deliver |
| Heat *removal* | *not a fixed η* — a Carnot function of the lift | refrigeration: see below |
| Patterning (Mind, Sense) | — | energetically negligible; see the specification problem |

**Where the losses go.** η < 1 means some of what you spend does not
arrive — and it does not vanish, because the first law holds. It is
dissipated **in the caster**, who is the other endpoint. So *casting
warms you*: a firebolt spends 35.2 τ, delivers 29.9, and leaves 5.3 kJ
in your own body.

This is the honest account of a folk belief every apprentice is told —
that novices "run hot" and masters stay cool. They do, and it is not
temperament. It is η. A novice at 0.35 dissipates six times what a
master at 0.85 does for the same delivered effect, and cooks
accordingly.

**Heat removal is the interesting row, and it does not have an
efficiency.** Cooling is not "destroying heat" — heat cannot be
destroyed. Cooling is *pumping* heat out of the target, and by the
postulate the only place it can go is **into the caster.**

The postulate does not exempt the second law, so moving heat up a
gradient costs **work**, bounded by Carnot. There is no single
efficiency number, because the cost depends entirely on the
temperature lift:

`W ≥ Q · (T_hot − T_cold) / T_cold`

Three results follow, and all three are real refrigeration
engineering.

**Near ambient, cooling is *cheaper* than heating.** Pumping heat from
20 °C air into a 37 °C caster is a lift of 17 K: ideal COP ≈ 18, and a
realistic device at 40% of Carnot still manages ≈ 7. So moving 100 kJ
out of a room costs about **14 τ**, while *delivering* 100 kJ as heat
costs about **118 τ**. A heat pump moves more heat than the work it
consumes — which is genuinely counterintuitive, which is why students
get it wrong, and which is why it is worth teaching here.

**Deep cooling diverges.** Take the target to −20 °C and the realistic
COP falls from about 7 to about 2.2 — 3.4× the work per joule moved —
and it keeps falling as the target gets colder, without bound. **Frost
is cheap. Ice is dear. Absolute zero is unreachable at any price,**
which is the third law, arrived at by a student watching a cost curve
run away from them.

**And it all lands in you.** The caster absorbs `Q + W` — everything
pumped out *plus* everything spent pumping. So the binding constraint
on a cold spell is not the reserve at all:

> **Destroy·Fire is limited by thermoregulation, not by mana** — a
> completely different limit from every other cell in the grid.

That makes cold the cheapest thing in the roster to cast and one of
the most dangerous to cast twice. A mage can chill a room for pocket
change and then die of the heat they took on doing it.

*(No cold spell is in the shipped roster. This section is therefore a
**prediction**: if one is ever authored, the theory says it should be
cheap near ambient, ruinous at depth, and dangerous to the caster
rather than to the reserve. That is exactly the kind of claim a
science should make about an experiment nobody has run — and it is
falsifiable the moment someone authors the spell.)*

### Competence is efficiency, and efficiency is capped at 1

The engine scales potency by how far a caster's competence sits above
a spell's floor. The science says what that scaling *is*: **delivery
efficiency.** A master wastes less.

Which sets a hard ceiling nothing can argue with. η ≤ 1. A perfect
master delivers every committed joule and not one more. If a novice
runs at η = 0.35 and a master at η = 0.85, mastery is worth
**2.4×**, forever, and no amount of training, no artifact, and no
authored content can make it 100×.

**The second law is this game's anti-power-creep mechanism.** That is
worth more than any balance patch, because it cannot be lobbied.

---

## The laws

### Halloway Equivalence — the founding result

> The energy a cast delivers is proportional to the mana the caster
> spends, times a delivery efficiency that depends only on the form
> delivered and the caster's competence.

`E_delivered = η · k · M_spent`

Halloway's experiment is the field's paddle-wheel — Joule's, in the
1840s. Cast repeatedly into a stone basin of known mass and specific
heat, measure the temperature rise, compute the joules, and compare
against the reserve drawn down. It is what made thaumology a science
instead of a craft, because it made every subsequent claim
dimensionally checkable.

### ⚠ And the field over-read it for a century

The result was stated as *"a mana point **is** a kilojoule."* That is
**not what the experiment shows**, and the error is the same one the
rhyme should have warned against:

> **Joule did not show that heat *is* work.** He measured the
> *mechanical equivalent of heat* — a **conversion constant** between
> two different things. The calorie survived as a unit precisely
> because it measures something you measure differently.

What Halloway's basin measures is **`k`, the coupling constant.** The
field read a fixed exchange rate as an identity — which is caloric
theory's exact mistake, made for exactly the same reason: the numbers
come out the same either way.

⭐ **And here is why nobody caught it.** You cannot distinguish a
coupling constant from an identity while **every mana point you have
ever measured came out of a human body.** With one source, the two
hypotheses make identical predictions in every experiment anyone
thought to run. It takes mana that did *not* come from a caster — a
mana-bearing substance, measured against the same basin — to tell them
apart.

**The stone basin did not lie. It answered a narrower question than
the field asked of it,** which is the most common way a founding
result goes wrong, and worth more to a student than a century of
correct answers.

**The stone basin in the Practicum's casting yard is a calorimeter.**
It was put there as a safe target. It is, in fact, the instrument the
whole field was founded on.

### Kell's Partition — impulse and binding

> Every magical act is either an **impulse** — a delivery of energy,
> after which the world takes over — or a **binding** — a state held
> away from equilibrium, which must be continuously topped up.

An impulse sets an *initial condition*. A binding maintains a
*boundary condition*. Magic writes initial conditions and boundary
conditions. **It never writes laws.**

The consequences are the ones the engine already implements:

- A firebolt's fire, once lit, is **ordinary fire**. It has no author
  and nothing can un-light it. It burns until it runs out of fuel or
  oxygen like every other fire.
- A glowlight is held. Cut the assertion and it goes out.
- **Suppression severs assertion, not energy.** In a warded cell,
  bindings go dormant and a mundane brazier keeps burning. So does a
  magically-*lit* brazier, because after ignition there was nothing
  magical about it.

The warded cell is this law in one room, and it is already built.

#### ⭐ The economic corollary — once is cheap, held is dear

The partition has a consequence the field states about *physics* and
almost never about *cost*, and it decides where magic is used at all:

> **Magic writes initial and boundary conditions, never laws. So it is
> excellent at what happens ONCE and poor at what must be HELD.**

An impulse is paid for once and the world carries it afterwards, free.
A binding is a standing bill. Set them against built alternatives and
the verdict is the same every time: iron lock over held lock, lamp over
glowlight for permanent light, furnace over firebolt for sustained
heat. This is reason 2 of *why magic never industrialized*, above.

⭐ **It also tells an engineer which supply a device needs**, with no
judgement required — an **impulse** device draws *per use* and runs off
a stored charge, while a **binding** device draws *per second* and
needs a standing supply. A teleport terminal is an impulse device; a
ward is a binding one. Nothing else has to be decided.

### Voss Decay — why a binding costs

A common student error, imported from real physics where it is
correct: *holding a weight stationary does no work, so why does
levitation cost anything?*

It costs because **the state leaks.** A thermostat holding a room at
20 °C burns fuel continuously, not because "temperature" is work, but
because the room loses heat to outside. Same here.

`t_life = E_committed / λ`

where λ, the leak rate, is set by how hard the world's own dynamics
push the state back toward what it would have been anyway. Predictions:

- **Glowlight is cheap to hold.** Nothing in the world is actively
  trying to make a room dark; the cost is just the radiant power. At
  100 lm and η = 0.30, that is about 2.2 W — **8 τ per hour**, so an
  all-night lamp runs about 64 τ out of a mid reserve of 300.
- **Levitation is dear.** Gravity does not get bored.
- **A veil erodes fastest under attention.** The thing pushing back
  is the observers' own accumulating evidence, and the more of it
  there is, the faster you pay.

### The Reeve Line — composure

> A person's resistance to being patterned by another is affine in
> their own metabolic reserve, with a **non-zero floor.**

`C = C_band · (f + (1 − f) · m)`, where m is reserve fraction

This is shipped, exactly, today (`magic.composure.floorFactor`). The
lab is straightforward and runs on the current build: drain a subject
to 25%, 50%, 75%, stage the imposed condition, plot, fit the line.

Two things fall out, and the second one is why this law is taught in
the ethics unit as well as the physics unit.

**The slope is real physiology.** Executive function, resistance to
suggestion, and emotional regulation genuinely degrade with metabolic
depletion. A tired person is easier to move. This is not a fictional
claim.

**The intercept is the field's most consequential number.** f > 0.
Extrapolate the line to an empty reserve and it does not reach zero.
Nobody has ever measured a person who could be fully written. Whether
f is *irreducible* or merely *not yet exceeded* is a live open
question (below) — and it is not an idle one, because **the Ordinance
teaches that f is an artifact of poor technique, and its civic
project is, functionally, a long experiment in driving f to zero.**

### Tarn's Rule — the limiting faculty

> A cast's potency is governed by the **weaker** of the caster's two
> competences — the verb and the noun — not by their sum or average.

Liebig's law of the minimum, restaged. A crop grows to the limit of
its scarcest nutrient; a line runs at the rate of its slowest station;
a parallel program is capped by its serial fraction. Being excellent
at Create and poor at Fire makes you a poor fire-caster, and no amount
of further Create makes it better.

This is one of the most transferable results in the document. Students
who internalise the minimum principle carry it into nutrition,
manufacturing, project scheduling, and systems engineering.

---

## The taxonomy

The grid — five verbs by thirteen confirmed nouns — is not a filing
scheme. It is an empirical claim about how the world is jointed, and
it has been revised by experiment more than once.

### The five verbs

The verbs classify what happens to the **addressed state**, not which
way the energy moves:

| Verb | The state |
|---|---|
| **Create** | comes into existence |
| **Destroy** | ceases |
| **Control** | changes parameters, remains itself |
| **Transform** | changes what it *is* |
| **Perceive** | is read, not written |

Energy direction is separate bookkeeping. Destroy·Fire happens to
require pulling heat *in*; Destroy·Mind is a patterning act at
negligible energy. Conflating the verb axis with the energy axis is a
standard novice error.

The theory predicts a cost ordering across verbs, which is checkable
against the roster: **Perceive cheapest** (a probe and a return),
**Create middling**, and **Destroy conditional** — cheap where it
means pumping heat a short way, ruinous where it means pumping it a
long way, and dangerous either way because the caster is the sink.
Note that this is the one verb whose cost cannot be read off a table;
you have to know the temperatures.

### Transform is not forbidden. It is unaffordable.

This is the field's most misunderstood result, and it is settled.

Chemical bonds run at 1–5 eV. Nuclear binding runs at about 8 MeV per
nucleon. That is a factor of roughly **10⁶**.

- **Chemical transformation is real and affordable in small amounts.**
  A caster can rearrange bonds. That is cooking, ignition, and — at
  the margins, with great skill — some metallurgy.
- **Elemental transformation is off by six orders of magnitude.** Lead
  into gold is a nuclear operation. A caster's entire reserve, spent
  perfectly, is not within a millionth of the cost.
- **Transformation of a living body** means breaking and reforming
  every bond in it, in a specified arrangement, without killing the
  occupant. The energy is prohibitive and the specification is worse.

So the shipped state — Transform has no working spell — is not a gap
in the game. **It is the correct answer to a calculation a first-year
student can do on one page.**

**What this settles about polymorph** (magic-items D18, built 2026-08-03).
The question "can a working turn one thing into another?" has two
answers and neither of them is *transform*:

- **Item polymorph is SEMBLANCE.** A shadow on **presentation and
  recognition** — `Sense`, not `Transform`. Nothing about the thing
  changes; what changes is what an observer's belief store reads off it.
  Affordable, honest, and already what shadows are for.
- **Body-swap is not transformation at all.** It is the movement of a
  *presence* between bodies — a reified inhabitant-relation, which is
  `presence-hollowing-slate.md`'s problem and carries no material cost
  because no material moves.

Both are written into `magic-items-slate.md` too. The price list makes
the ruling structural rather than editorial:
`lib/magic/PriceList.ts` prices `transform` three orders of magnitude
above every other verb, so a *transform* item derives a rarity that
keeps it out of circulation by arithmetic rather than by a rule
somebody has to remember.

### The thirteen nouns, and how they were carved

A noun is a **domain the world keeps independent books on**. The
roster is therefore discovered, not decided, and its history is a
history-of-science curriculum in itself.

Fire (thermal) · Water (fluid and phase) · Air (gaseous medium:
breath, pressure, **sound**) · Earth (solid matter) · Light (radiant)
· Lightning (electrical) · Storm (atmospheric) · Plant · Beast · Body
(anatomy) · Mind (cognition) · Sense (belief and semblance) · Arcana
(magic itself)

**The Crowe dissociation — Light split from Fire.** For a long time
these were one noun, on the entirely reasonable grounds that fire is
hot and bright. Crowe demonstrated they dissociate: **cold light**
exists — emission whose origin is not thermal at all — and **heat with
no visible emission** exists, since conduction and convection carry
energy with no light whatever and a merely warm body radiates only in
the infrared. Two effects that can be produced independently address
two registers, and by this field's own criterion — *a noun is a domain
the world keeps independent books on* — that makes them two nouns.

**Do not claim a historical rhyme here. Real physics made the opposite
move, and that is the more interesting fact.** The actual sequence:
Lavoisier's 1789 *Traité* listed *lumière* and *calorique* as two
separate imponderable simple substances, so chemistry **began** with
them apart; Herschel found invisible heating rays beyond the red in
1800; and Melloni, working from 1831 with the thermopile, showed that
radiant heat reflects, refracts, diffracts, and polarises exactly as
light does, and concluded it is the same phenomenon differing only in
wavelength. Real physics went **separate → unified.** We went the
other way.

That is not a contradiction, and a student should be able to say
precisely why: **the two taxonomies answer different questions.**
Physics classifies *what the phenomenon is*, and radiant heat and
visible light are indeed one thing. This field classifies *which
register a caster addresses*, and thermal energy in matter is not the
same register as radiant emission. Both cuts are correct for their own
question, and neither refutes the other.

Which makes for one of the best exam questions in the course, because
the wrong answer is the sophisticated one:

> *"Melloni proved heat and light are the same phenomenon. Doesn't
> that refute the Light/Fire split?"*

A student who says yes has conflated **radiant heat** (which is light)
with **thermal energy** (which is not), and has done so by way of a
real experiment they correctly remember. Catching that is worth more
than memorising the taxonomy.

**The Air narrowing — a boundary dispute settled by measurement.**
Air was once a grab-bag. Three claims were adjudicated:

- **Sound stayed.** Sound is pressure waves *in a medium*; it has no
  existence apart from the gas that carries it. There is no separate
  Sound noun and there should not be one.
- **Temperature left, to Fire.** Thermal energy is not a property of
  gases specifically.
- **Humidity left, to Water.** It is water, in a phase.

Each of these is a real physical carve, and students argue them in
seminar before being shown the resolution.

**Arcana is reflexive** — magic addressing magic. Dispel and
arcane-sight live here. It is the noun the field is least sure of,
and there is a standing minority position that it is not a noun at all
but a bookkeeping artifact: that "detecting magic" is detecting the
*energy signature* of an unusual transfer, and "dispelling" is simply
ceasing to top up someone else's binding, neither of which requires a
distinct physical register.

**Two nouns were confirmed within living memory.** Lightning and Storm
graduated from the frontier when their underlying phenomena became
measurable. Every currently enrolled student has professors who were
working before either was accepted. **This field is not finished**,
and it is important that students are taught by people who remember
it changing.

**Time and Spirit remain unconfirmed** — no identified physical
register and no energy pathway. They are not "hard"; there is nothing
there to address, and no experiment has ever suggested otherwise.

### The Ward Argument

The strongest evidence that the grid carves nature at its joints, and
not merely at our convenience:

> **You can build a ward that filters by grid address** — one that
> blocks Create·Fire while passing Perceive·Arcana.

If the grid were a teaching convention, selective suppression would be
impossible; you could only block everything or nothing. That it works
means the *specification* of a transfer — not merely its energy — is a
physically real, addressable thing that a field can discriminate.

Which is the best available evidence that the field's central open
problem is a physics problem and not a philosophical one.

---

## The specification problem

The postulate says energy moves *in a form of the caster's choosing*.
For heat, "form" is doing almost no work: heat is what energy does
when nothing else is specified. For a delivered pattern — a mental
state, a semblance — "form" is doing nearly all of the work at almost
no energetic cost.

By Landauer's principle the minimum energy to write one bit at room
temperature is about **3 × 10⁻²¹ J**. Energetically, information is
free. So Mind and Sense effects are not limited by the reserve at all;
they are limited by *specification*, and **nobody knows what fixes the
specification.**

### ⭐ Control·Body and the terminal network

**Teleportation is the third specification-limited effect, and the one
where the economics are visible from the street.**

Relocating a body is `Control·Body` — position is a parameter, and the
body remains itself. It accelerates nothing, so its energetic cost is
only the change in potential energy, `mgh`. Horizontally, that is
nearly nothing. **A teleport ought to be almost free, and it is not**,
because you must *specify where* to an accuracy no one can hold in
their head: arriving inside a wall is not a near miss.

> ⭐⭐ **Which is why the Teleport Authority is a fixed graph of
> registered terminals rather than a go-anywhere spell.** A terminal is
> a **surveyed destination** — the specification is paid once, built
> into the infrastructure, and amortised across every traveller. You
> cannot teleport to an arbitrary point because nobody has paid the
> survey for it, and `register` is the traveller being specified into
> the system.

The fare structure falls straight out and is worth putting in front of
students precisely because the intuitive answer is wrong:

| Factor | Cost |
|---|---|
| **Altitude** | real — `mgh`; arriving uphill costs more |
| **Mass** | real — `mgh` again; a loaded traveller costs more |
| ⚠ **Distance** | **none.** Locality is the thing that broke |

A hop to the next district and a hop across the realm cost the same.
Fares vary with **how high and how heavy**, never with how far — and a
student who can say *why* has understood what the postulate actually
breaks.

This is the honest centre of the field's ignorance, and it is where a
course should locate its "we don't know." It is a gap in explanation,
not a violation of physics — which is exactly the kind of open problem
a real discipline has, and exactly the kind that teaches students how
a mature science holds a question open without abandoning its
foundations.

It also makes the Reeve Line legible. Composure is resistance to
being patterned. A depleted person's own neural state is noisier and
less well-maintained, so it is easier to overwrite — and the floor f
is the claim that a person's self-maintenance never reaches zero.

**One prediction the theory makes here is testable and load-bearing:
a veil changes no photons, so it should not fool an instrument.** A
recording device sees what is there. If that holds, then trusted
recording is the answer to semblance, and the whole evidentiary layer
of the world stands on a thaumological result.

---

## The fault line — what *f* means

Every serious field has one question where the technical and the moral
are the same question. This is thaumology's, and it is a single
shipped constant.

**The number is `f = 0.4`** (`composureFloorFactor`). Worked out:

| | drained | quarter | half | full |
|---|---|---|---|---|
| composure (mid band) | **0.40** | 0.55 | 0.70 | 1.00 |

So a person emptied to nothing keeps **40% of their resistance to
being written by someone else.** The entire mana axis — everything an
attacker can take from you by exhausting you — is worth only **2.5×**.
Draining someone is worth a great deal tactically and is **nowhere
near enough to make them writable.**

That is the claim in the equation, and it is not a small one: *there
is a floor under a person that no amount of depletion reaches.*

### The Ordinance's position

The Ordinance teaches that *f* is an artifact of technique — a
measurement of how good we currently are, not a property of persons —
and its civic program is, functionally, a long experiment in driving
it toward zero.

**The steelman matters here, and the course must give it.** If *f* can
be reduced, then so can suffering. Trauma that will not fade, an
addiction nobody can talk themselves out of, a despair that has
outlived every reason for it — all of these are patterns a person
cannot rewrite from inside, and a floor under rewriting is also a
ceiling on rescue. Someone who has watched a person they love lose to
one of those is not being wicked when they ask whether the floor is
really a *good* thing.

So the fault line is not good against evil. It is **therapeutic
against autonomous** — the oldest live argument in the ethics of the
mind, and one with no settled answer in any real discipline either.
The Ordinance's horror is not that it wants to help. It is that it
answered the question once, centrally, for everyone.

### The measurement that would settle it, and why nobody trusts it

The obvious empirical question: **is *f* the same for everyone?**

Measurements cluster tightly. The error bars are wide enough that
nobody can honestly exclude a distribution. And that is exactly where
it becomes dangerous, because **finding that *f* varies between people
would be extremely convenient for one side of the argument** — a
distribution is a sorting, and a sorting is a policy.

Which is a real thing that happens to real sciences: when a
measurement would license a conclusion someone badly wants, the
research around it stops being disinterested, and you have to reason
about the *incentives on the evidence* rather than only the evidence.
Teaching a student to notice that is worth more than teaching them any
particular fact.

The house position is anti-essentialist and should be argued rather
than asserted: the clustering is what it looks like, the variation is
not established, and a difference small enough to be at the edge of
measurement is far too small to hang a sorting on.

### Why this is better taught in an invented science

A genuinely loaded question about whether people differ in some deep
capacity cannot be put in front of a first-year student in its real
clothing without every prior in the room firing before the argument
starts.

Put it in an invented science and the priors have nothing to grab.
Students reason about the evidence, the incentives, and the ethics
**on the merits**, and only afterwards notice what they have been
reasoning about. That is defamiliarization used as a teaching
instrument, it is a standard and respectable technique in ethics
pedagogy, and it is something a real biology course structurally
cannot do with its own hardest questions.

This is, in my view, the strongest single argument for an invented
subject existing at all — not that it is easier, but that it can hold
a question a real subject cannot hold steady enough to examine.

### It connects to the other course, and to the ledger

**Compact 200 has already legislated on this.** Article I §4 — *no
number is an authority; no tally shall stand as an authoritative
verdict on worth* — is a constitutional answer to precisely the
sorting *f* would license. So the natural science and the social
science **meet at one seminar**, which is what makes the pair read as
a curriculum rather than two demonstrations.

And it is not only a discussion. Imposing a mental effect on a
non-consenting sentient already writes an **accountability row** in
the shipped ledger. The ethics of this field is not a unit at the end
of the syllabus; it is enforced in code, on every cast, and the record
is admissible.

## Instruments

An instrument is not a way to see the truth. **An instrument is a
specific way to be wrong**, and knowing which way is the whole of
laboratory competence. Every entry below is paired with its
characteristic error, because the error is the teaching content and
the reading is just a number.

### The organizing spine: primary and secondary

Most of a real metrologist's discipline is **traceability** — knowing
which of your instruments is a standard and which is a convenience,
and checking the second against the first on a schedule.

- A **primary** instrument produces a value from first principles.
  The stone basin is primary: `Q = mcΔT` is traceable to mass,
  specific heat, and temperature, and nothing needs calibrating
  against anything else.
- A **secondary** instrument is convenient, fast, and *meaningless
  until calibrated against a primary.* The thermopile is secondary.

This distinction is the single most useful thing in this section, and
it is the whole explanation of the noon dip: **a secondary instrument
was trusted for years without being re-checked against the primary.**
Traceability is not bureaucracy. It is the thing that would have
caught it.

| Instrument | Reads | Tier | Characteristic error |
|---|---|---|---|
| Stone basin (calorimeter) | delivered joules | **primary** | slow; leaks to the room |
| Thermopile | fast thermal transients | secondary | cold-junction drift with ambient |
| Reserve gauge | a caster's E and D | secondary | biological variability; **reading it spends it** |
| Field survey meter | local suppression | secondary | point sampling of a field |
| Probe detector | incoming Perceive probes | secondary | no threshold is a good threshold |
| Thermometer | everything Fire does | primary | the most useful instrument in the field |
| *Thaumometer* | *nothing* | — | *see below* |

Note the ordinary thermometer sitting there. **Most thaumological
measurement is done with plain physical instruments**, because most of
what magic does is ordinary physics arriving from an unusual
direction. Students who expect exotic apparatus are surprised, and the
surprise is the lesson.

### The basin — honest, and slow, and leaky

Twenty kilos of granite (c ≈ 0.79 kJ·kg⁻¹·K⁻¹) taking a 29.9 kJ
delivery rises **1.89 K** — nineteen divisions on a 0.1 K thermometer,
comfortably resolvable.

But it **leaks.** At a modest 5 W to a cooler room it loses 3 kJ over a
ten-minute session — **10% of the signal**, and all of it in the
direction of making magic look weaker than it is. So you never read
the peak: you record the cooling curve and **extrapolate back to
t = 0.** That is not a trick, it is what real calorimetry *is*, and a
student who has done it will never again believe that reading a
thermometer is the same as measuring energy.

And it is **slow.** Its thermal time constant is around 50 minutes,
while a firebolt lasts under a second. The basin never sees the
transient at all — it only ever tells you the total. Which is the
trade in one line:

> **Slow, primary, honest** — totals and calibration.
> **Fast, secondary, drifty** — transients and convenience.
> You need both, and you must keep checking the second against the
> first.

### The reserve gauge — and the observer effect that isn't imported

Two errors, and the second one is structural.

The mundane one: it reads **a living system**, so it drifts with
hydration, recent food, sleep, and time of day. That is biological
variability, not instrument noise, and the fix is the real one —
repeated measures and within-subject designs rather than one reading
and a conclusion.

The structural one falls straight out of the postulate. Perception at
a distance is an **active probe** — you send energy out and read what
returns. So **measuring a reserve spends some of it.** You cannot
weigh the tank without drawing from it, and the error is worst
precisely where precision matters most: on a nearly-empty reserve, the
measurement is a meaningful fraction of what is left.

This is a genuine observer effect, and the reason it belongs here is
that it was **not imported for flavour** — nobody added it to make the
science feel quantum. It is forced by a postulate written for other
reasons, which is exactly how real theoretical constraints behave.

### The survey meter — you measured where you stood

Maps suppression by walking the boundary. Its error is **spatial
sampling**: you know the field at your sample points and nowhere else,
and the temptation to draw a smooth line through them is enormous. A
boundary that looks like a gradient at five paces' spacing may be a
step; one that looks like a step may have structure you walked past.

The lab is to map the same ward at two spacings and see the boundary
change shape. Real method — the same problem shows up in geophysics,
epidemiology, and polling, and "we interpolated between the points we
had" is behind a great many confident wrong maps.

### The probe detector — where the base rate eats you

The detector has a sensitivity threshold, and here is what that
actually buys, assuming someone genuinely is scrying you about 2% of
the time:

| Threshold | Catches | False alarms | **Alarms that are real** |
|---|---|---|---|
| high | 50% | 0.5% | **67%** |
| medium | 80% | 5% | **25%** |
| low | 97% | 30% | **6%** |

Turn the sensitivity down to catch nearly everything and **94% of your
alarms are nothing.** There is no setting that gives you both, the
trade is forced by the base rate rather than by the quality of the
instrument, and choosing where to sit on it is a **value judgement**
wearing a technical costume.

This is signal detection and base rates — arguably the most
consequential piece of statistical reasoning a person can carry
around, and one most undergraduates never meet. It also produces a
character: the mage who runs the detector wide open and sincerely
believes they are being watched constantly is not lying, and is not
mad. They are misreading a true instrument.

### The thaumometer — the founding *negative* result

The oldest instrument in the field measures nothing, and that is the
most important thing that has ever happened to it.

Early theory assumed what every theory assumes: a **medium.** Magic
must propagate through something, so there is a thaumic field, and it
must have a local strength you can read. The thaumometer was built to
read it.

It measured nothing. A better one was built, and measured nothing more
precisely. The instrument has been rebuilt at increasing sensitivity
for three generations and the reading has never moved off zero.

**The null result is what forced the shape of the postulate.** No
medium, no field, no local intensity — only two endpoints and a
transfer between them. Everything distinctive about this science, up
to and including *the caster is always one endpoint*, is downstream of
an instrument that never read anything.

The real referent is **Michelson–Morley**: an apparatus built to
detect the luminiferous aether, a null result, better apparatus, a
better null, and a physics that had to be rebuilt around the absence.
A field's most valuable experiment is quite often the one that found
nothing, and this is the cleanest way to teach that.

The thaumometers are still in the cabinets. They still read zero. It
is the most-repeated measurement in thaumology and every student
should make it once, precisely so they learn what a null result is
worth.

> **Supersession note.** The old capability-magic slate's Part II
> instrument sketch listed a *thaumometer* reading "ambient thaumic
> field" and an *affinity-meter* reading per-school coupling. Under
> the postulate **neither measures anything that exists** — there is
> no field, and there is no affinity (there is species faculty band
> and there is learned competence). Do not author them as working
> instruments. The thaumometer survives only as above: the honoured
> apparatus that reads zero.

---

## The literature

Reading papers *is* the discipline. A student who can derive a firebolt
cost but cannot tell a solid result from a fashionable one has learned
the easy half. So the field ships a **shelf**, and the shelf is
designed rather than accumulated.

> **The machinery lives elsewhere.** How a paper gets published,
> replicated, refuted, or believed is the inquiry substrate's job
> (`slates/builds/inquiry-slate.md`) — which is explicit that it is a
> *substrate, not a content catalog.* This section is the content: the
> actual shelf, and what each entry is for.

### The ratio is the curriculum

The most important design decision here is how much of the shelf is
wrong, and the instinct to make it mostly traps is exactly backwards.

> **A shelf where everything is a trap teaches distrust. A shelf where
> nothing is teaches credulity.**

Real literature is mostly right, occasionally wrong in instructive
ways, and rarely dishonest. The skill being taught is **calibrated
trust**, not blanket suspicion — a student who leaves believing all
published work is suspect has been mistrained just as badly as one who
believes it all. Roughly **five solid, two instructively flawed, one
actively wrong** is the shape.

### The four-part read

Every paper is read the same way, and the fourth part is the one
nobody teaches:

**Claim** — what is asserted. **Evidence** — what was actually
measured. **Range** — over what conditions. **Silence** — what was not
measured, not reported, and not mentioned.

Most bad papers are not false in their claims. They are **silent in
the right places.**

(Compact 200 teaches the identical skill on constitutional text —
*provision, purpose, tension, silence.* Same muscle, different
document. The two courses share it.)

### Where the teeth are

Because the sim is the answer key, **every claim on this shelf is
re-runnable**, so no paper can survive on authority. What varies is
the **cost of checking**, and that is what makes a wrong paper
dangerous:

- **Cheap to check** — one cast. A lie here dies the same afternoon.
- **Expensive** — a rare instrument, a long series, a scarce material.
- **Dangerous** — verifying means doing the thing the paper claims is
  safe.
- **Noisy** — a variable effect where one trial cannot separate a
  subtle error from ordinary scatter.
- **Frontier** — no established truth to check against; the paper is
  the only source.

**Design rule: put the wrong papers where checking is expensive.** A
falsehood parked in front of a one-cast verification is not a lesson,
it is a speed bump.

### The starter shelf — eight papers, eight jobs

| # | The paper | Status | What it is for |
|---|---|---|---|
| 1 | Halloway's calorimetry | **solid**, superseded in precision | reading a classic; right ≠ final |
| 2 | *On the Absence of a Range Term* | **overreach** | correct within evidence, unqualified beyond it |
| 3 | *Range Independence Confirmed* | **overreach — and chosen** | indistinguishable from #2 on the page |
| 4 | The noon-dip corpus | **wrong** (instrument artifact) | an effect that lives in one instrument |
| 5 | *An Apparent Surplus in Enclosed Trials* | **retracted** | extraordinary claims; what a retraction looks like |
| 6 | *Recovery at Riverside Sites* | **confounded** | real effect, wrong cause |
| 7 | *A Small Difference in Delivery* | **noise** | power, replication, the winner's curse |
| 8 | *A Note on Familiar Observers* | **solid, ignored** | publication ≠ influence |

Four of these deserve their reasons written out.

**#4, the noon dip.** The apparatus anomaly, with a literature on top
of it: several papers, a proposed mechanism with a respectable name,
and a reputation resting on it. Cheap to check *if you think to use a
different instrument*, and nobody did for years. That gap — between
"checkable" and "checked" — is the whole lesson.

**#6, riverside recovery.** A genuine, replicable effect: casters do
recover faster at the river sites. The paper attributes it to the
water. The cause is that the riverside pavilions have better seating
and people rest longer and more comfortably there — recovery scales
with rest quality, which is shipped mechanics, not hydrology. The
effect is real, the measurement is honest, and the conclusion is
wrong. **Teaches confounding better than any invented example,
because a student can go and sit in the other chair.**

**#7, the small difference.** Claims a 0.02 difference in delivery
efficiency between two groups, from **eight casters per group**,
against a between-caster scatter of about 0.05. It has never
replicated, and the arithmetic says why:

| n per group | power |
|---|---|
| 8 | **12%** |
| 20 | 24% |
| 60 | 59% |
| 200 | 98% |

At n = 8 the study had about one chance in eight of detecting its own
effect if the effect were entirely real. And the sting is the second
result: at that sample size, *any* result significant enough to
publish must be at least **0.049** — **2.5× the effect being claimed.**

> **An underpowered study that succeeds necessarily overstates.**
> Every time. Not sometimes.

That is the winner's curse, it is the engine of the real replication
crisis, and it is teachable here in one table against a real open
problem in the field.

**#8, the note on familiar observers.** Someone measured the
familiarity term years before it became famous, published it in an
unfashionable corner, and nobody followed up. It is entirely correct.
It is now anomaly III. Teaches the thing students least expect —
**that the stacks contain treasure, and that being right is not the
same as being read.**

### The paired exercise — the one that can't be answered

Papers #2 and #3 make the same claim, over the same range, with the
same quality of data, and are the same shape on the page. One author
measured what they could and generalized carelessly. The other chose a
range that excluded where the claim fails, and knew.

Hand a student both and ask which is which.

**The correct answer is that you cannot tell from these documents** —
and the follow-up question is the real one: *what evidence would bear
on it?* (Track record. What else the author published. What equipment
they had. What they were asked. Whether they went back afterwards.)
Intent lives in provenance, never in the claim.

This is the inquiry substrate's centerpiece in exam form, and it
teaches something students are rarely told outright: **some questions
are not answerable from the artifact**, and knowing which ones is a
skill.

### Scope discipline

**Eight papers, eight jobs.** Every entry earns its place by teaching
something no other entry teaches. A ninth paper needs a ninth job, and
"the shelf looks thin" is not a job — a paper that merely restates a
correct result is decoration, and decoration is what makes an invented
subject read as a theme.

## The history of the field

**The field exists because the magic got weak enough to study.**

In the Eternal Age, will-into-being was fiat. Nobody measures a thing
that simply obeys, and the archwizards left no data — they left
wonders. What the Fall took was the *will*; what it left was a
diminished, effortful, thermodynamically honest residue.

**The Fall is the moment magic began paying for itself.** Everything
in this document is the science of what remained. The felix culpa the
bible states for craft holds for knowledge as well: friction is what
made the field possible, and a power that cost nothing taught nobody
anything.

The sequence, in rough order:

1. **Folk practice** — centuries of technique without theory,
   including a great deal of correct practice held for wrong reasons.
2. **Halloway Equivalence** — the calorimetry that made a mana point
   a kilojoule and turned a craft into a science.
3. **Kell's Partition** — the impulse/binding cut, which explained
   suppression retrospectively and correctly predicted that a
   magically-lit fire is not suppressible.
4. **The Crowe dissociation** — Light split from Fire; the field's
   first taxonomic revision by experiment.
5. **The price list** — the noun cost ordering. Established over
   roughly two decades by many hands and **attributed to none of
   them**, which is how most science actually accumulates and is worth
   pointing out to students raised on a great-man narrative.
6. **Reeve, Tarn, Voss** — the modern quantitative laws.
7. **Lightning, then Storm** — two nouns confirmed within living
   memory.

Note the surnames. The ancients have grand names — Solus, Solvan. The
scientists have ordinary ones. That is not an accident of style: the
making power was hoarded when it was fiat, and the knowledge of it was
built by whoever showed up with a thermometer.

---

## Misconceptions

Every one of these is held by somebody in the world, taught by
somebody, and printed in at least one bad paper. They are the
teaching bank and the wrong-paper seed bank at once.

| Misconception | Correction |
|---|---|
| "Magic creates energy." | It relocates energy. Global conservation holds. |
| "Mana is a magical substance." | It is your metabolic reserve, in kilojoules. |
| "Cold spells destroy heat." | Heat cannot be destroyed. It goes into you — all of it, plus the work you spent moving it. |
| "Cooling must cost more than heating." | Near ambient it costs *less*. A heat pump moves more heat than the work it consumes. |
| "A cheap spell is a safe spell." | Conjure-water costs ~34 τ and kills you on the fourth casting. |
| "Magic pushes without being pushed." | Momentum is conserved. Brace, or you go the other way carrying the same energy. |
| "Melloni unified heat and light, so the Light/Fire split is wrong." | He unified *radiant heat* with light. Thermal energy in matter is a different register. Two taxonomies, two questions. |
| "Our taxonomy recapitulates the history of physics." | It does not. On light and heat, real physics went separate → unified; we went the other way, for a different reason. |
| "A bigger caster recovers faster." | Capacity and rate are independent axes. |
| "Levitation costs because holding a weight is work." | Static holding does no work. The cost is the leak. |
| "Suppression puts out magical fire." | A lit fire has no author. Only bindings are suppressible. |
| "You can conjure water anywhere." | Yield is humidity-limited. Deserts do not cooperate. |
| "Transform is forbidden by law." | It is unaffordable by six orders of magnitude. |
| "Mastery has no ceiling." | Mastery is efficiency. η ≤ 1. |
| "Distance costs more." | No distance dependence has been observed — **within the measured range.** See below. |

### The one that matters most

**"There is no distance term"** is the field's most-cited result and
its most dangerous, because it is true *within the range anyone has
measured*, and that range is a room.

Every published measurement of range dependence has been taken at
under about 30 m, because that is as far as anyone has reliably
delivered. The claim "the postulate has no distance term" is therefore
an **extrapolation from an evidential range**, and it is stated
without that qualification in most textbooks.

This is the canonical example of the honest overreach the inquiry
substrate is built around: correct within its evidence, unqualified
beyond it, and indistinguishable on the page from deliberate
curve-fitting. It is also, not coincidentally, exactly the shape of
the engine's own limitation — magic is room-scoped until the ranged
build lands. **The game's technical boundary and the science's
evidential range are the same boundary,** which means extending the
one is a genuine discovery event in the other.

---

## Worked examples

### Firebolt — Create·Fire

Target: a 50 g bundle of dry straw at the point of impact.

- Heat to autoignition: `0.050 kg × 1.5 kJ·kg⁻¹·K⁻¹ × 260 K ≈ 20 kJ`
- Drive off residual moisture (8% by mass, 4 g):
  heating `0.004 × 4.18 × 80 ≈ 1.3 kJ`, vaporising
  `0.004 × 2260 ≈ 9 kJ` *(2260 kJ/kg is the latent heat at boiling;
  conjure-water uses 2450 because it condenses at 20 °C — same
  substance, different temperature, and a student who notices the
  discrepancy has understood something)*
- Delivered: **≈ 30 kJ**. At η = 0.85, spent: **≈ 35 τ**

That moisture term is the whole reason wet wood will not light, and a
student who has done this calculation understands campfires forever.

**You pay for joules, not for outcomes.** The spell commits a fixed
budget; whether it ignites the target is a real materials question
that the fire subsystem answers. The same firebolt that lights straw
merely scorches wet oak and does nothing at all to a stone basin. This
is a correction to how nearly every other game handles magic, and it
is what makes the system teachable: the caster controls the input, the
world determines the result.

### Conjure-water — Create·Water

Matter creation is 9 × 10¹⁶ J/kg and therefore out. So the spell is a
**dehumidifier**: pump heat out of the surrounding air until its water
condenses.

- Latent heat at 20 °C: **≈ 2.45 MJ/kg**, so condensing 100 mL means
  removing **Q = 245 kJ** from the air.
- The lift is small (20 °C air → 37 °C caster), so a realistic COP of
  ≈ 7 makes the **work only ≈ 34 τ.** Conjure-water is *cheap.*
- But the caster absorbs `Q + W` = **279 kJ**, and
  `279 / (70 × 3.5) ≈ +1.14 K` of core temperature.
- Four castings is **+4.5 K**. That is hyperthermia, then death.

Note what that pair of numbers does: the spell costs about a tenth of
a mid reserve and will kill you on the fourth casting. **The mana bar
is not the danger meter**, and a student who has done this
calculation will never again assume the two are the same.

And the supply side is real psychrometrics. Air at 20 °C and 50% RH
holds 8.6 g/m³. Extracting 100 g means processing on the order of
20–40 m³ — most of a room — and **leaving it noticeably drier.**

The mechanic falls out on its own, and none of it had to be invented:

- **Cheap in fog**, where the air is already at its dew point and you
  need only nudge it.
- **Impossible in a desert**, where the water is not there to collect.
- **Self-limiting by heat**, not by mana — the caster is the condenser.
- **Leaves evidence.** A dried, disturbed room is a forensic trace.

The weather subsystem already models humidity. This is an authoring
rule, not new code.

### Glowlight — Create·Light, a binding

100 lm at a realistic broad-spectrum efficacy (≈ 150 lm/W) is about
0.7 W radiant. At η = 0.30, the draw is ≈ 2.2 W ≈ **8 τ per hour.**
An all-night lamp costs about 60–70 τ out of a mid reserve of 300.

Glowlight is cheap because nothing in the world is working to make the
room dark. Compare a levitation, which fights gravity every second it
exists. Same law, opposite leak rate.

### Spark — Create·Lightning

Air breaks down at about **3 kV/mm**, so a 1 cm gap needs ≈ 30 kV.
Once it breaks down, the real conduction graph resolves the current —
and the danger is the current, not the energy. A shock that would stop
a heart is a few tens of joules: cheap in thaums, lethal in amps.

**And the caster is an endpoint.** A mage casting spark while standing
in a brine pool is part of their own circuit, by the postulate, not by
bad luck. The Practicum's conductive gallery is a demonstration of the
postulate's central clause, and it was built before anyone wrote it
down.

### Dread — Destroy·Mind

Energetically negligible (Landauer). Entirely a specification act.
Resisted by the Reeve Line against the target's live reserve.

This is the cell where the theory is weakest and should be taught as
such. We can say precisely what it costs (almost nothing), precisely
what resists it (composure, affine in reserve, floor f > 0), and
precisely what we cannot explain (what fixes the pattern). Being able
to state the boundary of your own explanation with that much precision
is itself the lesson.

---

## Open problems

These replace the "where does the energy come from" question, which is
answered: it comes from the caster, and the books balance.

1. **The specification problem.** What fixes the form of a transfer?
   Acute for Mind and Sense, where the energy is negligible and the
   pattern is everything.
2. **Distance dependence.** No term observed under ~30 m. Nobody has
   measured further. Nobody should say "there is no term."
3. **The suppression mechanism.** Extremely well characterised
   behaviourally — it blocks assertion, not energy, and it can
   discriminate by grid address — and completely unexplained.
4. **Efficiency scatter.** Two casters of identical competence do not
   deliver identically. The species faculty profile predicts most of
   it. Nobody knows what the residual is.
5. **Is f irreducible?** The composure floor has never been beaten.
   Whether that is a law or a limit of technique is the most
   consequential open question in the field, and it is being tested
   right now, at scale, by people we do not want to be right.
6. **Is Arcana a noun?** Or is it the field's bookkeeping mistaken for
   a register?
7. **What was will-into-being?** A historical question the field
   deliberately declines to answer, on the grounds that there is no
   surviving phenomenon to measure. Some students find this
   unsatisfying. That is a good conversation to have with them.

---

## The anomalies

An open problem is *"we don't know X."* An **anomaly** is worse and
better: a measurement sitting on a bench that the theory does not
account for. Real fields are driven by these, and a field without any
reads as finished — which is the surest way for an invented subject to
feel like a toy. A course whose textbook is complete teaches a
catechism.

Three ship, and they are deliberately three *different kinds of
wrong*, because the meta-lesson is the ordering:

> When a measurement doesn't fit, the prior is **you**, then your
> **instrument**, and only rarely **nature.** But "rarely" is not
> "never," and telling them apart is the entire job.

If both of the first two dissolved and nothing survived, the lesson
would be the cynical one — that anomalies are always mistakes. That is
false about real science, so the third one is real and stays open.

> ⚠ **The resolutions below are instructor material.** Students get
> the measurement, the published claims, and the apparatus. They do
> not get this section.

### I. The sealed-chamber surplus — *your accounting was wrong*

**What you observe.** Repeat Halloway's calorimetry in a sealed
chamber — better insulation, cleaner numbers, obviously superior
method — and conservation appears to fail. The chamber gains far more
energy than the caster spent. Not marginally: over a careful
ten-minute session, roughly **212% over budget.**

**What the field says.** This is the anomaly that produces the most
bad papers, because it looks like the big one. Every few years someone
announces that the postulate leaks.

**Resolution.** The experimenter is inside the calorimeter. An adult at
rest radiates about **97 W** — over ten minutes that is **58 kJ**,
nearly **twice** the 29.9 kJ the spell delivered. Add the 5.3 kJ of η
loss that also lands in the caster, and the "surplus" is entirely a
warm body standing in a sealed box.

The fix is ordinary lab practice: shorten the exposure, run a no-cast
baseline to measure the drift, or **get the caster out of the
chamber** — which the postulate expressly permits, because the
transfer needs no medium and only one endpoint has to be the caster.

That last detail is why this anomaly earns its place: **the experiment
that corrects the error is itself a demonstration of the central
claim.** You fix your accounting by doing the one thing that proves
the postulate.

**What it teaches.** *Define your system boundary* — the most common
and most costly error in real thermodynamics, and one that is nearly
impossible to teach as a rule and trivial to teach as a burn. And it
rhymes with the field's whole character: you forgot to include
yourself, in a discipline whose one constant is that you are always
the other endpoint.

**Build cost.** A sealed room and a thermometer. Both shipped.

### II. The noon dip — *your instrument was wrong*

**What you observe.** Delivery efficiency measured at midday is about
**5–6% lower** than the same cast measured at dawn. It replicates. It
has replicated for years, in multiple labs.

**What the field says.** It is in the textbooks. There is a proposed
mechanism with a respectable name and a body of published work behind
it. It is the sort of result people build a career on top of.

**Resolution.** It is the thermopile. The standard instrument reads a
difference across its junctions, and an uncompensated one loses
sensitivity as the whole apparatus warms — about 1% at 20 °C rising to
6% at 35 °C. Midday is hot. **The instrument under-reads exactly as
much as the "effect."**

The tell is the one that always works: **the effect lives in one
instrument.** Repeat with a water calorimeter or a liquid-in-glass
thermometer and it is gone.

**What it teaches.** Calibration, temperature coefficients, and the
single most transferable rule in laboratory practice — *an effect that
appears in only one method is a property of the method.* Plus
something real about how science actually works: this one has papers
to retract and a reputation attached, so the correction is socially
expensive and slow, which is why bad results persist long after
they're checkable. That is not cynicism, it is the replication
problem, and it is the honest reason a discipline needs more than
good intentions.

**Build cost.** Two instruments with different failure modes, and the
existing day/night temperature cycle. The anomaly *is* the difference
between them.

### III. The familiarity term — *genuinely open*

**What you observe.** Commit an identical 40 τ to an identical veil,
change nothing but **who is looking**, and the duration moves by two
orders of magnitude:

| Observer | Holds for | Implied λ |
|---|---|---|
| a stranger | 6 h | 1.9 W |
| an acquaintance | 1.5 h | 7.4 W |
| a close friend | 15 min | 44 W |
| an intimate | 3 min | 222 W |

**120× in duration, for the same energy, the same act, and the same
caster.**

**Why it is a scandal and not a curiosity.** Voss Decay says
`t = E / λ`. So λ — a leak rate, in watts — is a function of *what
someone believes about you.* Which raises a question the field cannot
answer and cannot dismiss:

> **What is the exchange rate between a prior and a watt?**

This is the one place in thaumology where **the dimensions do not
close**, and the document's own enforcement mechanism is what finds
it. Everywhere else, dimensional analysis catches a fudge. Here it
catches something real: a term with no units, doing measurable work.

**Resolution.** There isn't one. This is the **specification problem**
in its most measurable form, and it is the field's live frontier.

Two camps, and the split is not merely academic. One holds that the
theory is **incomplete** — that there is a physical quantity behind
familiarity nobody has isolated yet, and λ will turn out to be a
function of it. The other holds that **belief is itself physical**,
and that the equations are telling us something we don't want to hear.

The second reading has an interested party. If belief is physical, it
is engineerable — and engineering belief at population scale is
precisely the Ordinance's civic program. **A technical dispute with a
moral fault line running through it**, which is what a serious field
looks like and what a toy never has.

**What it teaches.** That some anomalies survive. That dimensional
analysis is a discovery tool and not only a hygiene rule. And that the
question of which camp is right is not going to be settled by
argument — only by somebody finding the missing quantity, or failing
to.

**Build cost.** Nearly nothing. The belief substrate already models
per-viewer recognition, so the relationship is *already true of the
running world.* The anomaly is not authored; it is **measured off
shipped behaviour that nobody has written down.**

That last point is the general principle and the reason anomalies are
the best content in this subject: the world already computes more
relationships than the theory explains. Finding them is cheaper than
inventing them, and they are true.

## What runs on the shipped build today

Five labs, no new code:

1. **Halloway calorimetry.** Cast into the casting yard's stone basin,
   measure ΔT, compute delivered joules, compare against mana spent,
   derive η. The founding experiment, re-run by every first-year.
2. **The Reeve Line.** Drain a subject in steps, stage an imposed
   condition, plot C against reserve fraction, fit, extract the
   intercept. Discover f > 0 personally.
3. **The Ward Argument.** Grid-filtered suppression in the warded
   cell: block one cell, pass another, establish that specification is
   discriminable.
4. **Boundary mapping.** Suppression resolves by an outward
   containment walk, so a ward has a *geometry*. Walk it with a meter
   and map it. This is real field-survey method.
5. **Circuit membership.** The conductive gallery. Demonstrate that
   the caster is an endpoint by getting shocked by your own spark.

Plus **overchannel hysteresis**, which is a genuine hysteresis loop —
a concept students routinely get wrong — sitting in the build already.

The Practicum was built as a demonstrator for the mechanism. It turns
out to be a teaching laboratory that was only ever missing its theory.

---

## What this science demands of content

These are binding on anyone authoring magic content. They are the
price of the hard rule at the top of this document.

1. **Spell cost is energy committed, in kilojoules.** Authored costs
   must be derivable from the price list and the intended delivery.
   A cost that cannot be justified this way is a defect.
2. **Effects are authored as energy delivered, never as outcomes.**
   The world decides what happens when it arrives.
3. **Every new noun requires an identified physical register and an
   energy pathway.** No register, no noun. This is the in-fiction
   statement of the engine's governing invariant — *an Effect exists
   iff a gated Api already does the work* — and the two must never
   drift apart.
4. **Perceive is cheapest. Cooling has no fixed price** — it is a
   Carnot function of the temperature lift, cheap near ambient and
   divergent at depth — **and its limit is the caster's body
   temperature, not their reserve.** A cold spell authored with a flat
   mana cost is a physics error.
5. **Binding duration is `E / λ`.** A binding whose duration was
   picked because it felt right is a lie about a leak rate.
6. **η ≤ 1, always.** No artifact, augment, or authored effect may
   deliver more than was committed.
7. **Momentum is conserved.** Anything delivering kinetic energy
   recoils onto the caster. An unbraced push costs double and moves
   the caster too.
8. **If it cannot be dimensionally analyzed, it does not ship.**

---

## Course-readiness

The syllabus is not this document's job — that synthesis happens
against the study.com product analysis. What this document guarantees
is that the material is *there*, in the shape a real course needs,
across the strands a real science course actually carries:

- **Physics and chemistry** — energy accounting, specific and latent
  heat, phase change, efficiency and exergy, **heat pumps and the
  Carnot bound** (including the counterintuitive COP > 1), the third
  law arrived at as a diverging cost curve, **momentum conservation
  and recoil**, dielectric breakdown, Landauer's principle. All real.
- **Mathematics** — dimensional analysis, unit conversion, affine and
  exponential models, curve fitting, extrapolation and its dangers,
  error bars, orders of magnitude. Real math on invented data
  transfers completely.
- **History of science** — the Crowe dissociation set against the real
  Lavoisier → Herschel → Melloni sequence (which ran the *other*
  way), the Air narrowing, two confirmations within living memory, one
  major result with no name attached to it. How categories are made,
  revised, and occasionally mis-credited — and how two correct
  taxonomies can disagree because they are answering different
  questions.
- **Scientific literacy** — the four-part read (claim · evidence ·
  range · **silence**) over an eight-paper shelf built so that
  calibrated trust, not blanket suspicion, is what a student leaves
  with. Includes statistical power and the winner's curse, confounding
  a student can go and sit in, and the paired overreach papers that
  cannot be told apart from the page.
- **Ethics and philosophy of science** — falsifiability, the meaning
  of f > 0, consent to being patterned, and a named antagonist whose
  entire program is the denial of one measured intercept.
- **Laboratory practice** — five experiments that run today, with
  controls, replication, and a calorimeter that has been sitting in
  the casting yard the whole time. Plus the instrument curriculum
  proper: **traceability** (primary vs secondary standards),
  extrapolating a cooling curve back to t = 0, response time against
  accuracy, spatial sampling, **signal detection and base rates**, and
  what a null result is worth.

The invented content is one postulate and one taxonomy. Everything a
student is graded on is real. That is the whole demonstration.
