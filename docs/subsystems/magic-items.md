# Magic items

The build that makes **using a thing** a first-class way to act on the
world, alongside casting.

Before it, the engine had exactly one trigger for an effect — a caster
spending mana — and no abstraction for a discrete item that fires one.
This subsystem ships the trigger-agnostic effect spine, the three item
classes that hang off it, the charge economy that keeps non-consumables
from inflating, the blessed/uncursed/cursed axis, the identification
substrate, the spellbook/memory loop, and the distribution channels.

Governed throughout by [arcane-science.md](../arcane-science.md): **an
item is not a new physics, it is stored labour** — a maker paid earlier,
and the user spends it.

---

## The effect context — one parameter was doing four jobs

The shipped pipeline threaded a single `caster` through every executor,
and it was quietly serving four different purposes:

| Job | Read as | Where it showed |
|---|---|---|
| **origin** | where the working comes from | the reachability check |
| **actor** | who is acting | `target ?? caster`, the harm row |
| **source** | what supplies the energy | the reserve spend |
| **potency** | how hard it lands | already a parameter |

When a person casts, the first three are the same object, which is why
one parameter sufficed. **A wand pulls them apart**: the wand is the
origin, the user is the actor. A wand set down and pointed at a door is
a trap — and that sentence only means anything if origin is separable
from actor.

`lib/magic/EffectContext.ts` separates them. Two construction shapes:
`forCast` collapses origin/actor/source onto one object (so casting is
unchanged by construction), `forItem` pulls them apart.

Three consequences that would otherwise be silent bugs:

- **A self-effect targets `ctx.actor`.** Reading `origin` would cloak
  the wand instead of the person.
- **The accountability row names `ctx.actor`.** A wand must not be able
  to initiate a harm row; responsibility does not launder through an
  object.
- **`ctx.source` is who PAYS — and therefore who takes the recoil.**
  Not a convention: the momentum came from the energy.

### Provenance carries two ids

`MagicProvenance` splits `caster` into **`specifiedBy`** (the maker) and
**`firedBy`** (the user). Dispel and arcane-sight key off the tag;
accountability keys off who acted; and *who made this wand* is the item
economy's whole quality signal.

This is **persisted state** on three condition kinds, so
`MagicGrid.normalizeProvenance` reads a legacy `caster` as both ids and
`Vitals.setConditions` is the hydrate seam that applies it. There is
deliberately no live `caster` alias — that would recreate the ambiguity
the split exists to kill.

### The casting profile

`requiredBand` and `castSeconds` are the **caster-assuming** half of a
spell, so they live on a `CastingProfile`. An item **ignores the profile
wholesale** rather than silently ignoring two fields that happened not to
apply. That is why a novice can fire a wand of a working they could
never cast: the maker passed the gate at manufacture.

### `Arcane` — one shared grid footprint

Four consumers need to know where a magic-producing thing sits on the
verb × noun grid: **suppression**, **dispel/detect**, **rarity**, and the
**census**. `ArcaneMixin` is the one declaration all four read.

It sits **below distribution** on purpose. If the footprint lived on
`Circulating`, a ward would have to consult the distribution subsystem
to know whether to suppress a wand. It is named for the *property*, not
the object kind, so traps and NPC powers can wear it later with no
rename.

**A set of typed `{verb, noun}`, never a scalar** — and that is forced,
not preferred: the shipped code reads the two axes independently in all
three places (`MagicSuppression` matches either; competence is two
Discipline leaves; Tarn's Rule takes the minimum). **Two sources, one
accessor**: an item carrying a `spellId` *derives* its footprint and
never copies it; a bespoke item *declares*.

---

## The three item classes

| Class | Supplies | Endpoint | Bounded by |
|---|---|---|---|
| **Charged** — wand, orb, ring | energy **+** specification | the item | energy density |
| ~~**Focus**~~ — *cut; see below* | — | — | — |
| **Consumable** — potion, scroll | one packaged act | varies | one use |

The endpoint column is `ctx.source`. A charged item's recoil lands on
*the item*, which is what makes a spark wand safer than the equivalent
cast — the wand is in the circuit and the user is not.

> ⚠ **`Focus` was cut before merge.** It shipped a fade clock topped up
> through the *same* `recharge` verb a wand uses, so player-facing "my
> wand is low" and "my rod's pattern is fading" were the same sentence —
> a second instance of an existing decision rather than a new one. It
> also shipped with **no verb to fire it**, unnoticed for a whole build,
> because nothing pulled on it. And it was the one class with no NetHack
> analogue (wand / scroll / potion / spellbook all map one-to-one),
> which is why nobody had intuitions about it.
>
> The intuition worth keeping — **mages should have gear** — is a
> different mechanic: an item that *modifies what happens when you
> cast*, rather than one that casts using your mana. See
> implements-slate.md.

Consequences that are requirements rather than flavour:

- A 100 g wand delivering 200 J recoils at ~1.7 km/s and destroys
  itself, so **kinetic charged items must be braced** — gun-shaped, with
  the recoil running through a stock into the ground. Handheld wands are
  not kinetic delivery devices.
- A charged cooling item absorbs the heat it pumps and can crack.
- **A spark wand is safer than the equivalent cast**, because the wand
  is in the circuit and the user is not.

### Potions ride the MATERIAL, not the flask

The load-bearing call. The magic is in the *liquid*, so it travels with
the liquid: decant a potion into another vial and it is still a potion.
Everything a potion needs then falls out of the bulk substrate for free
— dose, dilution, splitting, spilling — instead of being invented.

It also lands identity in the right place: **a potion's class IS its
material**, so identification keys on the same singleton the effect
hangs off.

The rendering and identification paths both have to **reach past the
glass** for this to be visible, and both do:

- `look flask` → `Bulkable.getContentsDescriptionFor(viewer)` routes an
  identifiable material through `RecognitionApi.describe`, so the same
  flask reads as *"an iridescent crimson potion"* to a stranger and
  *"a veiling draught"* to someone who has learned it. It rides the
  shipped bulk-contents augmenter (the `viewer` parameter was always on
  that contract; identification is the first thing that needed it), so
  every path that renders a long description gets it — see
  [bulk.md](./bulk.md) § `getContentsDescriptionFor`.
- `read scroll` → the identify effect makes the **same** redirect: the
  flask's *contents* are what gets identified, never the glass. Without
  it the answer is "there is nothing hidden to learn about a stoppered
  glass flask" — true of the glass and useless to the player holding it.

One identification therefore covers **every flask of that draught**, and
decanting carries the knowledge because it carries the substance. The
flask itself is an ordinary `Receptacle` that learns nothing, which is
the whole point.

`PotableMixin` fires from the shipped `BulkableApi.ingest` bridge,
duck-typed exactly as the drinker's own `ingest` is — so bulk never
imports magic, and every ingestion route fires it. `Metabolic` is
unchanged: a mana potion is a **concentrated carbohydrate** and needs no
new mechanism at all.

### Route is declared; delivery is not

Every potion says whether it acts **orally**, **on contact**, or **as
vapour**. The delivery half — throwing, vessel breakage, splash —
defers whole to the ranged build, because there is no `throw` verb to
hang it on. Route lands now because retrofitting it later is expensive,
and because it kills the throw-everything case without a rule: an
ingestion-only potion is a wasted flask.

---

## The charge economy

**Decay is load-bearing, not flavour.** A depleted wand is a paperweight
with a socket, so the item *count* is the wrong quantity to bound.
Throttling inflow alone cannot work — stock grows without bound at any
throttle, because nothing ever leaves. With decay:

```
  dS/dt = inflow − d·S      ⟹      S* = inflow / d
```

**Two dials whose ratio is the answer**, and a system that settles
instead of inflating. Everything the item economy wants falls out of it:

- **Recharging is a service** — a recurring business, and competence
  stays valuable when items are common.
- **You find shells and buy charge.** Wealth cannot corner the found
  channel, because what money buys is caster-labour, which is capped.
- **Shell inflation is harmless**, so distribution can be generous.

> ⚠ **Charge decay has NO far-past absence guard**, and this is the
> easiest thing in the build to get wrong. Metabolism has one for
> fairness to an absent *body*; an **item must decay while nobody is
> looking**, because that is the entire basis of the equilibrium. Follow
> **husbandry**, not metabolism. Copying the guard silently breaks
> distribution.

**Always-on is the expensive mode.** A worn item that holds its effect
up draws continuously, flattening a charge in days where a triggered
item lasts months — a real tactical choice that bites hardest on exactly
the class most prone to inflation, because people wear rings and stow
wands.

The canon line is *magic perishes, matter doesn't* — the ruins hold
perfect blades and faded rings. (D9's pattern-rot clock was the
`Focus` half of that and went with the class; a conduit has no pattern
to rot, only a tool's ordinary wear.)

### Recharging: three things, and a coupling that loses some

`recharge` used to move a caster's reserve into a shell on the strength
of **intent alone** — any caster in reach of a charged item could top it
up, at a flat 1:1, with no apparatus, no working and no competence.
Nothing in the fiction said how it crossed, and the flat rate made the
caster a **perfect pump**: the one free lunch in a model where a
firebolt spends 35.2 τ to deliver 29.9, and where mana is a metabolic
fraction (what ends up in the wand was food and water a few hours ago).

The physics was already in the tree and nothing read it —
[electricity.md](./electricity.md) ships an Ohm's-law core with
resistance, and the descriptor doctrine already asserts of wands that *a
wand IS brass, which tells you real things: **it conducts***. So:

> **Energy does not cross from a reserve into a shell by wanting it to.
> It crosses through a coupling, and couplings have impedance.**

Efficiency stops being a balance dial and becomes what it physically is.
It also makes a wand's material load-bearing for the first time, which
the doctrine always claimed and no mechanism ever cashed.

| supplies | without it |
|---|---|
| your **reserve** | *"You have no gift to pour into it."* |
| the **working** (`transfer`) | *"…no idea how to send it."* |
| a **conduit** (`ConduitMixin`) | *"…bare hands are a poor road for that much energy."* |

`delivered = committed × coupling × competence` — crude 0.6 · field 0.85
· bench 0.98, against bands 0.40 → 0.92.

> ⚠ **Efficiency can never reach 1, by construction.** Both factors are
> below one, the setter clamps, and a nonsense value degrades to *crude*
> rather than perfect. Not tuning: 1 τ ≡ 1 kJ against a conservation
> law, so a lossless pump is a perpetual-motion machine. This is also
> why it does **not** reuse `potencyFactor`, whose competence term is a
> bonus `≥ 1` above the required band — multiplying delivered energy by
> that mints joules.

**`transfer` is gated at `novice` on purpose.** It is plumbing, not the
interesting part of being a mage — and it is the part that makes a mage
*useful to other people*. Gating it high would turn a service anyone can
sell into a specialist's monopoly; competence still bites, continuously,
through the ladder rather than as a cliff.

> ⚠ **The back door is closed at the mechanism, not at the spell.**
> `transfer` authored `delta: 20` against `cost: 4` and was generating
> 16 kJ a cast, straight past the coupling. `adjust-reserve` now routes
> **any** positive delta on a `charge` reserve through the one
> implementation (`MagicApi.transferCharge`), so *no effect can add
> charge without a coupling* — not just the one that did. The
> `recharge` controller calls the same method: one mechanism, two
> triggers, which is this subsystem's own rule applied to itself.

The verb stays contributed by `ChargedMixin`, so the affordance follows
the **target** and a conduit-less caster fails audibly — exactly as a
flat wand still affords `zap` (D34). Hiding it would turn the affordance
list into a free inventory check.

### Mana recovery spends satiation and hydration

The magic tree contained no reference to either, anywhere, so **a caster
refilled their pool from nothing** — a live first-law violation, while
`arcane-science.md` prices recovery at ~300 W of metabolic work.

Mana is now the **second consumer of metabolism's coupled-recovery
keystone**, not a parallel mechanism: `coupledRecovery` takes a
`coupledConsumers()` `@hook`, and `CasterMixin` super-appends `mana` when
the faculty is active. Fuel is consumed in list order — **body before
gift** — rather than pro-rata, because a shared split would make a
caster recover endurance *slower* than a non-caster.

---

## BUC — a potency level on the item's own effect axis

Not a hidden alignment tag and not a second stat. A cursed wand of
firebolt is a *weaker* wand of firebolt. `Blessing` is a direct copy of
the `Grade` shape — quality and blessing are the same **shape** of fact
(an ordinal verdict, never a slider), so reusing it makes them compose.

`scale` and `pick` put the ordering in the **engine**: an author supplies
two ends of a range and the band picks the point, so the direction stops
being a convention every author has to remember. It is wired into the
item-discharge potency chain alongside the maker's `deliveryEfficiency`
and any pattern fade — they **multiply** (`BUC_POTENCY`: cursed 0.6×,
blessed 1.4×, ordinary the interpolated midpoint).

### ⚠ Blessed means EFFICIENT, not BENEVOLENT

A blessing amplifies **whatever the item already does**; a curse damps
or inverts it. The axis carries no opinion about whether that is good
for you — a blessed wand of dread is a *better* wand of dread. **There
is no band that means "safe."**

The payoff: knowing an item's BUC is *not enough* to know whether using
it is safe — you need to know what the item **is** as well. The two
hidden axes (class, via the belief store; band, via the
`BlessingBucket`) stay independent, and a player needs **both**.

### ⭐ The engine owns the ORDERING; the working owns the FUNCTION

The load-bearing rule, and the one this build initially got wrong.

> **BUC is a potency level on the item's own effect axis.** The engine
> owns the ordering `cursed < uncursed < blessed`; **the item owns the
> effect-as-a-function-of-potency**, and honors one contract: monotonic
> in potency. — `magic-items-slate.md`

The first cut shipped a global `BUC_POTENCY` constant (cursed 0.6×,
blessed 1.4×) applied to every discharge. That moved the *function* into
the engine, and it was wrong twice over:

- **every cursed item became the same item** — the opposite of what
  magic is for, which is that each one is its own thing;
- **it was unobservable.** Bands are shown, never numbers, so 0.6×
  firebolt and 1.0× firebolt read identically. BUC became the one hidden
  axis a player could not deduce — while the whole identification design
  rests on matching observed effects against a known vocabulary.

**Band-varying authored fields** replace it. Any scalar field of an
effect may be written as an ordered 2- or 3-step list, and the band
picks one (`Blessing.pick`, engine-side, so an author cannot invert the
ordering):

```yaml
# firebolt — MAGNITUDE, plus its own answer to being cursed
- kind: inject-channel
  channel: heat
  energy: [1, 2, 4]          # cursed · uncursed · blessed
- kind: inject-channel        # ← exists ONLY at the cursed band
  bands: [cursed]
  self: true                  # ← lands on the ACTOR, deliberately
  channel: heat
  energy: 3

# remove-curse — the SIGN INVERSION a multiplier cannot express
- kind: adjust-blessing
  steps: [-1, 1, 1]
  limit: [cursed, uncursed, uncursed]
```

Two directives beyond field variation, because *"a cursed wand also
burns your hand"* is an **extra effect**, not a different number:

- **`bands`** — membership. This effect exists only at those bands.
  Consumed at catalogue build; no executor sees it.
- **`self`** — the endpoint is the **actor**, deliberately, even with a
  target in hand. Distinct from the shipped `target ?? ctx.actor`
  fallback, which only fires when nothing was aimed at.

> ⚠ **Backfire is *firebolt's* answer, not a rule about cursed wands.**
> A frost wand might go brittle; a rod of shoving might shove the
> wielder. The engine offers the capability; what the low end of a
> working *feels* like is the author's, and it should fit that working's
> identity.

**Cursed must feel bad where it can, not merely weaker** — and it must
*read* that way. The first cut landed the backfire correctly and
narrated it with the ordinary bolt's line, so a cursed firing looked
like the same sentence printed twice and the player learned nothing.
A hidden axis cannot afford that.

### Potency: scale it if it has a size, gate it if it does not

Potency is the fraction of the working that landed —
`deliveryEfficiency × dose × competence`, all multiplying. Every effect
kind falls into one of two shapes, decided **once in the engine**, so an
author never writes anything for it:

| shape | kinds | what potency does |
|---|---|---|
| **magnitude** | `inject-channel` (energy) · `adjust-reserve` (delta) · `conjure` (volume) · `cloak`/`emit-field` (**duration**) · `afflict` *(banded)* | scales it continuously |
| **outcome** | `move` · `sense` · `afflict` *(flat)* | must clear `OUTCOME_FLOOR`; below it, nothing |

An author writes the full-strength value; dose, competence and a maker's
efficiency then work on every working ever authored, for free.

> ⚠ **It used to reach `inject-channel` and nothing else.** A half-flask
> of a healing draught healed fully; half a veiling draught veiled for
> the full term — while the draught's own seed *claimed* "half a flask
> veils you for half as long". Silent, and it looked configured. That is
> the failure mode this doctrine exists to close.

### The band is a PARAMETER, not something read off the item

`DischargeOptions.band` names the band to fire at, defaulting to the
item's own. The shipped verbs therefore behave exactly as they read,
while an author can preview a working at each band, a trap can fire
cursed from an uncursed housing, and a test can assert all three without
minting three items. Hard-wiring the read would have made each of those
need its own mechanism. Potency was already open this way
(`potencyScale`); this is the same principle one axis over.

### ⭐ Composing `Blessable` obliges you to author it

**Not optional.** `pnpm lint:blessed-bands` (CI-gating) fails the build
when a template whose class composes `BlessableMixin` carries a working
with no band variation — naming the item, the class and the working, and
offering both exits: author the bands, or stop composing the mixin.

Before it, "band-indifferent" was a quiet default: a cursed item fired
identically to an ordinary one, still reported its band, still answered
`remove curse`, and nothing said the axis was inert.

The lint checks **presence, not quality** — `energy: [2, 2, 4]` passes
while leaving cursed identical to uncursed. Same boundary the slate drew
around the monotonic contract: the lint stops silence, review stops
laziness.

### ⚠ …and the high end must not betray the working's identity

The first draft gave blessed remove-curse `limit: blessed`, so it
*consecrated*. That is a different working sharing a name: this one is
for **removing curses**.

What "better at removing curses" properly means is **scope** — the
slate's `[worn-only, all-worn, whole-inventory]`. More curses removed is
unmistakably more of the same act and cannot drift into another one. It
needs multi-target, which does not ship, so the high end is deliberately
the same cure and its value is **certainty**: you cannot read a scroll's
band before reading it, so blessed is the one you know will not turn on
you. The real high end stays named for when multi-target lands.

A cursed scroll of remove curse **lays** a curse. Not a weaker cure —
the opposite act, because that is the low end of *remove-curse's own*
axis.

> ⚠ There is **one** scroll template, not three. A per-band variant is
> the mistake this axis exists to prevent: BUC is **per-instance**, drawn
> at mint from `blessingOdds`, so `scroll-of-remove-curse` already
> produces all three bands. Three templates that differ in one field
> were briefly authored here and deleted — the test before adding a
> variant is *"is this difference per-instance state a field already
> carries?"*, and for BUC, grade, material and charge the answer is
> always yes.

Resolution happens **once, at validation** (`MagicEffects.validateForBand`),
so the catalogue holds three concrete effect lists per spell and every
executor keeps receiving a plain `Effect`. No executor learns about
bands, and there is no duplicated effect data or second catalogue entry.

**A cast always fires the ordinary branch** — a caster has no BUC. That
is the honest reading of *spells are spells and items are items*:
potency is an **instrument** fact, reachable only through the item door.

A working that authors no band-varying field is band-**indifferent**,
which is the common and honest case.

### Scoped to things with an effect axis

`BlessableMixin` is composed by `Wand` and `Scroll` — everything
that fires a working. `Scroll` earned it once the band selected the
working's own branch; while BUC's only consumer was the release gate it
was meaningless for a one-shot, and now it is the archetype.

Deliberately **not** `PotionMaterial`: BUC is per-instance and a material
is a shared singleton, so one cursed draught would curse every flask of
it. What a cursed *sword* or *chair* would mean stays unbuilt rather
than guessed at.

> **BUC is a band that every consumer interprets in its own terms.**
> There is no single mechanism and there should not be — discharge reads
> band-varying effects, the release gate reads `refusesRelease`, charge
> reads `dischargeIntoHolder`. Two engine primitives fix the *ordering*;
> the *meaning* is always local.

### Cursed sticks — the release gate

D11 sharpens it for a charged item: not merely *the slot will not
release* but **stuck on you and discharging into you**. From the
wearer's side that is one fact, so it is one call —
`Blessable.tryRelease(holder)`, returning `null` (it comes away) or a
`ReleaseRefusal` carrying what the refusal cost you. Splitting the veto
from the discharge invites a caller to check one and forget the other,
which yields a curse that sticks but never bites: a silent, plausible
bug no test of either half would catch.

Every verb that takes something off your body runs the gate through
**`Slotted.tryReleaseFromSlots(item)`** — `remove`, `unwield`, `drop`,
`give`, `put`. All-or-nothing across the item's slots, so a two-handed
cursed thing can never end up half off. The occupant makes the refusal
(the `canEvict` shape: the engine asks, the object answers, default
permit), and the slot substrate never imports the magic tree — it asks
through `MixinApi.isBlessable`.

`Wand` is `Wieldable` so there is a slot for a curse to stick
to. Without a slot claim the gate has nothing to bite: a cursed wand in
your **pack** refuses nothing, because the curse is a fact about
wielding it, not about owning it.

> ⚠ **A pre-existing bug this fixed.** `drop` / `give` / `put` moved the
> Stuff without ever vacating the slot it occupied, so a dropped sword
> lay on the floor while the hand it came from stayed full — a **phantom
> occupant, for all equipment**, reproducible with a shipped steel mace
> and nothing to do with curses. It was also the hole that made the
> release gate decorative: you could not unwield a cursed wand, but you
> could drop it.

### Generation odds — how often a mint comes out cursed

Weights, not percentages: they compose, need no total, and an absent
band weighs 0 — so `{uncursed: 1}` says "never anything else" without
spelling out zeroes. An empty or all-zero table draws `uncursed` rather
than throwing, and a negative weight is treated as absent (an author
typo must not silently curse the world).

Precedence is the stock-target shape exactly (`regionTarget` ←
`zone.stocks`), one axis over:

| Declared on | Means |
|---|---|
| an item template (`blessingOdds`) | this KIND's baseline |
| a **Zone** (`blessingOdds`, inherited down the zone walk) | **this PLACE is like that** — overrides the item wholesale |

The zone form is the interesting one: a haunted delve skewing everything
it spawns is one authored line. No shipped zone declares it yet — the
same status as `stocks` and `favours`, whose zone half also ships
unexercised.

> ⭐ **The roll happens at the random mint and nowhere else.** A
> *randomly generated* item rolls; a *deliberately made* one inherits
> intent. An author's `clone`, a crafted output, a restocked
> consignment — none come through the spawn sweep, so none get a
> surprise curse. Putting the roll in `clone` would have every test,
> every `populates:` cascade and every crafted item rolling dice.

Neither side declaring is a **no-op**, which is what lets a deliberately
cursed exemplar stay cursed instead of being re-rolled to ordinary by a
sweep that had nothing to say.

The draw lives on `Blessable.applyMintOdds` rather than in the sweep, so
distribution never imports the magic tree — it asks through
`MixinApi.isBlessable` and hands over a table, exactly as `Circulating`
reads `Arcane` without importing it.

Shipped odds are `{cursed: 3, uncursed: 95, blessed: 2}` on the wand and
rod. Cursed can roll **at all** only because `remove-curse` now exists;
the numbers are *calibrate at launch* under D21 like every other
distribution constant.

### Remove curse — `control · arcana`, not `destroy · arcana`

The obvious reading is that a curse is another caster's working laid over
the thing, which would make lifting it `destroy · arcana` — `dispel`'s
cell, and `dispel` would already do the job.

That reading is wrong here, and the distinction is load-bearing. BUC is
not a bound working; it is the item's own potency one notch down. So
lifting it **changes a parameter of a thing that remains itself**, which
is what `control` means on the grid. `dispel` cannot reach it and should
not: `relieve` scans for `magicOrigin` conditions on a **body**.

The price list then does the balancing without anyone choosing a number:
`control` is 3×, the dearest verb short of the unaffordable `transform`,
so one casting runs ~45 τ against `dispel`'s 20 — near half a mid
caster's whole reserve. A cheap cure would make the curse meaningless,
and the cost falls out of the verb rather than a dial.

**The mechanism is `adjust-blessing`** — the first effect kind that
writes an *item's* durable state (all ten others address a creature or
the world). A signed band `steps` displacement plus an authored `limit`:

| Field | Why |
|---|---|
| `steps: 1` | a displacement, not a destination — one kind serves both directions |
| `limit: uncursed` | **what makes it a cure and not a buff.** Unbounded, `+1` on an ordinary wand *blesses* it, and the right play becomes reading one over everything you own |

Two clamps, both found by testing: the ceiling bounds how far the working
pushes, and the result is then re-clamped against where it **started**,
because a ceiling must never drag something backwards — remove-curse read
over an already-blessed wand leaves it blessed. Undoing a blessing is a
different working.

It **reveals the band either way** — the first caller of the
long-unwired `revealBlessing()` seam. That makes remove-curse a BUC
detector, deliberately: a *paid* one, at the dearest verb on the list,
which is the same shape D24 gives identify.

### The condition veto

`Vitals.canAfflict` is the `canEvict` shape: the engine asks, the object
decides, permission by default. That default keeps it inert for all
seven shipped `inflict` callers while making **immunity expressible
without a registry**.

> ⚠ It runs **after** the covering-stack fold and **before** the write.
> Earlier and armor stops attenuating; later and the body is already
> hurt. Pinned by test, because either drift is silent.

### Sustained: host-held vs term-bought

A binding must be paid for continuously. A **charged host** can pay — its
standby draw meters the cost — so it re-buys its term and the hold
survives while it has charge. A **consumable** paid once and has nothing
left to renew with.

This makes the old guideline a **derivation** rather than a rule. Nothing
forbids a shadow sourced from a potion; it simply cannot outlive the term
it bought — which is why long-lived sustained effects are forged as rings
and not bottled.

---

## Identification

**Appearance is derived, never stored.** It resolves on read from
*(class, generation)*, and that single decision removes an entire
subsystem: no withdrawal sweep, no generation on the signature, no
two-live-generations transition, no ownership veto, no secondhand-market
heirloom hazard. A stashed potion re-renders on retrieval.

The accepted cost is one incongruity — a player's out-of-character memory
outruns their character's — and `label` is the fix.

A consequence worth stating: **all current-generation items of a class
look identical.** Per-instance visual variation is closed off
deliberately.

### Turnover is a window, not a switch

A hard changeover reads as a glitch. So an item's appearance derives from
its **stable position within a transition window**: nothing flips at
once, each item changes at *its* moment across weeks, and classes are
staggered against each other. The shape it produces is one everyone
recognizes — **old and new stock coexisting on the shelf**.

> ⚠ **The window position comes from a persisted seed, not `stuffId`.**
> `stuffId` is a fresh UUID per construction, so hashing it would make
> every item's flip moment jitter on reboot and items would visibly flip
> *back*. The seed stores a **position**, not an appearance, and is
> excluded from `globIdentityFields` — otherwise stacks would stop
> merging entirely.

**A stack is a batch, and batches turn over as batches**: one Stuff, one
seed, one window position. Staggering happens *across* stacks, and since
most items live in stacks that is ample. The window is **self-healing** —
once a stack passes its flip point the merge ripple folds it in.

### `knownAttributes`, and the hedge

There is **no `identificationLevel` scalar**. A stored percentage of
knowing is exactly the shape this codebase avoids: competence bands
derive, renown derives, nothing stores a fraction of a fact. You know
*facts*; how identified something is falls out of which you hold.

Each record is stamped with the generation it was learned in, so a stale
record **hedges rather than lies**:

| Record | Shows as |
|---|---|
| current generation | *a potion of healing* |
| **prior** generation | *a blue potion — you once knew blue to mean healing* |
| none | *a blue potion* |

One field, no sweep, and it only does work in the rare case. Knowledge is
never invalidated; only its applicability fades.

### Descriptor banks — two decorative axes, never material

Each bank is the **product of two orthogonal axes**: ten words on each
gives a hundred distinguishable descriptors from twenty authored words,
and every word stays meaningful. A class with N item types needs
`N × (reuseDelay + 1)` overall — about 80 at N = 20 — which no flat list
of colours supplies without descending into *pale blue* versus *light
blue*.

> **The invariant, CI-gating:**
> descriptor banks ∩ (material **names** ∪ **keywords** ∪ **appearance**
> words) = ∅

A collision is a **parser ambiguity bug**, not a stylistic wobble — if
*amber* is both a wand descriptor and a material keyword, `look at amber`
has two answers. And it is a modelling lie besides: material is a closed
curated set with real physical consequences.

⚠ **The lint runs in both directions.** Adding *amber* as a gemstone
years later collides just as badly, and that is the direction nobody
thinks to check.

The authoring trap, stated out loud: the tempting words are material
claims in disguise — **gilded, vellum, leathery, cloth, crystal, glassy,
waxen, iron-bound**. Every one asserts a substance.

What this buys is that **material stays orthogonal**. A wand *is* brass,
honestly and permanently — which tells you real things (it conducts, it
will not burn) and tells you **nothing** about which spell it holds.

### The class owns the unidentified paragraph

> **The prose under the name has to withhold what the name withholds.**

A bank supplies a *name*, and for a while that was the whole of it — the
long description stayed the authored one, always. That is a leak with no
author on the hook for it: a template's `longDescription` is written for
the **identified** item, so *a crooked beaded wand* sat over "A
finger-length rod of pale ash. A pull on it throws a lance of true
flame." The name did its job and the paragraph gave the answer away, and
the author had no reason to suspect they had written anything sensitive.

So each bank carries an **`unidentifiedLong`** — one paragraph per class,
with `{descriptor}` interpolated so it agrees with the name above it —
and the split is: **the class owns the unidentified prose, the item owns
the identified prose.** Which one you get is a fact about the reader. An
author never writes an unidentified variant of their own description,
which is the point: there is nothing to remember to do.

Two tokens, and the second is not decoration. `{a}` / `{A}` resolves
**the article for the drawn descriptor** through the same `GrammarApi`
the derived name uses. A hand-written *a* is one draw away from being a
typo at all times, and because the pool rotates on a schedule the typo
would surface months after the sentence was written, on a word the
author never saw.

The seam is `Visible.getLongFor(viewer)` — the per-viewer **base text**,
which `getMarkupLong` augments. Augmenters could not have done this:
they are handed the finished string and may only add to it, and
withholding needs a hand on the base. Silence falls through to the
authored paragraph, so a class with no prose is exactly as it was.

> ⚠ **The gate is `RecognitionApi.knowsTrueType`, which is strictly
> narrower than "shows a type name."** A **believed** name reads as
> knowledge from the inside and is not knowledge; a record from a prior
> generation hedges. Both keep the generic prose — because handing the
> authored paragraph to a misidentified reader would contradict, in the
> very next line, the name the curse just planted. **False information
> is only worse than none while it stays uncontradicted.**

Two authoring rules, both CI-gated by `lint:descriptors` where they can
be seen structurally:

| Rule | Checked? |
|---|---|
| every bank authors prose | ✅ lint |
| never hand-write the article before `{descriptor}` — use `{a}` | ✅ lint |
| **claim no material** — a specimen is truly brass or ash *per instance*, so class prose saying "of pale ash" is false on the brass one | ❌ review |

The third is deliberately not linted. The reserved-material vocabulary is
built for single-word descriptors and includes ordinary English
(*plain*, *mixed*, *down*); run against a paragraph it flags almost every
sentence, and a check that cries wolf trains authors to add exemptions.
The first two fail structurally; "does this sentence assert a substance"
is a reading, and readings are review's job.

### The parts leak harder than the prose

The same split, one layer down: a bank also carries
**`unidentifiedDetails`**, and `Wand` / `Scroll` / `Spellbook` compose
`DetailedMixin` so there is something to lens. An unidentified item is
exactly the one you want to look at closely.

> ⚠ **A detail key is a parser token.** An author names a part for what
> it *does* — `sigil`, `scorch`, `focus-lens` — so the key gives the
> answer away before the text is read, and `look wand:sigil` resolving
> *or not* is a free identification oracle. Same shape as a flat wand
> dropping `zap` from its affordance list (D34), one layer down.

So an unidentified item presents its class's parts (`grip`, `tip`) and
its own are unaddressable. Four surfaces reach a detail, and **all four**
now take the viewer they always had:

| Surface | Was |
|---|---|
| `look wand:sigil` (MQL chain → controller) | resolved by key, printed the text |
| the MQL **candidate walk** | offered authored keys as targets |
| the **inspection card** projection | `read: (stuff)` — enumerated every key + description to anyone with the card open, with no viewer in the signature at all |
| the **detail-key augmenter** | anchored any key-matching word in the prose |

> ⚠ **Two rules invert here, and both are deliberate.**
>
> **Silence fails closed.** `unidentifiedLong` falls back to the authored
> paragraph, because an item with no description is broken. An item with
> no examinable *parts* is an ordinary item, so silence shows nothing.
>
> **The keys ARE linted** for material collision, where the prose is not
> — because they are parsed. `look at amber` cannot have two answers.

The lensing lives in **`Detailed.detailRoot(viewer)`**, the single root
every read starts from. `Detailed` *asks* through the mixin registry
rather than `Identifiable` overriding it: an override binds only when
`Identifiable` composes above `Detailed`, which is true of every class
today and enforced by nothing. Asking is order-independent, and keeps
`Detailed` from importing the identification tree at all — the same
duck-typing that lets `BulkableApi.ingest` fire `PotableMixin` without
bulk importing magic.

The augmenter row is worth its own line: **tests could not have caught
it.** It needs prose and keys authored independently to collide, and
they did — the unidentified scroll's class prose says *"in a hurried
hand"*, the scroll has an authored `hand` detail, and the paragraph came
back with `<detail key="hand">` in it. Nothing in the text revealed
anything; the anchor did, by asserting the thing has a part called that.
It took driving the game.

### BUC and merge behaviour

Stack identity keys on the per-instance **bucket** (`unknown` until
revealed), never the true band. Two unknown items merge regardless of
their true state, so **merge behaviour leaks nothing** — a stack that
silently refused to merge would be a free curse detector.

A **labelled** item does not auto-merge: a player wrote on that one on
purpose.

---

## Spellbooks and memory

**A book mints a claim, never a deed.** Competence is derive-on-read over
(Discipline × Transcript), so a book that granted skill would have to
write evidence of practice that never happened. The ordinary outcome is
the honest one: **you know a spell you cannot cast** — the practicum
thesis in one object.

**Claim lives in the chronicle; sharpness lives on the mixin.** The
chronicle is append-only and idempotent, right for *you have read of
this* and useless for a decaying quantity. Sharpness is *specification
state*, which is permitted to decay: **competence never fades;
specifications do.**

> Amnesia therefore strips the **held specification**, not the chronicle
> claim — "I once read this" is a fact about the past nothing should
> unmake. The asymmetry is preserved and the ledger stays append-only.

### Fade is felt as cost, never as failure

A hazy spell costs more mana for the same effect and eventually is not
worth casting. **It never fails outright** — `costMultiplier` is finite
at zero sharpness on purpose. That is what makes memory *felt*
continuously, in a number the player already watches, rather than felt
once at the worst moment as a gotcha. Casting renews the pattern, so you
lose only what you do not use.

Four axes, each with a real basis, and **none of them is the book**:

1. **Maturity** — each refresh lengthens the next interval (the spacing
   effect). The anti-treadmill mechanism: upkeep falls on *new*
   additions, so a large mature library is nearly free.
2. **Competence in the relevant Disciplines** — expertise slows
   forgetting for structured material *in its own domain* (Chase &
   Simon). Competence pays twice, which makes **breadth expensive and
   depth cheap** — and items are the counterweight that buys breadth
   without upkeep.
3. **Specification complexity** — derived from the spell's own shape, so
   no author can quietly make theirs cheap to remember.
4. **Interference** — the **repertoire limiter**. There is no slot count
   and nothing forbids reading every book in the library, but the more
   you hold the hazier they all get. **Vancian preparation emerges**
   rather than being imposed: a broad generalist is permanently mediocre,
   a specialist's few are razor-sharp.

### The comprehension floor and the defective copy

A book carries a comprehension floor **below** its casting floor. The gap
is a **read-ahead window** — a band where you can take a specification on
board and not yet execute it, so practice has a visible target.

**Reading above the floor does not fail.** It produces a *defective
specification the reader believes is correct*: costs far more mana than
it should, behaves oddly, and the holder does not know. Honest
(half-understanding is worse than none precisely because you don't know
you're wrong), **legible** (the efficiency is visibly off), and
**recoverable** (re-study once competent replaces it cleanly).

That is what gives the library teeth — gorging on books above your level
is not free-but-useless, it fills your head with copies that cost you on
every cast. And it makes a **trustworthy teacher** valuable, since what
they sell is *not getting a bad copy*.

**Book quality sets refresh speed, not fade rate.** Once a specification
is in a mind, the book that put it there has no say in how fast it
degrades; a well-written one only gets you back to sharp faster. That is
the teaching product, and it is legible.

Books are **identified items** on the same axis as potions, which puts a
library's product in the right place: **the catalog, not the books.** New
stock causes a *cataloguing backlog*, not a rush.

---

## Distribution

**Rarity derives; there is no authored rarity table.** Spawn weight is
the **inverse of stored labour**, read off the grid cell through
`lib/magic/PriceList.ts`. An authored table would let the economy and the
physics disagree — something expensive to make and common to find.
Multi-effect items take rarity from their **most expensive cell**.

### The census

> **Circulation = what is reachable in the world now.** A wand in a
> logged-off player's pack is a *withdrawal* until they log in.

That is the discovery slate's own stock model, and the right quantity for
the decision being made: both injection channels place into *live*
regions, so live regional stock is what they must not over-fill.

Deliberately **not** a reach into persistence: `holder_snapshots` rows
are per-mixin capture blobs, not an item index, so counting from them
would need a new index plus a write-path obligation on every item move —
while the snapshot-parked tail is self-correcting and bounded.

> ⚠ It is an **MQL system-mode query**, not a registry walk.
> `check-world-scan` allowlists exactly three files for
> `getAllObjects()`, and this is not one of them. If profiling ever shows
> it too costly, fold the count into `ResidencyLogic`'s existing sweep —
> already allowlisted, already walking every object once.

### Both channels, and how they coordinate without knowing about each other

- **Deliberate placement** — a declared par on a `ResettableMixin`
  holder, which the reset sweep tops toward. Par is a *target*, so a
  taken item is replaced only when the holder is below par.
- **Random draw** — the weighted table, on the third self-maintenance
  sweep alongside eviction and reset.

**Authored placement counts and spends nothing.** It simply shows up in
the census, and the random channel backs off on its own. **No allocation,
no budget, no new economy** — an allocation would be a new resource
needing an owner, a ledger and administration, and the corrective already
exists: **decay**. An over-filled area becomes an area full of dead
shells, which are harmless.

The census binds **regionally**, never globally — a global one would let
one author's hoard starve the world.

`populates:` is deliberately **not** the injection mechanism. It stays
what it is: a clone-time cascade for set dressing.

---

## Verbs, and the two rules that govern them

**Capabilities confer verbs; items never do.** A potion never says *"I
grant quaff"* — it is `Bulkable` and potable, and that is sufficient. The
verb is written once on the capability mixin and every member gets it
free.

> **⚠ Conferral controls the affordance LIST, never the PARSER.**

If `drink` is unlisted because nothing here is drinkable, typing `drink`
must still answer *"there is nothing to drink"* — never *"unknown
command."* Hiding from a list is helpful; hiding from the parser teaches
players that verbs evaporate. The dispatcher's **parser floor** consults
`allDefinitions()` before calling a verb unknown, and emits `unafforded`
with authored prose — one general mechanism, so every verb in the game
gets the floor.

> **⚠ An affordance may never vary with hidden state.**
> **A conferred verb keys on what the holder can SEE.** Appearance
> derives from class, so the **kind** is always visible while the
> **class** is hidden. Therefore: **verbs key on kind, effects key on
> class.**

Three consequences, each of which would otherwise ship as a bug:

- **There is no `identify` verb.** A scroll affords `read` like every
  written thing, and what happens when you read it is the effect. An
  `identify` affordance would identify the scroll for free.
- **No per-class verb variation.** Every wand affords the same `zap`; if
  a fire wand afforded `scorch`, the verb list would identify it.
- **A depleted wand still affords `zap`.** A verb that vanished when the
  charge ran out would be a free charge meter — and charge is meant to
  need an instrument.

### `read` decomposes into perceive + decode

```
read = perceive(the marks) + decode(the script)
```

Looking is one sense channel resolving; reading is perception **plus
symbolic decoding**, and the two come apart in both directions. So every
written thing carries a **modality** — inked (visual), embossed
(tactile), or both — reusing the shipped `SenseChannel` vocabulary rather
than declaring a new one.

What falls out is mechanical rather than gestural: **an embossed text is
readable in the dark**, a real advantage worth paying for on an
expedition; and a character without functioning sight is not excluded
from the spellbook economy, by the same mechanic.

### Two axes, not one

`form` gates **perceive**; `script` gates **decode**, and they are
independent:

| | form | script | perceive | decode |
|---|---|---|---|---|
| raised common lettering | embossed | common | touch | ✓ |
| **braille** | embossed | braille | touch | ✗ (v1) |
| a ciphered dispatch | inked | cipher | vision + light | ✗ (v1) |

Braille is what proves they are two: touch-only *and* its own system,
where raised common letters are the same physical make in the ordinary
system. Under one field those are indistinguishable. `obj/Signpost` +
`seeds/obj/items/dotted-slate.yaml` are the worked A/B against the
equally-embossed `scroll-of-remove-curse`.

The symmetry worth keeping: **braille and cipher are the same field
pointed in opposite directions** — one includes a reader through a
different channel, the other excludes one through a different system.

**Perceive is an intersection**, both directions: the reader's channels
against the marks' modalities, with light gating only the vision branch.
The first cut asked *"are these vision-only? then check light"*, which
left touch unconditional — embossed text was readable by anyone, in the
dark, with no sense of touch, and the "a sightless reader is not
excluded" payoff came from a gate being **skipped** rather than from a
model.

> ⚠ **Touch is not organ-modelled.** `sensoryPorts` describes organs
> (count, position — eyes, ears, a nose), so **no shipped body plan
> declares touch**; it is the integument and the schema cannot say it.
> A gate that required it in the sensorium made every embossed text
> unreadable by everyone. `NON_ORGAN_CHANNELS` is the placeholder, and
> it shrinks to empty once a real touch model exists — at which point a
> numbed or gloved character correctly fails to read raised lettering.

Literacy is out of scope for v1 — everyone reads the common script — but
the seam is in place: unknown scripts are **withheld, never mangled**
(the medium here IS text, so scrambled output is indistinguishable from
a bug), and the reader-side "which systems do you know?" lookup slots
into `decode` without disturbing `perceive`.

**A working that needs a mark ASKS for one.** `read scroll` is the only
form; when `MagicApi.requiresMark` says the working demands a target,
the controller raises a `PromptApi.mqlObject` over reachable candidates
and spends nothing until one comes back.

> ⚠ It briefly took the mark as a preposition — `read scroll at flask`.
> That was wrong and the reason is worth keeping: **reading is not
> aimed.** You read a scroll; you do not read *at* a flask. The clause
> existed because the prompt substrate went unused, not because the act
> is directional, and `read sign at flask` parsed happily.
>
> The test is whether the **act** is aimed, not whether the *working*
> is. `zap wand at troll` keeps its preposition — pointing a wand is a
> real directional act — and gained the same prompt for its bare form,
> because refusing a `zap` that only needs to know *what at* is
> answering a question with a wall.

The demand itself is the shared targeting floor
(`MagicEffects.everyEffectNeedsTarget`), not a `read` special case: the
`cast` path reads the same one.

---

## Files

**Effect spine** — `lib/magic/EffectContext.ts` · `CastingProfile.ts` ·
`Arcane.ts` · `Grid.ts` · `obj/api/MagicLogic.ts` (`discharge`)

**Item classes** — `lib/magic/Charged.ts` · `Focus.ts` · `Consumable.ts` ·
`Potable.ts` · `Charge.ts` · `Dose.ts` · `Conduit.ts` · `obj/magic/Wand.ts` ·
`Scroll.ts` · `Spellbook.ts` · `obj/material/PotionMaterial.ts`

**BUC** — `lib/magic/Blessing.ts` (`draw`/`hasOdds`) · `Blessable.ts`
(`applyMintOdds`) · `obj/api/ResidencyLogic.ts` (the mint-site roll) ·
`lib/vitals/Vitals.ts` (`canAfflict`) · `obj/magic/Wand.ts`
(the only composers) · `obj/api/MagicLogic.ts` (`BUC_POTENCY`,
`execAdjustBlessing`) · `seeds/obj/magic/Spell/remove-curse.yaml` ·
`seeds/obj/items/scroll-of-remove-curse.yaml` ·
`seeds/obj/items/wand-of-firebolt-cursed.yaml`

**Identification** — `lib/identification/Appearance.ts` ·
`DescriptorBank.ts` · `Identifiable.ts` · `lib/description/Labelled.ts` ·
`Marked.ts` · `packages/content/arcane-descriptors/` ·
`lib/bulk/Bulkable.ts` (`getContentsDescriptionFor` — the reach past the
glass, shared by `look` and the identify effect)

**Memory** — `lib/magic/SpellKnowledge.ts` · `Memorized.ts` · `Fade.ts` ·
`StudyActivity.ts`

**Distribution** — `lib/residency/Circulating.ts` · `Census.ts` ·
`SpawnTable.ts` · `lib/magic/PriceList.ts` · `obj/api/ResidencyLogic.ts`
(`installSpawnSweep`)

**Verbs** — `magic/zap` · `magic/recharge` · `magic/study` ·
`perception/read` (+ `at <mark>`) · `inventory/label` · `bulk/drink`
(+`quaff`)

**Lint** — `pnpm lint:descriptors`
(`scripts/check-descriptor-banks.ts`), CI-gating, both directions.

## Cross-references

[magic.md](./magic.md) · [arcane-science.md](../arcane-science.md) ·
[belief.md](./belief.md) · [bulk.md](./bulk.md) ·
[metabolism.md](./metabolism.md) · [residency.md](./residency.md) ·
[reserve.md](./reserve.md) · [glob.md](./glob.md) ·
[chronicle.md](./chronicle.md) · [advancement.md](./advancement.md) ·
[content-packs.md](./content-packs.md)
