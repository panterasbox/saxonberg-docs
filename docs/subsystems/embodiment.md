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

`SlottedMixin` answers about its own slots:

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
only fits Elven feet, etc.).

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

⭐⭐ **Six verbs, two controllers.** `equip` / `unequip` are the
orchestrators; `wear` · `wield` · `remove` · `doff` · `unwield` are the
precise acts and **keep their own views**. See
equip-slate.md *(slate retired 2026-09-06 — fully absorbed)*.

| Verb | Action |
|---|---|
| `equip` | The whole kit, **innermost-first** |
| `equip <X>` | `occupyAll` on `X.slotClaims[actor.bodyPlanPath]`, worn or held |
| `equip <X> --from <box>` | Draw from a container in reach first |
| `equip set <name>` / `--save` / `equip sets` | The wardrobe stanza |
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
`EquipController` and `UnequipController` each back three views. The
gates (fit, the ladder, the slot) and the timing live in one place,
because duplicating a controller to express a grammar difference is how
two halves of one rule drift apart.

⭐ The consequence is that **the invoked verb is real input**. A gauntlet
is `Wearable` and `Wieldable` both: `wear gauntlet` puts it on, `wield
gauntlet` takes it up, and bare `equip` prefers the worn reading (the
shipped precedence — `getSlotClaim` was always read off `Wearable`
first). `EquipController.claimAs` is the one place that decides, and
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

## Hand slots are for activities, not storage

A design rule the shipped verbs already obey, written down so
future verb design doesn't drift:

> **Body slots model active commitments. Ambient inventory
> models possession. The two are distinct destinations.**

`wield`, `wear`, `mount`, posture-bearing seats — all occupy
slots, because each represents a body-level commitment: the
hand is being used for this sword right now, the torso is
being clothed by this shirt right now. Releasing the
commitment (`unwield`, `remove`, `dismount`, `stand`) frees
the slot.

`get`, `drop`, `give`, future `put`/`take` — all move items
in and out of the actor's general `Container`. This is the
"on your person" abstraction: pack, belt, sheath, pouch are
all narrative flavors of the same ambient-inventory slot. The
model deliberately does not author backpacks-as-slots; if a
container-on-the-body is meaningful, it's a Stuff worn in a
slot whose contents are still ambient inventory addressing.

The corollary that matters for verb design: **receiving an
item never requires a free body slot.** A character dual-
wielding two swords can still be given a spellbook — it goes
into ambient inventory, and they can `wield` it later if they
free a hand. The realism is preserved where it pays off (you
can't actually hold three swords); the friction is removed
where it didn't (handing someone a book shouldn't depend on
their combat loadout).

Future activity verbs that need a body slot for the duration
of the activity (cooking grabs a hand for the spoon; reading
grabs `attention` for the book) compete with `wield` /
`wear` / posture for slot occupancy — that's the right
tension, and the engagement framework already supports it.
See [activity.md § Engagement slots](./activity.md).

## Cross-references

- [slot.md](./slot.md) — substrate.
- [race.md](./race.md) — BodyPlan's `slots: SlotSpec[]` declaration.
- [command-spec.md](./command-spec.md) — verb authoring.
