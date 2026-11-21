# Embodiment

Body-side affordances on the slot substrate. `Wearable` and
`Wieldable` are specialized `Slottable` mixins that carry per-body-
plan slot claims and the corresponding `fitsSlot` test.

## The Cast

| Name | Location | Role |
|---|---|---|
| `Wearable` | `lib/slot/Wearable.ts` | Body-slot claim for clothing / armor |
| `Wieldable` | `lib/slot/Wieldable.ts` | Body-slot claim for weapons / held items |

Both compose on `Stuff & Slottable & Containable` — wearables and
wieldables live in inventory before being worn / wielded.

## Per-body-plan claims

Each Wearable / Wieldable carries `slotClaims: Record<string, string[]>`,
where the key is a body-plan template path and the value is the
ordered list of slot names this item claims on that body plan.

```yaml
# Boots template
slotClaims:
  /idea/race/bodyplan/biped:    ['foot:left', 'foot:right']
  /idea/race/bodyplan/quadruped: ['hoof:fore-left', 'hoof:fore-right',
                                   'hoof:hind-left', 'hoof:hind-right']
```

A body plan that doesn't appear in `slotClaims` is ineligible —
`fitsSlot` returns false.

`WearableMixin.getClo()` is the thermal insulation a worn garment
contributes. ⚠⚠ **It is DERIVED and the field is gone** (textiles): a
wool coat is warm because wool conducts at 0.04 W/mK and its form traps
air, not because somebody typed a number, and an authored `clo` silently
overrode the whole thermal model. It reads material density and
conductivity, the construction form's loft, the garment's mass, and its
own `slotClaims` (so a garment states its clo with no wearer). See
[thermal.md](./thermal.md) § *Worn insulation*.

### ⭐ The covering stack — one walk, on the wearer

`AttiredMixin` answers about the wearer's covering (the Attired split — `lib/slot/Attired.ts`):

| method | answers |
|---|---|
| `wornStack()` | everything worn, outermost-first (`Wearable` occupants only — a sheathed sidearm is *slotted*, not worn) |
| `coveringAt(part, {includeHeld})` | the covering over one body part, outermost-first |
| `outermostAt(part)` | which layer takes a deposit — the soiling seam |
| `insulationAt(part)` / `bodyInsulation()` | clo over a part / surface-weighted over the body |
| `windproofing()` | how well the outermost layer breaks a wind |
| `wouldLayerViolate(candidate)` | the ladder refusal |

⭐⭐ **Three logic singletons hand-rolled the same outside-in walk** —
the trauma covering walk, the struck-site armor stack and the conduction
walk — and all three now call `coveringAt`. Each already holds the host,
so the call *drops* a parameter rather than adding an Api hop. ⚠ There is
no covering-stack Api and there must not be: a covering read is one host
answering about its own slots, which is none of the four mandates
`check-object-verbs` allows.

**The ordering rule, in one comparator:** *form sets the band; wear-order
breaks ties inside a band.* Bands are `Construction.getLayerDepth()`
(padded 0 · quilted 1 · hide 2 · mail 3 · plate 4, and a fabric's own
`layerBand` on the same ladder); anything with no covering form sorts
innermost. Wear order is slot insertion order, and the persistence spine
re-wears through `occupyAll` in the captured order, so it is **durable
with no new field**.

⚠ `WearController` refuses only the **inversion** — a low band outside a
high one, i.e. a shirt over plate, with a `layer-order` note.
Shirt-vs-coat is not refused: both are band 0, which goes on first is
the player's call, and its consequence is being cold.

⚠⚠ **Covering slots need `capacity` > 1 and did not have it.** Every
wear slot defaulted to 1, so a body could hold exactly one torso garment
— the shipped gambeson and the shipped hauberk could never be worn
together, and the entire outside-in model had nothing to walk. The
biped's and quadruped's covering slots now carry `capacity: 4` (shirt,
gambeson, mail, surcoat). It is a cap rather than unbounded because
"wear forty shirts" would otherwise be free insulation.

## ⭐⭐ Fit — two derived numbers and one stamp

A garment fits a body or it does not, and the whole model is two
numbers:

```
statureM   = species.getStature()
girthIndex = √(massKg / statureM)          a ponderal index
```

`massKg` is `Creature.getMass()`, which already reflects composition.
⭐ **That is the lineage seam, and it is one line** — individual variance
arrives through `getMass()` alone, and nothing in textiles is touched to
consume it. A test asserts exactly that.

**The stamp** is three scalars on `WearableMixin` — `cutToBodyPlan`
(`''` = stock), `cutToStature`, `cutToGirth` — each persistent,
authorable and validated on set. ⚠ Three named scalars rather than one
composite: a **fixed-key** composite of three scalars is precisely the
case the persistent-fields doctrine says decomposes.
(`Wearable.slotClaims` is the contrasting **variable-key** case and
stays a raw map; the distinction is fixed keys vs. variable keys, not
"object vs. scalar" by eye.)

⭐ **An absent stamp means STOCK**, and stock resolves to the wearer's
body plan's average — so all fifteen shipped rows read as ill-fitting
hand-me-downs **with no content edit**, and a near-average body is
served passably while an unusual one is served badly. Which is exactly
what off-the-rack clothing does, and it needed no authored fallback.

`garment.fitOn(wearer)` returns a `FitReading`: the two measurements,
the relative `distance` between them, and a signed `looseness` /
`tightness`. ⭐ **It sits on the GARMENT, not the wearer** — the garment
carries `cutTo` and is the thing that fits or does not; the wearer is
the argument.

**Consequences, each on a shipped mechanism:**

| reading | consequence | where |
|---|---|---|
| *loose* | air gaps → `insulationAt` scales the layer's clo by `1 − looseness × dial` | `Attired.insulationAt` — it needs the wearer, which is why it is not inside `getClo()` |
| *tight* | a burden surcharge on the placement coupling | `LoadBearing.getBorneBurden` |
| *tight* | a multiplier on the **existing** per-blow condition decrement — ⚠ never a clock | `ConditionLogic`'s covering-wear site |
| *impossible* | `WearController` refuses above `textiles.fit.refuseAbove` | a `fit-impossible` note |

⚠ A `cutToBodyPlan` **mismatch is a hard refusal independent of
distance**, and it is not redundant with `slotClaims`: every playable
species is `biped`, so slot matching alone would let a halfling's coat
onto a dragonborn. ⭐ And the *distance* refusal fails on a **number**
rather than a species check, so a heavy human and a light dragonborn
shade into each other correctly.

Free consequences, none of them authored: a **secondhand market** (a
garment cut for someone else fits you badly), **cross-species failure**,
and **the tailor's economic reason to exist**.

## `fitsSlot` overrides

`Slottable` ships a default `fitsSlot(host, slot) => true`.
`WearableMixin` and `WieldableMixin` override it: walk
`host → species → bodyPlan` via `SpeciesApi.tryGetBodyPlanPath`,
then check `slotClaims[bodyPlanPath].includes(slot)`. The host's
`Slotted.canOccupy` calls this after the slot-side mixin check
passes, so the candidate gets the final say.

Subclasses override `fitsSlot` for richer rules (a magic boot that
only fits Elven feet, etc.; a shield answers yes for `back`, so it is
slung there without the claim naming it).

⭐ **A carrier's hand is the exception** (size and carrying W4/W7): the
claim says where `wield` puts a thing, not which hands may close on it.
Any `Attired.handSlots()` slot takes any slottable thing — the second
hand by weight, a readied compass, a lead rope's end, a mug — and the
grip (`Carrying.wieldPlanFor`) is what can refuse.

> ⭐ **Prosthetics are the augmentation build's, not embodiment's.** A
> stand-in for a lost limb is a **"replace" augment** — vitals owns the
> loss (`severPart`, `ownFunction → 0`), augmentation owns the
> replacement (slot-confers-capability, a capability profile, not a
> scalar). See [augmentation.md](./augmentation.md) /
> `augmentation-slate.md`. The recovery build deliberately did NOT ship a
> prosthetic mixin.

## Multi-slot atomicity

A wearable claiming multiple slots (boots → both feet, longbow →
both hands) uses `occupyAll` in the controller — either
every slot is claimed or none. Failure surface names the blocked
slot. Removal vacates every claimed slot.

## Wearable + Wieldable overlap

Some objects (gauntlets, bracers) compose both. Each side declares
its own `slotClaims`. **The OBJECT selects the mode, not the word**:
`equip gauntlet` claims whichever side the gauntlet offers. No umbrella
mixin.

## Verbs

⭐⭐ **Six verbs, one controller each over two bases.** `equip` / `unequip`
are the orchestrators; `wear` · `wield` · `remove` · `unwield` are the
precise acts and **keep their own views** (`EquipControllerBase` /
`UnequipControllerBase` since controllers-are-io W7; `doff` is deleted). See
equip-slate.md *(slate retired 2026-09-06 — fully absorbed)*.

| Verb | Action |
|---|---|
| `equip` | The whole kit, **innermost-first** |
| `equip <X>` | `occupyAll` on `X.slotClaims[actor.bodyPlanPath]`, worn or held |
| `equip <X> --from <box>` | Draw from a container in reach first |
| `outfit wear <name>` / `outfit save <name>` / `outfit list` | The wardrobe verb (`WardrobeMixin` affords it; was `equip set …` until controller-liquidity W16) |
| `unequip [<X>]` | Vacate claimed slots, **outermost-first** |
| `wear <X>` / `remove <X>` | Precise: `requires: WearableMixin` |
| `wield <X>` / `unwield <X>` | Precise: `requires: WieldableMixin` |

### ⚠⚠ Why the precise verbs were NOT made aliases

A draft collapsed all six into two, which reads well and costs a real
distinction. An arg that admits `WearableMixin|WieldableMixin` **cannot
refuse either**, so `wear sword` becomes legal — and the refusal that
used to be the ARG's (a grammar error, before any controller runs)
would become a rule, or nothing.

So the split is the **grammar**, and the act is one implementation:
`EquipControllerBase` and `UnequipControllerBase` each back three views
(one thin controller per verb since controllers-are-io W7 — `context.verb`
decides nothing; the controller's own mode does). The
gates (fit, the ladder, the slot) and the timing live in one place,
because duplicating a controller to express a grammar difference is how
two halves of one rule drift apart.

⭐ The consequence is that **the invoked verb is real input** (which
controller ran, never the typed word). A gauntlet
is `Wearable` and `Wieldable` both: `wear gauntlet` puts it on, `wield
gauntlet` takes it up, and bare `equip` prefers the worn reading (the
shipped precedence — `getSlotClaim` was always read off `Wearable`
first). `EquipControllerBase.claimAs` is the one place that decides, and
`slotsFor` reads the claim off the face it selected. The fit gate and
the covering ladder are the **worn** reading's alone: a gauntlet in the
hand is not covering anything.

### ⭐⭐ Ordering is the reason the verb exists

The covering ladder means a low band may not go outside a high one — a
fact about the model that the engine knows exactly and that the player
was rediscovering one refusal at a time. `equip` sorts candidates by
`Construction.getLayerDepth()` before the walk and re-checks
`wouldLayerViolate` per item for what is *already* on.

⚠ **A retry-sort is not enough**, and the first implementation learned
it the hard way: repeatedly putting on whatever is currently legal fails
once a high band lands first, because nothing takes it back off. A
cuirass ahead of a shirt in the pack dressed the actor in the cuirass
alone.

### ⭐⭐ Dressing costs time

Each layer is a `DressingStep` (`lib/slot/DressingStep.ts`) occupying
`hands` — voice stays free — for a duration **derived from the garment's
mass** (`equip.baseMs` + `equip.msPerKg`), never authored per row. So a
shirt is seconds and a hauberk is minutes, and **you cannot armour up in
an ambush**.

⚠ Layers **chain**: the next starts from the previous one's completion,
because firing them together is an `engagement-conflict` and dresses you
in exactly one garment. An interrupted dressing leaves what went on ON —
the covering stack is a stack, and a shorter one is already coherent, so
no rollback exists or is needed.

⚠⚠ **The completion path is STATIC** (`EquipControllerBase.put` and the
reads it makes). A controller is one ephemeral clone per execution,
destructed when `execute` returns, so a step completion calling back into
the instance ran on a destroyed Stuff and the proxy no-op'd it: *"You put
on a pair of stout leather boots"* and the feet slot stayed empty —
**nothing was ever worn in a live world** until the rubber build's
browser drive (2026-10-18). ⭐ `wear` now also names the fit: *"It is
tight on you."* / *"It hangs loose on you."* (`fitWords`).

⚠ A skip is a **caveat, not a rejection**: `equip` that got five of six
pieces on succeeded, and says what it left and why without emitting
`controller-rejected`.

Validators (the controllers fail-fast on type narrows that shouldn't be
reachable post-validator; user-facing rejection lives in these):

- Every verb: verb-level `requiresAnimate`, `requiresSlotted` (the
  actor's body needs slots to claim into), `requiresEmbodied`.
  `requiresConscious` on all but `wear`.
- The four precise verbs: `required: true` target, `scope: inventory`,
  `mustBeInInventory`, and the narrow `requires:` above.
- `equip` / `unequip`: `required: false` (bare is the whole-kit form),
  `requires: WearableMixin|WieldableMixin`; `equip` alone widens
  `scope` to `[inventory, reachable]` for `--from`.
- The controller's own per-slot scan surfaces "you aren't wearing that"
  when nothing actually vacated.

## ⭐⭐ Hands hold what you carry — the rule since the carrying flip

**Reversed 2026-11-12 (size and carrying).** This section used to say
*hand slots are for activities, not storage* — `get` put a thing into an
ambient "on your person" pool and receiving a thing never needed a free
hand. That pool is gone. The rule now:

> **A hand is a slot whose body part serves manipulation, and it holds
> whatever you carry in it. Everything on a body is held, worn, or placed
> on something worn.**

- **Hands are derived, not named.** `Attired.handSlots()` is the slots of
  the body plan whose part `serves: [manipulation]` — the biped's
  `hand:left`/`hand:right`, the `canid` plan's `mouth`. A horse has none.
- **A hand takes any good.** `Good` and `Vessel` compose `SlottableMixin`,
  and a carrier's hand slot admits any slottable thing whatever its
  `accepts` — the grip decides (`Carrying.wieldPlanFor`: one hand, two, or too
  heavy, from the holder's own carry capacity and stature, or two because
  the thing's *use* needs two — a bow, a musket, a pike). Every other slot
  still asks its `accepts`. A garment in a hand is **held, not worn**
  (`Attired.wornStack` skips hands); `wear` dresses what is held.
- **`get`, `give`, `buy` and every other way onto a body are
  `ContainmentApi.move(item, body)`** — the body places it (a free hand,
  else a pocket or a hook, by `carry.order`), a carrier routes the rest
  into a bag it carries, else sets it down at its feet with the line
  (*Your hands are full — you set the cup down.*). Only `get` refuses
  rather than pick a thing up to set it back down (`Carrying.wouldSetDown`),
  in words that name what the hands hold. A dual-wielder handed a
  spellbook takes it in a pocket or a bag if one fits, else it lands at
  their feet.
  That friction is the point now: what you hold is a choice that costs you
  something (see [carrying.md](./carrying.md)).
- **Readying is `get X from` / `fight switch`; nothing frees a hand for
  you** (size and carrying W10). `get compass from pouch` brings it into a
  FREE hand, or is refused naming what the hands hold and the ways out
  (`drop`, `unwield`, `put`); an instrument verb (`measure`, `read`,
  `light`, `pour`, the crafting tools) uses the tool where it sits; a
  thing in a shut container or a packed pack is refused, naming the fix
  (`open`, take the pack off). In a fight every hand act is a priced beat
  by where it comes out of.
- **`wield` is still a commitment.** `wield`/`wear`/`mount`/a posture seat
  occupy slots because each is a body-level commitment; `equip` reads
  `Carrying.wieldPlanFor(item, claim)`, which widens a one-hand claim to two
  when the holder's weight or reach says so, and refuses `too-heavy` /
  `hands-needed` / `hands-on-cart` in words.

Activity verbs that need a hand for a duration still compete with `wield`
for the slot (the engagement framework's slots — [activity.md
§ Engagement slots](./activity.md)); the difference is that what you are
carrying competes too, which is the honest tension.

## History

- **Size and carrying (2026-11-12, `39d4ba23e..50f562ed4`).** *Hand slots
  are for activities, not storage* was reversed: hands hold any good, the
  ambient carry pool retired, `receive` was the one way onto a body (W8:
  folded into `ContainmentApi.move` — the body places what arrives), and
  `equip` consults the holder's grip (`wieldSlots`/`wieldRefusal`, now
  `Carrying.wieldPlanFor`). Finalize (2026-11-16; the whole build
  `39d4ba23e..HEAD`): `handSlots` is `Attired`'s, `handsFor` folded into
  `wieldPlanFor`, readying never frees a hand for you. The
  hand-slot `fitsSlot` short-circuit lives on `Wieldable` (a carrier's
  hand) and on `Slotted.canOccupy` (any slottable in a hand).

## Cross-references

- [slot.md](./slot.md) — substrate.
- [carrying.md](./carrying.md) — hands, `move` onto a body, the tiers, readying.
- [size.md](./size.md) — what fits, the envelope.
- [race.md](./race.md) — BodyPlan's `slots: SlotSpec[]` declaration.
- [command-spec.md](./command-spec.md) — verb authoring.
