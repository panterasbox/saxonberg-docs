# Ranching

**Keeping animals.** The ground they stand on is
[soil.md](./soil.md)'s — *pasture is a field* — so `trade-ranching`
depends on `trade-farming`; what lives here is the keeping: the herdbook,
the boundary acts, handling, the taps, and what happens at the end.

---

## Where it is proved — the campus farm

`packages/content/eternal-university/content/world/eternal/campus-farm.yaml`
(D103) is **small, real, and working**: modelled on Davis and Cal Poly,
which teach agriculture with an actual production unit on campus — small
enough to walk, real enough to sell; students do the work; the unit is a
teaching facility and a business at once. ⭐ It is the mechanism/expression
cut's own falsifiable test (AC 62): the farm is authored against the farm
and byre archetypes with **zero pack code** — every file under
`campus-farm/` is YAML, and the classes it names are `trade-farming`'s and
`trade-ranching`'s. The campus field site teaches the loop; the world
charges for it. (Graduated from the campus-grounds slate, 2026-09.)

## ⭐ The individual is the base case (D19)

The ranching slate's stance — *"a rancher does not win over a cow"*,
livestock are fungible and managed at scale — is true of a 500-head
operation and false of six goats on a quarter acre, which is the land
this game actually has.

**Pets settles it:** there is never a herd of pets, so if the herd were
the base case, pets would be a special case of it, and it obviously is
not. This is the individual, with a **compression** applied to the
animals you have stopped looking at.

⚠⚠ **A herd is NOT a glob.** A glob's members are *identical* and share
one state; a herd's are *unindividuated* and their states **diverge** —
not a weaker version of the same thing, the opposite thing. The
management game is about the tail (the three thin ones, the lame one, the
barren cow), never the mean.

---

## ⭐⭐ The herdbook — you file; you do not hold the pen (D20, D79, P4)

> ⓘ The herdbook is a record filed elsewhere because a herd MOVES. A
> stand of trees does not, so forestry's stand is NOT the herdbook's
> fourth consumer — it is `Field`'s second: a cover carried by the
> place ([forestry.md](./forestry.md)).

A herd is a **record**, not an object in a room: *these head, this
composition, this age structure, on this ground*. There is no herd-object
to `look` at, and there never will be.

It lives at **`/trade/ranching/herds/<…>`** — a branch titled to the
ranching trade's own group, **not to the animals' owner** — and that is
the security requirement rather than a filing convenience:

> **A record about you lives on a branch titled to somebody else,
> because its value depends on its subject not being able to edit it.**

D79 makes the herdbook a **sales document**: records exist so a buyer can
trust a claim about an animal they did not watch grow. A record its
subject can rewrite is the lemons fraud with the engine supplying the
pen.

⭐ It is historically exact. Real herdbooks are kept by **breed
societies** — Coates's Herd Book, 1822, worked because it was independent
of the men selling the bulls — and three things fall out free: custody of
the record is separate from ownership of the herd; transfer is a registry
act rather than a file edit; and **filing is a gated act**, so *who may
register a herd* becomes a live question the polity can answer.

⚠ **Two homes are refused, for one reason.** `/home/<self>` gives an
owner their whole home branch with no broader grant; the owning **parcel**
is the same hole one step out. This bug class was closed once already —
land use lives in the gated `parcels` collection rather than on the zone
template, because *"a content author could rezone their own land"* — and
must not be reopened.

### ⚠⚠ Read-side verification is mandatory

The document store is shared and the kind tag is **forgeable**:
`document-store.md` is explicit that every read must re-verify what the
transport guarantees. **Every herd read checks the path sits under the
registry prefix**, or somebody writes `kind: 'herd'` on their own home
branch and it counts. The separator is part of the check —
`/trade/ranching/herdsX/…` is not under `/trade/ranching/herds`.

### The write needed a register transport

`DocumentApi.save`'s gate admits the **branch owner**, which here is the
trade — so a keeper drafting a head out could not write, and granting
them the branch would hand them the pen. The herdbook writes through
`DocumentApi.saveToRegister` instead: it takes **no owner and no kind**,
because a `Registrar` declares its own branch, owner and kind and the
transport derives all three.

> ⚠⚠ The first cut was a pinned `saveHerd` with `/trade/ranching/herds`,
> `/trade/ranching` and a `FromTemplate` gate naming this class **as
> kernel constants** — a pack's namespace hardcoded in the engine, and
> not something that survives the second register. It was replaced
> during the pre-merge review.

⭐ What keeps it narrow is an invariant rather than an allowlist: **the
owner must be a prefix of the register's own template path, and its
branch must lie under that owner** — so a register keeps its own book and
nobody else's, and the kernel never learns which societies exist. See
[document-store.md § The register transport](./document-store.md).

### ⚠ Two sources, deliberately

The document holds **composition, ownership and claimed home**;
containment holds **position**. Their disagreement *is* straying (D95) —
derivable on read, needing no new event, and the reason a herd has a
jurisdictional anchor at all.

### Open — `herdId` should SEED itself

⭐ Left by the hydration build (2026-10-02). A `Herdbook`'s `herdId` is
filed imperatively today; it is a textbook **`seed`** field —
`fieldMeta`'s `seed: true` plus a `seedHerdId` applier, run in the
applier's phase 3 at mint only. Now a **pack-only** change: since
`hydratorClass` retired there is no kernel row to name and no second
applier to subclass. See
[templates.md § The three phases](./templates.md).

---

## ⭐⭐ Draft and return (D21)

Head *n* is a deterministic function of the herd's identity and its index
— the field pattern's fourth instance, *seeded never drawn* — so **head
17 drafted twice is the same animal**, and the answer was true before
anyone asked.

> **Identity is earned by being measured** stops being a metaphor here
> and becomes the implementation.

⚠ **The asymmetry is honest.** Drafting mints an object; returning
**destructs** one and folds what became of it into a sparse overlay,
because *its identity was the record, not the flesh*. Draft it again
tomorrow and you get the same animal, a day older and exactly as thin as
you left it.

⚠ The stockman's own word is what moves it. Nothing promotes a head
automatically and there is no "this one is interesting" hint — which is
what keeps the management game about the tail.

---

## Handling (D27, D46)

A cow must not have pet-love, but without *something* an individual is
per-head bookkeeping. **Handling** — temperament, flight zone, ease of
working — is real animal husbandry, earned by contact and lost by
neglect, with a **diminishing return** so the first session is cheap and
the twentieth is not.

⭐⭐ **And it is a SAFETY mechanic before it is an efficiency one.** Quiet
stock handling exists in the real world because flighty animals injure
people; cattle are the most dangerous thing on a farm. `handlingRisk()`
is the complement of tractability **squared**, so the whole quiet end is
safe and the wild end is sharp — which is how handling injuries actually
distribute, and why this is a reason to handle stock properly rather than
a tax on doing so.

The energy of a blow is the **animal's mass**: a hen cannot hurt you and
a cow can break your ribs against a gate, with no species table.

⚠ It is kernel substrate (`lib/husbandry/Handling.ts`) because **pets
will want it and pets is not ranching** — its composers share no pack
ancestor.

### ⭐⭐ `handle` — precision costs an act (D24)

One act does two things and that is the design: it **works** the animal
(raising handling) and it gives a **precise body-condition score**,
because real body condition scoring *is* palpation of spine and ribs. By
eye you get a band; with your hands you get a number. The person who
handles their stock is the person who knows what condition it is in, and
neither is bought separately.

---

## Condition — `flesh` (D24, P7)

**Body condition is fat cover, which is a stock.** It is a fourth
biological reserve on every living body, and it resolves D24's apparent
tension: the **reserve is stored; the band is derived**.

> **`satiation` is hours; `flesh` is months.**

The partition leg runs **last** in the metabolic slice, and that ordering
is the mechanism: intake is spent maintenance → thermoregulation →
growth → production → reproduction, and only the surplus can be banked.
**Production dies before condition does** with no special case.

⚠⚠ It does **not** cushion starvation. A body with fat on it still
starves in a day of nothing; `emaciation` is non-lethal and never reaches
the death seam on its own. What months of shortfall buy is worse and
free: `VitalsMixin.getConditionBand` already sums floored biological
reserves, so a wasted body is a degraded body **with no new wiring**.

⚠⚠ **The far-past guard is narrowed to bodies nobody owns.** It exists
because real-life absence must never starve YOU; a kept animal inheriting
it would gain nothing across any absence longer than lunch, and the
winter-feed budget would never bite. Read through the chattel stamp,
synchronously, on the reconcile path.

⭐ **Why a cascade, and not farming's limiting factor.** Farming's lesson is
Liebig's minimum — independent inputs, yield set by the scarcest — and a
ranching that copied it would be farming with legs. Ranching's is one shared
budget spent in a fixed priority order, so the consequences a designer would
otherwise write in fall out of the ordering: underfeed and production dies
first, then growth, then condition, then the animal; a cold snap raises
maintenance (`ThermalRegulation` spends satiation to hold the setpoint), so
milk drops at constant feed and shelter is load-bearing rather than
decorative. Two optimisation idioms on one reconcile engine — a player who
learns both has learned two real things, not one thing twice. (Graduated
from the ranching slate, 2026-09-19.)

⚠ **The cascade has no INPUT yet** (verified 2026-09-19). Nothing grazes:
`Field.swardGrazingDemandPerGameDay()` sums `grazingDemandPerGameDay()` over
the field's occupants and **no class declares that method** — `Livestock`
included. Nothing feeds a head: the platform `feed` verb requires a
`CultivableMixin` target (compost into a bed), and `Livestock` composes no
`BehavedMixin`, so the `feeds` / `eats` brains never run on it. The
partition leg therefore runs on whatever satiation a drafted head carries,
the winter-feed budget above cannot yet bite, and *"a herd eats whether or
not it produces"* is not yet true of this game. The feed loop is
ranching-slate's first `Left` item.

---

## Ownership, and the two one-liners (D22, D98)

`ChattelMixin` was composed in exactly one place — `lib/stuff/Thing.ts` —
and `Creature` descends from `Agent`, so **nothing alive was ownable**.
One composition line gives livestock, pets and future aquaculture
per-instance ownership with chain-of-title, and retires the pets slate's
sketched `CompanionMixin`.

⭐ Chain-of-title is also **rustling's answer**: a stolen animal keeps its
provenance and cannot be sold cleanly, so fencing is the hard part
exactly as in life.

⚠⚠ **Branding did NOT reach the stack by the same move, and the attempt
was a defect.** `BrandedMixin` was Things-only and the argument for
moving it was the right one — *branding livestock is what marks were
invented for* — but it was composed on `Creature`, which is the base of
`Character`. That put a maker's mark on every player Avatar, every Cast
member, every Extra, every ShadeAvatar and every corpse in the game. Nothing
on that stack ever read it, so nothing failed; the one surface that
would have shown it, `wiki branded`'s *composed by* panel, was reading a
scan root that has never held a row.

The base-class narrowing build moved it to the two hosts the argument
points at: the kernel's `KeptAnimal` (so a pet, a canary, a
`WorkingAnimal` and a fishmonger's tank fish can be marked) and
`Livestock` itself. ⭐ **`Livestock` composes it directly**, because it
extends `Creature` and not `KeptAnimal` — a head of stock is not a pet —
exactly as it already composes `HandlingMixin`. Chattel stays where it
is: *ownable* and *marked* are different questions, and only the second
one is false of a person.

---

## The taps (D25, D93) — ✅ **PROMOTED; the mechanism is [taps.md](./taps.md)**

> ⭐⭐⭐ **A tap is not a livestock concept, and as of the taps build
> (2026-10-01) the code says so too.** `ProducingMixin` is the KERNEL's
> now — `lib/husbandry/Producing.ts`, beside `Handling` — because its
> composers have no common pack ancestor: a cow (here), a hive
> (apiculture) and a sap-bearing tree (forestry). `TapSpec` was already
> kernel, on `Species.production[]`.
>
> ⭐ **The window predicate this doc asked for exists.**
> `TapSpec.window` is declared data (`always · event · photoperiod ·
> biome · weather`), so a new season is a row — and the idiom is this
> doc's own: *a photoperiod season, not a date, and the refusal names
> the reason.* ⚠ Milk's window is `event` (a lactation is not a season);
> the hen's is `photoperiod`; the hive's is `biome`; sap's is `weather`.
>
> **The mechanism, the act, the window kinds and the host-placement
> table now live in [taps.md](./taps.md).** What stays here is what is
> true of RANCHING's three.

⚠⚠ **A tap fills from the production slice of the energy budget and
mints nothing.** Copy `Stock`'s reset *sweep*; never its `par`
semantics, which is a faucet wearing a hat. The rate scales with
`flesh`: an animal in poor flesh gives less because it has less to give.

### ⭐⭐⭐ The feedback law — and two of the three rows changed

`behaviour` says what NEGLECT costs. What decides how each product PLAYS
is whether the act of taking feeds back on the rate, and asking that
question reversed two of these three:

| | behaviour | neglect | the judgment at the act |
|---|---|---|---|
| **milk** | `expire` | she **dries off** for that lactation — a large **slope**, not a cliff | ⭐ **none, and that is the finding.** Lactation is demand-driven, so a take always empties her and holding some back would suppress her rather than save it. What the player trades is **attendance against her rate** — labour |
| **eggs** | `accrue` | ⚠ **REVERSED.** Not *they spoil in the nest* — a clean unwashed egg keeps for weeks. A hen is an **indeterminate layer**: a full clutch left sitting makes her **broody**, and she stops laying | ⭐ take the clutch and she starts again, or leave it and she stops. One state flag, no invented punishment |
| **wool** | `continuous` | a worse fleece, and a hot sheep | none at the act — but ⭐ the year is **recorded** now (`TapState.worst`), carried away by the take and stamped onto the fleece's grade band. A ewe who went hungry in February carries the break in June, and feeding her up afterwards does not heal it |

⭐⭐ **Accrual for the on-ramp, expiry for the committed.** The
forgiving end of the roster accrues and expiry is what you take on when
you commit — which is why hens are the on-ramp and a dairy cow is a
tyrant, and why **a player's real-life cadence honestly decides what
they can keep**.

### The act, and what changed at the verbs

Every take is an **engagement** now (`TapActController`): observable
time on the `hands` slot, abortable with `stop`, and ⚠ walk away and
*nothing came of it* with no state change.

- **`milk <cow> [into <vessel>]`** — a `volume` tap, so it REQUIRES
  something to catch it in and the refusal names the pail. ⚠ The *"pail
  of milk"* Provision is **deleted**: minting the pail along with the
  milk was a faucet shape for containers. What the pail will not hold
  goes on the straw, and that is the one line in the build that teaches
  what a vessel is for.
- **`gather <hen>`** — a `count` tap: it mints EGGS, not a kilo of egg,
  which is what lets a recipe ask for two (and is half of why eggs had
  no sink for their whole shipped life — the other half was the egg
  material carrying no `egg` tag).
- **`shear <ewe>`** — the fleece is `/trade/textiles/thing/TextileStock`
  now: ranching's ROW, textiles' CLASS. That is what made `spin fleece`
  reachable at all; see [textiles.md](./textiles.md).

### ⭐ What a `look` says

`Livestock.markupAugmenters` appends `productionRead()` — in words, with
**no digit in any of them**: *She is in full milk · She is heavy and
wants milking · She is overdue, and will dry off for the season if
nobody comes · She has dried off · She is sitting tight on a clutch and
has stopped laying · A lean spell has left a weak point in the fleece ·
The fleece is so heavy it is starting to shed.*

⭐⭐ That is how *going off is visible before it is lost* is met, and it
holds **no state**: the milk bands are read off the window clock, which
is the one number milk was ever about. ⚠ `stockmanRead()` stays where it
is — that answers *what is this animal worth* (draft/return's question),
not *what is it doing*.

### ⛔ Two mechanisms specified and deleted

Recorded because both came from the plan rather than the requirements,
and the test suite is what caught both:

- **`TapState.vigour`** (a milk suppression curve) is
  **unimplementable**: `ceiling = perGameDay × windowDays`, so she fills
  to her ceiling exactly as her window closes and the region where she
  sits full and suppresses herself is *empty by construction*.
- **`shear --quick`** (speed for quality) is a judgment the biology does
  not put at the act — the feedback law's third row says so.

### ⚠ Still open: the relief

Milk's answer to attendance is a **standing instruction** (`instruct
keep <line>`, a brain on the player's own body) and it **did not ship**:
its host is build-3's `lib/character/Avatar` and MR !315 had not merged.
AC 7 moves to the dairy build with the rest of milk's story — and the
argument that this is survivable is that **nothing consumes milk yet**,
so nobody is actually dairying and the harshness is theoretical.

---

## Breeding (D26) and slaughter (D28)

**Breeding is a photoperiod SEASON, not a date.** A ewe's window is
stated in daylength and the calendar decides when that happens, so
**lambing in spring is a consequence of the calendar** rather than a
flavour decision. The refusal names the daylength, because *"the days are
still too long, she will not take"* is something a player can act on.
Nobody authors a lambing date, and that is the whole idea.

### ⚠⚠ What `breed` does: it writes SERVED

The herdbook's ruled columns are *number, dam, born, served, **calved***.
`breed` writes the **fourth** — she was put to the male, in season, and
the book records the date. **The tally does not move**, because nothing
has gestated.

> ⚠ The first cut incremented the tally the instant the verb was typed,
> while telling the player *"it will be 145 days"* — so gestation was
> **announced and not modelled**, the offspring were born adult (every
> head read as a flat 400 days old forever), and the herd was an
> unbounded faucet for the length of a season. It also claimed the
> offspring's character was *"seeded from its parentage, which gives
> selection real traction"*, and that was **false**: the new head was
> `hash(herdId#index)` and the parentage was a free-text note nothing
> read. Breeding the best ewe and the worst gave statistically identical
> lambs.

⭐ **Ages DERIVE, in two branches, and the order is the design.** A head
with `bornAt` in the overlay is as old as the time since; otherwise it is
founding stock, and its age is the herd's own `foundingMeanAgeDays`
**plus the game time elapsed since `founded`**. The herd gets older the
way everything else does. ⚠ Nothing writes `bornAt` yet — it is read
first precisely so that the day gestation lands, a lamb is a lamb with no
special case at the draft.

**What is NOT built, and belongs to the breeding follow-on:** gestation
(the wait between *served* and *calved*), birth, and heredity of any
kind. The design for all three is in
ranching-slate § Breeding — the
short version being that the animal breeder's equation is
`R = h²·S / L`, and it is the **generation interval `L`** that makes
animal breeding a different lesson from farming's.

**Slaughter is sober and complete.** No minigame, no guilt meter, no
confirmation ritual: **make waste the thing that feels bad, not the
killing**, and let the density dial do the rest — a number in the
herdbook is easy to cull and an animal you named is not. What comes off
scales with condition, so *finish it before you kill it* is something a
player works out. ⭐ Bone goes back **into the soil** as phosphorus, so
the animal that ate the field feeds it back at both ends.

⚠ The hide is a **stated seam**: tanning wants tannin, which is the
dyeing trade's. A hide is a real thing that stores and sells and is
already waiting. What must not happen is a `leather` row minted out of
nothing.

---

## The roster is a commitment ladder (D30, D92)

Each species earns its place by unmet demand **and** by sitting at a
distinct point on the real-life attention scale:

| | demand | why it is here |
|---|---|---|
| **hens** | forgiving | ⭐⭐ the on-ramp, and laying stops in short days — photoperiod teaches itself on day one |
| **sheep** | seasonal | textiles' sourceless wool; a short-day breeder |
| **cattle** | a dairy cow is a tyrant | the cooking chain's protein and tallow; near-aseasonal, so the roster teaches by CONTRAST |
| **pigs** | moderate | the waste converter — `spent-grain` ships in two packs with nowhere to go |
| **bees** | a look every real week or two | ✅ built — [apiculture.md](./apiculture.md); the colony is the organism |

---

## Working animals (D40–D42)

**Draught power is body mass; there is no new mechanism.** A person is
~70 kg and an ox is 700, so an ox is worth about ten of you and **that
ratio is authored nowhere**. By hand ploughing is punishing; with an ox
it is work — which makes the ox a genuine capital investment. ⚠ And an ox
eats whether or not it works.

⭐⭐ **The dog is the fourth rung of the automation ladder** (attention →
wages → compute) **and the only one that costs a relationship.** It does
real economic work, so a poorly bonded dog works badly — which gives the
bond an economic consequence **without giving any livestock a bond
stat**. Three jobs: it herds, it guards (a dog in the yard is why the fox
does not come), and it keeps deer off the crop.

⚠ It cannot do the acts that need judgement about THIS animal — the
draft, the cull, the paddock move. That is the batchable test's other
side.

### ⭐⭐ Three ROLES, not three kinds of object

The collie shipped as a `Livestock` and is not one. The tell was already
in the code: **five of the seven verbs `Livestock` afforded re-narrowed
their target at execute time** — *"that is not an animal that gives
anything"*, *"that is not an animal you can breed"* — because the class
promised a verb set its own instances did not uniformly satisfy. ⚠ And
`butcher`'s guard was *"does it have a handling score"*, which a sheepdog
does, so the sheepdog was butcherable.

> **A guard that re-narrows the host set is the tell that the affordance
> is on the wrong host.** The question a controller asks late is the
> question the affordance should have asked early.

| | kept for | the axis |
|---|---|---|
| **livestock** | what it yields or becomes | the **taps**, and the carcass |
| **working animal** | what it does | its **skill** |
| **pet** | itself | the **bond** (pets-slate) |

⚠⚠ **They overlap, which is why this is not a taxonomy.** The ox is
livestock *and* a working animal — it ploughs all its life and is beef at
the end — so it stays a `Livestock` and keeps `butcher`, and `plough`
reads its **mass** rather than its class. A collie is a working animal
and, in most households, also a pet. A prize bull is livestock and
breeding stock. Exclusive classes would need a fourth one immediately.

What makes the roles separable is that **the verbs live on capabilities**:

| verb | afforded by | because |
|---|---|---|
| `milk` `shear` `gather` | `ProducingMixin` | it has taps |
| `handle` | `HandledMixin` | it has a temper you can find out |
| `return` `breed` `butcher` | `Livestock` | it needs a herd behind it or a carcass in front |

So `WorkingAnimal` adds nothing but a name and an **absence**, and the
absence is the point.

⚠ Since the pets build (MR !257) the collie is `HandledMixin(KeptAnimal)` —
it gained the bond, a name, a home and `BehavedMixin` (its `herds` brain had
never run before that) — see [pets.md](./pets.md). The absence that remains
is the taps and the carcass.

### ⚠⚠ What the carcass opens onto — two silent failures, both found live

`butcher` announces *"nothing is wasted"* and lists what came off. Both
halves of that sentence were false for a while, and neither threw.

1. **The beast weighed nothing.** Yields are fractions of live mass, and
   a drafted head's mass seeds from its species — which authored none,
   over a `quadruped` body plan which authored none either. Every cut
   rounded to zero and the act reported the zero-yield line, *"there was
   less on it than you hoped"*, which is a true statement about a body
   that massed 0 kg. Fixed by `Species.adultMass` (see
   [race.md](./race.md)); a cow is 550 kg and dresses accordingly.
2. **The meat line named a MATERIAL, not a row.** It pointed at
   `/stuff/idea/material/food/stew-meat` — what a cut is made *of* —
   so the clone threw, `mint`'s deliberate catch swallowed it, and the
   carcass came apart into tallow, hide and bone with **no meat at all**,
   reported as success. It is `/stuff/thing/items/stew-meat` now: the
   shipped `Provision`, so butchered meat arrives carrying Freshness,
   Cured and Contaminable and the clock the cooking chain starts at the
   kill is the same clock ([spoilage.md](./spoilage.md)).

⭐ The catch in `mint` stays — a missing cut row genuinely should not
lose the rest of the carcass — so the guard is a test that walks the
yield table (`carcass-rows.test.ts`), because `lint:census` reads
path-valued fields in shipped ROWS and these paths live in TypeScript.

⚠ Both were invisible to the suite for the same reason: a fixture hands
the mass and the row straight in. The first one masked the second.

⚠ `HandledMixin` is the affordance half only — the kernel owns the
mechanism (`lib/husbandry/Handling`) and cannot name a pack's command
view. It does **not** compose `HandlingMixin`: nesting a factory inside a
factory collapses TypeScript's inference through the chain, so the two go
side by side at the call site.

⚠⚠ And it carries `static _mixinName`, which is load-bearing and whose
absence failed **silently in one direction only**: affordances are
collected off a class's own static plus every *registered* mixin, so
`WorkingAnimal` (no own static) inherited `handle` and looked correct
while `Livestock` (own static) shadowed it and lost the verb. One host
working is exactly how this hides.

⭐ **The skill axis is not built.** A working animal is an animal with a
**transcript** ([advancement.md](./advancement.md)) — which is what
separates *training* from *taming* — and `pets-slate` already assumes a
dog holds competence bands in retrieval or guarding, with `stockmanship`
shipped as a Discipline. That is a follow-on, and it belongs beside pets
rather than inside ranching.

---

## Hazard (D45–D52)

⚠⚠ **D45 scopes all of it.** What accrues in your **absence** is a slope;
what happens in your **presence** may be a cliff; **weather is neither**,
because it is not a judgement on you.

- ⭐⭐ **The hay fire** — hay put up wet heats until the barn burns, weeks
  after a mistake that was invisible at the time. No roll anywhere in it:
  microbes respire, a big stack cannot shed the heat, and above ~55 °C
  the chemistry runs away. The moisture is stamped at **baling** and
  never re-read, which is exactly why *"it seemed dry enough"* is the
  sentence that burns barns.
- ⭐⭐ **The slurry pit** — and it needed **no new medium**: `stinkdamp`
  (hydrogen sulfide) already ships as the mine's bad air, so a slurry pit
  and a mine sump are the same fact about the world. The rescue trap is
  not scripted: one person goes down and the people who go in after them
  die too, because that is what an unbreathable medium does. ⚠ The one
  hazard that punishes courage rather than negligence.
- ⭐⭐ **The fox** kills the lot and carries one off — a sudden total loss
  in one night, with a real defence (a dog stops it outright, no roll)
  and a **commons** dimension: predators range across parcels, so
  abatement is a hired job on the shipped work-contract substrate (D64).
- **Handling injury** — see above.

---

## ✅ Bees: BUILT — see [apiculture.md](./apiculture.md)

The colony is *the herdbook with the individual end amputated* — you never
draft a bee — and it was designed here precisely because the pollination
coupling would have been painful to retrofit. It was severable by
construction, was marked cuttable at its decision, and was cut when this
build ran long.

⭐ **It shipped 2026-09-30 as the `trade-apiculture` pack**, and everything
this section listed as missing is there: the hive (a `Vessel`, for the
winter arithmetic), pollination pushing onto a latched set count, the
forage range as a walk over EXITS priced in minutes, honey's character
riding the comb's composition, and swarming on filled-its-room-in-a-flow.

What ranching keeps, and what apiculture reaches back for:

- the **taps** (`ProducingMixin`) — a hive has a `honey` tap like a cow
  has a `milk` one, and `rob` is a `TapController` subclass. ⚠ The three
  tap views are on `Livestock` rather than on the mixin for this reason:
  a hive composes `ProducingMixin` and must not be promised `milk`.
  `TapController.discipline()` is the hook that lets the fourth product
  credit another trade.
- ⭐⭐ **`HandledMixin.workedOver(actor)`** — the BODY of `handle` lives on
  the animal now, and its default is this trade's: the rail-slam hazard,
  the flesh score out of a hundred, the hand down the spine. A hive
  overrides it with bands, because a colony has no spine to read and no
  number to give you. `lint:verb-collisions` refuses a second `handle`
  view, correctly, so there was never a second-verb way out.

---

## History — the carcass chain and the butchery build (2026-10-05)

⚠⚠ **Two claims above are now false, and the sections that make them are
kept because the reasoning in them is why the change happened.**

**1. `butcher` is not this trade's verb any more.** The yield table
(`YIELDS`, five hardcoded fractions) and the `ButcherController` that
destructed the animal and minted five goods are **retired**. There is
**one death path**: `ConditionApi.die` mints a `Corpse` stamped with
`_speciesPath`, `causeOfDeath` and `diedAtGameSec`, and `slaughter
<animal>` calls it exactly like a fight, a fall or starvation does, then
writes the herdbook. Breaking the body down is `trade-cooking`'s
`butcher <carcass>`, because **butchery specializes cooking**. Three
things fall out of it that no amount of guarding the old verb could buy:
a beast killed in a fight is butcherable on the same path as one you
slaughtered; the recorded `butcher` collision between two trades
(*"undiagnosed"* in `KNOWN_COLLISIONS`) is retired rather than
documented; and the *"guard that re-narrows the host set"* argument above
reaches its conclusion — the affordance moved to the object the act is
actually performed on, which is a body, not an animal.

**2. The yield is the SPECIES', not this pack's.** `Species.butcheryYield`
is kernel and authored per species, so a ewe and a cow no longer share
five fractions. What survives from here is the **condition scaling** —
`finish` off the `flesh` reserve — as a multiplier *on* the species
yield, which is the half of the old table that was a real idea. ⭐ And
the two silent failures recorded in *"What the carcass opens onto"* are
structurally unrepeatable now rather than fixed: a cut is a row claiming
tissues the body plan carries, and `lint:anatomy` refuses a species whose
yield claims a tissue its plan does not have.

⭐ **`slaughter` is unchanged in spirit and that matters**: sober, no
minigame, no guilt meter, waste rather than killing being the thing that
feels bad. It is the same act with the join moved one object along.

⚠ **The hide's "stated seam" above is CLOSED.** Tanning exists —
`trade-tanning`, a pack of its own, with a tanpit whose liquor the hide
reconciles against. Bark is a fourth `fell` yield declared per species
(oak tans, birch does not), which is where the tannin comes from. See
[butchery.md](./butchery.md) and
[forestry.md](./forestry.md).

⚠ **Bone is organic matter, not phosphorus.** The sentence above says
phosphorus and the soil has no such reserve; `bone-meal` carries
`slow-amendment`, which credits organic matter — see
[soil.md](./soil.md). The fiction ("the animal that ate the field feeds
it back at both ends") is intact; the nutrient named was wrong.

Full model: [butchery.md](./butchery.md).
