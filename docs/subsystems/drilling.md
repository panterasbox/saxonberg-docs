# Drilling — a hole in the ground that pays wages

The drilling trade: how a bore works, what it costs, and the one thing
about it nobody can ever find out without paying.

> ⭐⭐⭐ **The premise, in one sentence.** The survey has **two factors
> and only one of them has a channel.** A trap's STRUCTURE is readable —
> with a bracket that narrows with the instrument and the band. Whether
> the trap is CHARGED is seeded, decided before anybody looked, and
> **there is no reading anywhere in this game that reports it.** That is
> why a dry hole survives every improvement to the instruments, and why
> the money a bore spends is a **bet** rather than an expense.

## The shape of it

| | what it is | where |
|---|---|---|
| the body | a `FluidBody` on the ground column — a trap, a fluid, a charge | `ground` pack, `Deposit` |
| the hole | a `Wellhead`: depth, liner, banked swings, four bulk seams | `trade-drilling` |
| the rig | a `Derrick`, which affords every act | `trade-drilling` |
| the lift | a `Bailer`, which affords the acts you need **before** a rig | `trade-drilling` |
| the payroll | a `DrillingOutfit extends Business` | `trade-drilling` |
| the record | a `bore` document, filed under the trade | `trade-drilling` |
| the withdrawal | a `body` document, keyed on the BODY | `ground` pack |
| the reads | `measure structure` · `measure head` | two `Reading` rows |

## ⚠⚠ A bore is a POINT, not a PLACE

A hole mints no `Location`, adds no node to the location graph, and has
no inside anybody walks into. The wellhead is a fixture standing in an
ordinary surface room and asks the ground **through that room**
(`GroundPointMixin`). Everything else in the design follows from this: if
a hole needed a room to know where it was, every hole in the realm would
be a room.

⭐ `GroundPointMixin` composes `GroundQueryMixin`, which is the
place-based position trio — resolve the deposit through the zone
citation, convert the cell to metres through `cellSize`, derive the seed
from the covering Locality's address. That trio had been written three
times privately (`trade-mining`'s `SurveyReading`, `trade-farming`'s
`SoilReading`, and this build's channel would have been the fourth)
before the ground pack got a home for it.

## ⭐⭐⭐ The trap, and why the crest matters

A lode is a plane you cut; a fluid body is a volume you tap, and they
share no field. A body is a **fold**: a crest, an axis, two half-extents
and the vertical closure under the crest.

The fluid leg is **derived from that** and not authored:

```
spillZ  = crest.z − closureM          (the same everywhere)
topZ(x,y) = crest.z − closureM × r    (r = 0 at the crest, 1 at the rim)
leg(x,y)  = topZ − spillZ = closureM × (1 − r)
```

⭐ **So the leg is the full closure directly over the crest and nothing
at all at the rim.** That is the single most load-bearing piece of
arithmetic in the trade, because it is what makes the structural survey
worth paying for rather than decorative: **a bore at the rim of a charged
trap finds a metre of it and the same money at the crest finds forty.**

⚠ The first cut of this was a SLAB — `topZ`/`baseZ` authored on the body,
uniform wherever the trap covered you — and writing the channel's prose
is what exposed it. With a slab the crest's bearing and distance are
noise, a narrower bracket buys nothing, and *a better instrument gets a
narrower structural bracket* is a true sentence about a number nobody
needs. The two author fields went away with it, which is the usual
dividend: a derived leg cannot contradict its own closure.

Capacity derives from the same shape — `∫(1 − r) dA` over an ellipse is
`π·a·b / 3`, so a closed volume is a third of the bounding prism's, times
the pore fraction. An author who wants a specific well life pins
`capacityL` and the derivation never runs.

### Charge

`isCharged(body, seed)` is the pin over the lean over
`Seeded.unit(seed, hash('charge:' + key))`. ⚠ It has **no `Reading` row,
no channel token, no bracket and no band that narrows it**, and nothing
in the pack reads it before the bore reaches the leg.
`uncertainty.md`'s epistemic provenance at its purest: the ground was
always this way, and two worlds from one seed give one answer.

## ⭐⭐ The structural read, and the bracket that may not leak

`measure structure` reports, per body under this ground: the crest's
depth, how far off it is and in what direction, the axis bearing, the
total closure, and **how much of that closure is under your feet**.

⭐ **The error is a FRACTION of the reading, and it widens with depth.**
A bearing's error does not scale with the bearing (mining overrides the
ratio default with absolute degrees for exactly that reason); a depth's
does, and sharply, because what a structural survey measures is a ratio
and the error compounds over the distance it is projected through. See
`DEPTH_FRACTION` — `{untrained .5 … expert .04}`. **That is what makes a
deep bet different in KIND from a shallow one**, and why an improving
prospector's first real gain is being able to chase deep ground at all.

⚠⚠ **And the quoted half-width is a fraction of the READING, never of
the truth.** A bracket scaled off the truth would **leak the truth
exactly**: quote *± 55 m* at a half fraction and the reader knows the
crest is at 110 m to the metre, which makes an untrained eye the sharpest
instrument in the game. The reading is solved as

```
R = T / (1 − u·f)        |u| ≤ 1,  f ≤ 0.5
quoted half-width = f·R
```

which contains the truth for every `u` by construction and is computable
by the player from what they were told and nothing else. A real
instrument's accuracy is quoted as a percentage of reading for the same
reason.

⚠ A charged body and a dry one produce an **identical** reading, field
for field, at every band. There is no field in `StructureReading` that
charge touches.

## ⭐⭐⭐ The bottom of the hole is not the wellhead

The piece of physics the whole Stage A → Stage C arc hangs on.

- A body with **head** drives its own fluid to the surface. Flow
  accumulates in the wellhead's own slot on the clock, and you
  `fill <cask> from <wellhead>` like a tap.
- A body with **no head** — most of them, and brine certainly — sits at
  the bottom of the hole. `bail` is the only way up: a leather bucket on
  a rope, `BAILER_L` at a time, which is how every well on earth worked
  before there was a pump.

So a dead well's slot is empty not because of a gate but because the
brine is a hundred metres down. ⭐ **`Wellhead.liftL()` is the pump
build's entire attach point** — one number and one act, and the half that
matters is turning bucket-at-a-time into a continuous rate.

Head itself is derived and never stored:

```
headAtm(now) = headAtm0 × (1 − Σdrawn / capacity)
```

⭐⭐ and the sum is over **every straw in the body**, which is why
withdrawal lives in a register rather than on the hole: two owners who
have never met watch the same gauge fall. `measure head` is the only
instrument that reports it, **nothing in the trade announces a decline**,
and `dismiss` is the only thing that stops the meter. That decision is
the content.

### Withdrawal — `body` documents

`BodyRegister` (`/system/ground/idea/BodyRegister`, kind `body`, filed at
`/system/ground/bodies/<locality address>/<body key>`) holds the
per-straw draw and nothing else. Capacity is **never stored**, so no copy
of a record can outlive an edit to the row. ⚠ **Recharge is zero** —
depletion, the degenerate case of a recharge law, and the reason an oil
country is a boom with an end in it.

A body nobody has bored has no document at all.

## ⚠⚠ Depth is BANKED; only the swing is ENGAGED

Engagements do not survive a server restart (the `restart` abort reason
is unbuilt), and a bore is weeks of work — so nothing here may be an
engagement longer than one swing. `swingBank` counts swings toward the
next metre and survives everything; a restart costs at most one swing.

Two things feed the bank:

1. **a present person's own swing** — `bore` is one `engageAct` on
   `hands`;
2. **the crew's presence** — `reconcileRig()` credits the stretch since
   the last sample per hand that is rostered to this hole's outfit, on
   shift, standing in this hole's room and holding no engagement.
   ⭐ *The engine measures presence, not virtue* — and the resolver is
   the **room's own contents**, which is why an NPC resolves here where
   the contract watch's connected-Avatars-only reader never would.

`postRegister` arms an hourly reconcile on the WORLD clock, so weeks pass
with nobody reading.

`cutBill()` is the host hook — the `improvementBill` shape — and prices
the next metre on the host rock's `hardnessMPa`, the field the mining
build documented as *what carve cost is priced on*. ⚠ `null` means *the
ground has not said* and the acts refuse in words; silence treated as
*nothing owed* is the failure mode this repo keeps paying for.

The liner threshold is a **threshold and never a roll**: soft rock under
standing water will not stand open, the bill says so before the swing,
and the refusal names the remedy.

## ⭐⭐ Drilling is EMPLOYING, not collaborative

A bore needs several pairs of hands on a beam for weeks, which looks like
a cooperation mechanic. It is not. What the design wants from the crew is
not conversation or company — **it is a wage bill that accrues while the
hole gets deeper**. The crew is the mechanism by which *depth costs money
over time* rather than *depth costs a click*.

⭐ So NPCs satisfy it completely, and that is the correct answer rather
than a concession. A player who wants to be on a crew `apply`s and
`clock on` like any other job, gets the same wage and counts for the same
swings, because the engine measures who is standing at the rig and never
asks who is behind the eyes. ⚠ That sentence is only true because the
minted outfit advertises places — `headcount` absent or zero means *no
opening is ever advertised*, and for one revision it was zero, which
would have made the help text a lie.

⭐⭐ **The seat's SHAPE is on the seed row; whether it is HIRING is on
the minted outfit.** Two different facts with two different homes, and
`lint:openings` is what drew the line: a seed that advertised a place
was told, correctly, that it *authors no `banksAt` — there is no
operating account for the wage to come out of, the shift settles into a
throw, and the worker is never paid.* Which is exactly true of a seed,
which has no bank, no claim and no proprietor until the siting. So the
seed says what a roustabout's seat IS and the mint says how many places
a real outfit has, because **an opening is a claim about a going
concern.** ⚠ The mint reads the seats off the seed and adds only the
headcount, so the seat is defined once.

⚠ A hired hand is **sent to the rig** (`operatingLocations[0]`, the
`moveForShift` rule). A roustabout has no brain on purpose, so he does
not walk anywhere — hiring at the rig would otherwise be unreachable by
construction. ⭐ Which is also why the roustabout cast stands **at the
flat**: the requirement's own words are *a bore crew is hired at the
bore, out of whoever has walked up to it*, so the hands are already where
the work is and no relocation is needed for the shipped content.

⚠ An unpayable wage follows the shipped rule: an arrear on the book, a
line to a resident proprietor, and the crew keeps working on credit. This
build does not add *the crew downs tools*; the owner finds out at the
bank, which is where a person actually finds out.

## ⭐⭐ A bought act has TWO acts

The owner earns the skill they **used**, never the skill they **bought**.

| act | discipline | who |
|---|---|---|
| siting a hole | `geology` (`hard` when the target is deep) | the owner, who typed it |
| the answer (first bail, wet or dry) | `geology`, `hard`, **`success` either way** | the owner |
| a swing | `mining` | whoever swung — player or hand |
| `measure structure` | `geology` | the reader |
| `measure head` | `physics` | the reader |

⭐ An owner who pays a crew for a month of beam work advances **no labour
discipline at all**. And the *answer* credits success even when the
answer is no: a dry hole is a fact somebody paid for, the judgment was
exercised, and the world answered.

## You file; you do not hold the pen

The bore log is a `bore` document at
`/trade/drilling/bores/<locality address>/<claim leaf>`, titled to the
trade, written only through the gated document surface, and
**append-only**.

⭐⭐ A log is a **sales document**: a buyer reads *what did you cut, how
deep, and what came up*. If its subject could edit it, it would be worth
nothing and the engine would be supplying the pen for the fraud. Real
well logs are filed with a survey or a state geologist for exactly this
reason, and the reason the practice exists is that an operator's own word
about a dry hole is worth nothing.

⚠ `fluid: null` is a **finding**, not a gap. *Nothing at two hundred and
ten metres* is the sentence the payroll bought, and it is worth money to
the next person who looks at that country.

⭐ What the owner MAY keep private is their own field book — the
structural readings in the DISCOVERY realm, which are theirs and always
were. The log is what the hole DID; the field book is what they thought
before they dug it. Only one of those is a public record.

## The verbs

| verb | what it does | afforded by |
|---|---|---|
| `bore` (`drill`) | sites a hole and raises a rig, or swings the beam | the bailer (`environment`), the derrick (`peers`) |
| `bail` | lifts what is standing, and clears the hole so it can go deeper | the same |
| `line` | runs tube down so the hole will hold | the same |

⭐⭐ **Three verbs, and the trade ships NO employment verb.** `hire` and
`dismiss` were this pack's for one build and should never have been:
taking somebody on and letting them go are the employment subsystem's
acts, every house in the realm needs them, and the kernel had the methods
(`OrganizationMixin.appoint` / `.dismiss`) the whole time. `hire` is an
alias on the kernel's `appoint` now and `dismiss` is a kernel verb, so
both are core and need no affordance — which is why the table above is
the three acts that genuinely need a derrick standing over a hole. See
[employment.md](./employment.md) § Appointment and § Dismissal.

⭐⭐ **The derrick is raised by the SITING ACT**, not by `make`.
`CraftingLogic` lands a tangible output at the maker, so a `fixedInPlace`
ton-and-a-half frame would arrive in a pocket — and six lengths of mine
timber is 240 kg, which no body in this game can carry to a hillside.
Raising it on site is what happens in the fiction, and it is what the
trade's own thesis says should be cheap: *the expensive part of a bore is
never the derrick, it is the wages that go down the hole after it.*

⭐ That left a circularity — a derrick cannot be what affords raising a
derrick — closed by the kernel's own blessed split: *`plot` is afforded
by the SPADE, not the field; the two halves of the ladder are afforded by
the two things actually present at each end of it.* So the **bailer**
affords the acts, and it is the one tool on the rig from the first yard
to the last.

⚠ The bailer declares `environment`, not `inventory`. The buckets name
WHO RECEIVES from the declaring object's point of view: `inventory` is
everything nested *inside* the object. A bailer declaring `inventory`
granted `bore` to whatever was inside the bailer, and a player holding
one was told *I don't understand 'bore'*.

## The barrel — `separation: fractions`

⭐⭐⭐ The kernel change Stage C needed, and the distinction is not a
matter of degree.

- A **pot still** separates one substance into GRADES of itself.
  Foreshots, heads, hearts and tails are all new-make spirit; the
  distiller's art is deciding where to cut and then blending what they
  kept. One `productMaterial`, several qualities of it, and recombining
  is legitimate.
- A **refinery column** separates one substance into DIFFERENT
  SUBSTANCES. Gasoline is not a grade of kerosene and no amount of
  blending makes it one. ⚠ **You cannot distil crude and choose not to
  make the light ends** — joint production, and the honest reason a
  player ends up holding something nobody will buy.

So in `fractions` mode every span names its own `material`,
`productMaterial` must be empty, and **`getBulkAvailable` clamps at the
boundary**: the column will not hand you gasoline and kerosene in one
cask. You draw until the character changes and you change casks.

⚠ The pair is validated in `onCreate` and not in a setter, because
`separation` and `fractions` are two authored keys and the applier
dispatches them in whatever order the row lists them. A setter check
would depend on authoring whitespace.

⭐ The per-fraction `requiresHeatK` ladder is exercised here for the
first time: naphtha comes over at 350 K and gas oil does not, so a cold
column gives the light ends and stops — the seam the fractionation
substrate's own header predicted *a refinery column is built on and a pot
still never exercises*.

**Kerosene is the shipped `lamp-oil` material**, not a new row: its
keywords already said kerosene, the lantern burns it, the civic fuel
store counts casks of it by its tag, and the street-lighting bill reads
that store. The fraction's key is `kerosene` and its material is
`lamp-oil`. **Paraffin is a TAG** (`candle-stock`), so chandlery is
untouched and a third wax needed no recipe.

### ⚠ The waste, and the leg that is missing

Gasoline has **no buyer anywhere in the realm** and must not have one
until there is an engine. Two disposal routes ship and need no code:
storing it is a fire risk by the material's own authored fields, and
flaring it burns it for nothing, brightly, where the whole valley can
see. ⛔ **The third — pouring it into a river — is NOT built.** Nothing
shipped turns a poured liquid into a discharge, and building that half is
cross-cutting water substrate a trade build may not solve. The whole
withdrawn shape is on
water-design-pack.

## An oven on a pipe

`Oven.fuelSlot()` answers with the interior of a **sealed vessel of
something that burns standing ON it**. Everything else follows from the
burner substrate with no further code, and the brine hearth's
`placements: [on]` is the one key.

⚠⚠ **Why `Oven` and not `Firebox`:** a `Retort` is a Firebox that is also
a `Placing` host, and what is placed on a retort is its **product** — the
condenser, the gasometer. On `Firebox` the hook would need a guard to
tell a fuel vessel from a product vessel, and **a guard that re-narrows
the host set is the tell that the host is wrong.**

⭐ A DRAINED coupled tank is a fire with **no fuel**, not a fire that
falls back to the bed: silently reverting to the cordwood would make
running out of gas invisible and the bed's depletion inexplicable. The
remedy is to take the bladder off, and the refusal says so.

## The falsifiable line

A second well town needs **zero pack code**: its own `fluids:` on its own
column, its own sites and prose, and an import. `rejection` ships four
bore sites, four `surfaceWorkings` entries, three fluid bodies and three
surface showings, and **no TypeScript at all.**

⭐ A fourth kind of surface showing is a row: the structural channel's eye
rung knows nothing about springs, gas blows or oil seeps — it asks the
room what is `showing` and repeats the prose.

## ⚠⚠ Known-soft: the crew's RATE

Measured live, 2026-10-09: **two roustabouts on shift, a full game day,
one metre.** Against `CREW_SWINGS_PER_HOUR = 120` × 2 hands × 24 h and
`SWINGS_PER_METRE_REF = 6` that is an order of magnitude light, and
lining is **not** the gate — Rejection's deposit authors no `waterTable`,
so the default −45 puts the liner threshold at 45 m.

⭐ The suspect is `SAMPLE_CAP_S`: `reconcileRig` clamps `elapsed` to one
hour, so a clock **jump** is credited once rather than replayed hour by
hour. The cap is the right shape — it is what stops an unobserved rig
minting unbounded depth, and it follows *depth is BANKED, only the swing
is ENGAGED* — but it makes this method's own *"weeks pass with nobody
reading"* only partly true.

⚠ Untuned on purpose: which number is wrong is a **balance** question
against a running game. The drive asserts the mechanism and no rate.
Full write-up and the other deferrals:
drilling-slate § Tail.

## ⛔ And nothing CAPS a well

An uncapped well leaks and there is no act that stops it — not even a
refusal. Recorded as the polity's to price rather than as a mechanism;
see the slate's tail.

## Cross-references

- [ground.md](./ground.md) — the column, and the `fluids` field
- [mining.md](./mining.md) — the lode, the hardness, the `surveying` dial
- [instrumentation.md](./instrumentation.md) — the reading ladder
- [fractionation.md](./fractionation.md) — the cut, and now the column
- [bulk.md](./bulk.md) — the four policy seams, and the gas that escapes
- [fire.md](./fire.md) — the burner, and the coupled fuel vessel
- [employment.md](./employment.md) — the roster, the wage, and ⭐ the KERNEL's `appoint` (alias `hire`) / `dismiss`, which this trade deliberately ships none of its own version of
- [activity.md](./activity.md) — why a bore may not be an engagement

## The pump build (2026-10-09) — the attach point, consumed

The wellhead's `liftL()` *"is the pump build's entire attach point"* — and
the pump attached there. `Wellhead` answers `LiftSource` (the hole's depth,
the body's fluid, the sump) and `PumpSource` (one pump, set in it by `put`,
vetoing everything else); it affords `pump`. `liftInto` is bounded by the
sump and the trough; `lift()` — the bailer — is unchanged and asks no
pressure: a bucket has no ceiling at any depth. A crew at a fitted pump
earns a continuous rate in `reconcileRig`, only where the pump's own law
lets it lift. `bodyKey` is **authorable** now: Rejection ships an old brine
bore at the spring (25 m, the salt leg's rim) — a venue asserting a history.
See [pump.md](./pump.md).
