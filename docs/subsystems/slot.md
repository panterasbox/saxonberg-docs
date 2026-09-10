# Slot substrate

`Slotted` / `Slottable` are the canonical capability pairing for "I
have named occupancy positions" / "I can occupy one of them." The
substrate underneath the body-side affordances (Wearable, Wieldable),
posture (Postured), conveyance (Mountable, Drivable), and the
boundary fixture system (post-retrofit `Adornable`).

## The Cast

| Name | Location | Role |
|---|---|---|
| `Slotted` | `lib/slot/Slotted.ts` | Host mixin — exposes slots that things can occupy |
| `Attired` | `lib/slot/Attired.ts` | Host mixin — the **covering** half, split off `Slotted` 2026-09-06. Composed only on `Creature`: a body is the only thing that wears anything |
| `Slottable` | `lib/slot/Slottable.ts` | Marker mixin — anything that can sit in a slot. Carries `fitsSlot(host, slot)` with a default `() => true`; Wearable/Wieldable override |
| `SlotSpec` | `lib/slot/Slotted.ts` | Per-slot declaration (name, accepts, capacity, postures, userFacingDetail, `bodyPart`, `covers`). `bodyPart`/`covers` are optional `body.*` references to anatomy (Vitals) — a slot is its own axis that *references* anatomy where it has a home; see [vitals.md](./vitals.md) |
| `UNBOUNDED_CAPACITY` | `lib/slot/Slotted.ts` | Sentinel = `Number.MAX_SAFE_INTEGER`; JSON/BSON-safe substitute for `Infinity` |
| `BodyPlanSlotsMixin` | `lib/slot/BodyPlanSlots.ts` | Sibling provider — Pattern B, derives universe from species → bodyPlan |
| `Foldable` | `lib/slot/Foldable.ts` | Host capability — two-state fold/unfold; a folded host refuses its posture-bearing slots (gates `canOccupy`) |
| `Slotted/Slottable` | `api/slot.ts` | Cross-cutting helpers (multi-slot transactional ops, inverse lookups, slot resolution, conveyance ripple walker) |

`Slotted` composes on `Stuff` (no `Container` prereq). Composing
`Slottable` doesn't constrain a host — it just marks a Stuff as
slot-occupant-eligible.

## ⭐ `Slotted` vs `Attired` — occupancy vs covering

They were one mixin until 2026-09-06. **A slot is not a garment.** Nine
of ten `Slotted` composers are not bodies: a chair, a coat rack, a
garden bed, a door, a saddle and a wall sconce all have named occupancy
positions and none of them wears anything.

- **`Slotted` = occupancy.** `getSlotNames` / `getSlotSpec` /
  `occupy` / `vacate` / `getOccupants` / `canOccupy`. Composed widely.
- **`Attired` = covering.** `wornStack`, `coveringAt`, `outermostAt`,
  `insulationAt`, `bodyInsulation`, `windproofing`,
  `concealmentOffset`, `attentionFactor`, `wouldLayerViolate` — plus
  the `worn` subscribable field and the dressed-impression line.
  Composed on **`Creature`** and nowhere else, which is why barding
  works and why a `Corpse` is still dressed.

`Attired` requires `Slotted` beneath it (the covering reads walk the
occupants), so `MixinApi.isAttired` narrows to
`Stuff & Slotted & Attired`. **Narrow on `isAttired`, not `isSlotted`,
before any covering read** — before the split those calls compiled
against a coat rack and answered zero at runtime; now they do not
compile at all.

⚠⚠ **A mixin's `static _mixinName` must WIDEN to `string`** — write
`static _mixinName = 'FooMixin';` and nothing else. Annotating it, or
initialising from the `as const` `Mixins` table, pins it to a literal
type, makes the class static side incompatible with the rest of the
chain, and — because `Base` is a type parameter, so the check is
deferred — reports as several hundred errors in unrelated files with
`AvatarBase` collapsed to `never`. This cost two sessions; see
the slate *(slate retired 2026-09-06 — fully absorbed)*.

## Slot universe — three patterns

The `getSlotNames` / `getSlotSpec` surface is **overridable**.
Consumers always call `host.getSlotNames()`; they never ask where
the universe came from.

- **Pattern A — Static slots (default).** Host declares
  `staticSlots: SlotSpec[]` on the class; the default
  `getSlotNames` reads it. Chairs, beds, saddles, the default floor.
- **Pattern B — Body-plan-driven.** `BodyPlanSlotsMixin` composes
  Slotted and overrides the universe surface to walk
  `species → bodyPlan → slots`. Avatars, NPCs, and any Organism
  whose slots flow from anatomy. The body-plan declaration lives in
  the `BodyPlan` template's `slots: SlotSpec[]` field (see
  [race.md](./race.md)).
- **Pattern C — Dynamic (Adornable retrofit).** `Adornable` composes
  Slotted and overrides the surface to derive from its live
  fixture-name keying. Locations carry fixtures (wall sconces,
  ceiling lamps, BoundaryAnchors, floor Adornments) through this
  surface.

A new host whose universe is computed differently ships a new
sibling mixin that overrides the surface. No change to `Slotted`,
no change to consumers.

## `canOccupy` algorithm — `accepts` + `fitsSlot`

Two-part check:

1. **Slot side.** `SlotSpec.accepts` names a `Mixins`-registry
   constant (`'WearableMixin'`, `'SlottableMixin'`,
   `'AdornmentMixin'`, …). The candidate's constructor must compose
   the named mixin.
2. **Candidate side.** Every Slottable implements
   `fitsSlot(host, slot): boolean`. `SlottableMixin` ships a default
   `() => true`; `WearableMixin` / `WieldableMixin` override it to
   walk their per-body-plan `slotClaims`. The candidate gets the
   final say — `canOccupy` returns its result after the slot-side
   mixin check passes.

Future affordance mixins ship a new mixin, register it in `Mixins`,
and slots that want it declare `accepts: 'NewMixin'`. No central
switch, no closed enum.

> ### ⚠ `fitsSlot` is a PLACEMENT policy — it must not veto a restore
>
> A `fitsSlot` that consults mutable world state (a capacity, a size, a
> volume) will eventually refuse a *legitimate re-seat*, because the
> persistence restore re-occupies slots by replaying the recorded occupancy —
> it is re-establishing an arrangement that already existed, not asking
> permission to make a new one. `Plant.fitsSlot` hit this exactly: it
> compares the plant's per-stage root demand against its pot's soil volume,
> so a **root-bound plant became unrestorable** — and root-bound is an
> ordinary designed state, not an error.
>
> The rule that resolves it: **a candidate already inside the host's contents
> always fits.** Verbs consult `fitsSlot` *before* they move anything, so the
> sizing gate still fires for a real placement; only the restore (where the
> occupant is already in the contents, put there by the container slice)
> reads permissive. Any new sizing-shaped `fitsSlot` wants the same shape.
> See [husbandry.md](./husbandry.md) § The verb surface.

## Foldable — the folded gate

`FoldableMixin` (`lib/slot/Foldable.ts`) is a two-state fold/unfold
capability for collapsible furniture — a folding camp chair, a card
table, a cot. Composed onto a posture-bearing host (a `Chair`), a
**folded host refuses its posture-bearing slots**: you can't sit on a
folded chair. The gate is **Part 0.5 of `canOccupy`** — it fires only
when the host `isFoldable`, is currently folded, and the slot is
posture-bearing (`spec.postures?.length > 0`), mirroring the
`fracture-impairs-slot` anatomy gate a few lines above. No verb knows
about folding; `fold`/`unfold` and `sit` compose because the occupancy
check owns the coupling.

Surface is deliberately narrow: `isFolded()` / `setFolded(value)` (the
predicate/setter contract pair, boolean-noun convention) plus the
idempotent action verbs `fold()` / `unfold()`. The guarded-boolean
storage delegates to `BistateMixin` (`lib/Bistate.ts`), the shared
substrate under `Sealable` (open/closed) and `Switchable` (on/off) —
Foldable is the folded-axis naming layer over `getState()` /
`setState()`. `folded` is a persistent field; `BistateMixin` itself
carries **no** `_mixinName` and is not in the `Mixins` registry (it's
shared implementation, not a queryable capability — consumers narrow on
`MixinApi.isFoldable`, never an `isBistate`).

Players drive it through the global **`device`-category** verbs
`fold <thing>` / `unfold <thing>` (`cmd/device/fold.yaml`,
`cmd/device/unfold.yaml`), which target a reachable object and are
narrowed to a Foldable by the controller (`MixinApi.isFoldable`,
`platform/idea/cmd/device/`). `FoldController` additionally refuses to fold a
host whose slots are occupied — you can't fold a chair someone is
sitting on. The concrete host is `FoldingChair` (`obj/FoldingChair.ts`
= `Chair` + `Foldable`, its own file so seeds resolve it by path); it
seeds **deployed** (`folded: false`). See [posture.md](./posture.md) for
the `Chair`/`sit` posture-slot side this gates.

## Capacity

`SlotSpec.capacity` defaults to 1. Authored values:

- `1` (default) — chairs, mount slots.
- `> 1` — benches (4), queen bed (2), and ⚠ **every covering slot**
  (`capacity: 4` on the biped's and quadruped's `head` / `torso` /
  `legs` / `feet` / `hands`). Worn-clothing slots used to take the
  default, which meant **layering was impossible**: one torso garment,
  full stop, so the shipped gambeson and hauberk could never be worn
  together and the covering stack had nothing to walk. Four is a real
  historical stack — shirt, gambeson, mail, surcoat — and it is a cap
  rather than unbounded because "wear forty shirts" would otherwise be
  free insulation. The ladder decides the ORDER; this decides the
  DEPTH.
- `UNBOUNDED_CAPACITY` (= `Number.MAX_SAFE_INTEGER`) — floor's
  `ground:1`. **Don't use `Infinity`** — it doesn't round-trip
  through JSON/BSON. The sentinel constant is JSON-safe and
  `isSlotFull` math stays trivial.

`isSlotFull(slot)` returns `count >= capacity`. `isSlotOccupied`
returns `count > 0`.

## ⭐ The wardrobe — `equip set`, and zero verbs of its own

`WardrobeMixin` (`lib/slot/Wardrobe.ts`, composed by `Avatar`) holds
named outfits as `wardrobes: Record<string, string[]>` — a set name to
an ordered **keyword** list, innermost-first.

**Why a mixin field**, stated because each alternative is wrong for a
different reason: not a Mongo collection (forbidden, and it rides the
Avatar's existing `holder_snapshots` capture for free); not a
`Property` (a prop is for a slot whose *key* is computed at runtime,
and this field is narrowed on and Hydrator-reflected); not an
`EnvironmentMixin` setting (fixed keyspace — wardrobes are whatever the
player calls them). It is byte-identical in shape to
`Wearable.slotClaims`, and it is the persistent-fields doctrine's named
**variable-key** escape hatch — the exact contrast to the fit stamp's
three fixed scalars.

⭐ **Keywords, not instance refs.** A saved set survives buying a
replacement shirt (the new one answers to the same word), a keyword
resolving to nothing is **skipped with a readable line** rather than
dangling, and there is no lifetime relationship to maintain.

**Replay** dresses in the saved order, which is wear order, which is
innermost-first — so a saved set never trips the covering ladder's
refusal. ⚠ **Failures are per-item and non-fatal**: a dressing mistake
has to be survivable and readable, and that starts here.

### ⚠ It is a STANZA, not a verb

`equip set <name>` (with `--save`) and `equip sets` are `subcommands:`
on `cmd/inventory/equip.yaml`, alongside its existing `args:` — the
`measure strike` precedent, second instance. `fallthrough: true` is what
keeps bare `equip <item>` working once the verb has subcommands: an
unrecognised first token binds against `args:` instead of erroring.

⚠ It shipped as `wear set` and **moved to `equip` on the same branch**,
when the orchestrator landed. The stanza is unchanged; it simply rides
the verb that owns getting your whole kit on. See
equip-slate.md *(slate retired 2026-09-06 — fully absorbed)*.

⚠ **The requirement said *"`dress` is not taken"* and that premise was
stale.** `medical/treat.yaml` has shipped `verbs: [treat, bind, dress]`
since the medic build, where it means *dressing a wound*. So "unclaimed"
was already false and the constraint as written is unsatisfiable. What
it protects is checkable and is what the source-shape test asserts:
**the wardrobe adds no verb**, so `dress` still resolves to exactly one
view and that view is medical's.

## Mutation surface

- `occupy(candidate, slot)` — throws on unknown slot, full slot,
  double-occupy, or type mismatch.
- `vacate(slot, candidate)` — removes a specific occupant; returns
  null if the candidate wasn't present. Throws on unknown slot.
- `vacateSole(slot)` — convenience for single-capacity slots;
  throws if multi-occupant.
- `occupyAll(host, candidate, slots)` — multi-slot atomic
  claim with rollback on partial failure.
- `transferOccupancy(candidate, from, to)` — vacate-then-
  occupy with rollback. Used by every posture verb to swap the
  actor's posture-bearing slot atomically.
- **`tryReleaseFromSlots(item)`** — take `item` off this host
  entirely, every slot it occupies, as **one decision**. Returns
  `{released: false, dumpedTau}` (the occupant refused; **nothing** was
  vacated) or `{released: true, vacated}` (`0` = it was not in any slot,
  which is a no-op rather than a failure).

  **Every verb that takes something out of a slot goes through this** —
  `remove`, `unwield`, `drop`, `give`, `put`, and (TPA reform) `get`,
  which is the first non-body caller. It exists because leaving
  a body is one event several verbs perform, and only the first two used
  to vacate at all: `drop` moved the Stuff and left the slot claiming
  it, so **a dropped sword lay on the floor while the hand stayed full
  forever**. Reproducible with any shipped weapon.

  The refusal is the *occupant's* to make — the `canEvict` shape, where
  the engine asks and the object answers, defaulting to permission. v1
  refuser: a cursed `Blessable` (see
  [magic-items.md](./magic-items.md) § Cursed sticks). The slot
  substrate does **not** import the magic tree; it asks through
  `MixinApi.isBlessable`, so a slot knows only that occupants may
  refuse, never what a curse is. All-or-nothing across the item's slots,
  so a two-handed cursed thing cannot end up half off.

## ⭐ The verb surface — `put` / `get` reach a non-body slot

> Before the TPA reform **nothing in the game could put anything into a
> non-body slot by any verb.**

`equip`/`wear`/`wield` are body slots, `plant`/`repot` the plant slot,
`mount` is conveyance, and the whole `device` category (`arm · disarm ·
douse · fold · ignite · pump · switch · unfold`) drives no slot at all.
So a battery bay, a lamp's oil reservoir and a mill's replaceable stone
were all authorable and none of them fillable. That is a hole in this
substrate, not a requirement of one build, so `put`/`get` were extended
once and every slot-bearing fixture anyone authors gets it.

**`put X in Y`** takes the slot branch when three things hold, each
load-bearing:

1. **`Y` is not a body** (`!isVitals`) — dressing someone else is not
   `put`, and the two verbs that do body slots already exist.
2. **`Y` is a `Container`** — a part inside a machine has to physically
   BE somewhere. Occupancy is not containment, so **contents move
   first and the slot claims second**, exactly the `plant`-into-a-pot
   order. A `Slotted` host that is not a `Container` has nowhere to put
   the part and says so.
3. **Some open slot accepts `X`** — *a slot is more specific than a
   container.* An item that fits the bay goes in the bay; one that does
   not is ordinary containment.

⭐ Condition 3 is why **no existing target changed behaviour**: a seed
still just goes in the pot, because a seed is not a `Plant`. `on` is
never a slot (`ContainmentApi.placeOn` owns surfaces).

**`get X`** is the reverse and needed exactly one thing: vacate the slot
before the move, through `tryReleaseFromSlots` — otherwise the bay stays
full of a cell that is now in somebody's pocket. The occupant's refusal
(a cursed thing) is honoured, and it is all-or-nothing across the host's
slots. The actor's own body is skipped: taking your own worn thing is
`remove`.

## Lifecycle

- `Slotted.onDestruct()` clears every slot's occupant set before
  chaining to super. Vacated occupants do NOT auto-destruct — they
  detach.
- `Slottable.onDestruct()` walks every host the candidate is slotted
  into and vacates from each slot, before chaining. Guarantees no
  host holds a stale reference to a destructed Stuff.
- `Slottable.onSlotReleased?(host, slotName)` — optional witness fired
  synchronously by `Slotted.vacate(slot, candidate)` and
  `Slotted.vacateSole(slot)` immediately after the candidate is
  removed from the occupant set. v1's `Mobile.onSlotReleased` clears
  `engagedMode` for passthrough modes (ride / drive) so a dismounting
  rider's engagement clears automatically. The invocation lives in
  `Slotted.vacate` so direct callers like
  `DismountController.execute` also trigger it. Future witnesses
  (polymorph revert on dismount, status-clear, etc.) compose the same
  optional-method shape.

## Detail-targeted resolution

Slots have **canonical internal names** (`hand:left`, `mount:1`,
`sit:1`) and **optional user-facing detail keywords** (`back`,
`seat`, `floor`). The internal name keys the slot map; the detail
keyword is the noun the player types.

A slot's `userFacingDetail` references a keyword on the host's
`DetailedMixin` map. `resolveSlot(host, { detail: 'back' })`
returns the slot name whose `userFacingDetail` matches.

```yaml
# Horse — back:1 has userFacingDetail 'back'; the horse's
# DetailedMixin defines a 'back' detail too.
slots:
  - { name: 'back:1', accepts: 'SlottableMixin', userFacingDetail: 'back' }
```

`mount horse` finds `back:1` via host resolution (the only mount-
eligible slot). `mount back` resolves `back` as a Detail keyword,
then `resolveSlot({ detail: 'back' })` finds the slot.
Same slot, two paths.

`setStaticSlots` validates that no two slots on the same host share
a `userFacingDetail`.

## ⭐⭐ The occupancy back-reference — the candidate knows its host

Occupancy is recorded on **both** sides, and the two sides are not
symmetric in status:

- **`Slotted.slots`** — the host's forward map, slot name → occupants.
  This is the side that is authoritative and the side that restores.
- **`Slottable._occupancy`** — the candidate's private `Map<host, Set<
  slot>>`. **Runtime only**, rebuilt by the same `occupy` calls that
  restore the forward map.

⭐ It exists because *which host holds me?* is a **reverse-relational**
question, and MQL has no predicate for one. Answering it used to mean
reading every object in the world and looking inside each one's slots —
on the metabolism path, once per creature per tick, where a live drive
found it pinning a CPU core. The back-reference makes the same question
a map lookup.

Both writes are gated by a **participant contract**, not `ApiOnly`:
`FromMixin('SlottedMixin', { where: caller is args[0] })` — the host
that is the other party to the occupancy, writing *itself* in. The
`@Final @Unshadowable` pair is what seals it.

⚠ **Deliberately not declared in `fieldMeta`**, exactly as its forward
twin `Slotted.slots` is not: neither side persists, so neither is a
live-ref field the R2.1–R2.4 rules govern. Both sides are already
cleared by destruct — `Slottable.cleanupOnDestruct` vacates the
candidate from every host, `Slotted.cleanupOnDestruct` vacates every
occupant from the host — and both routes go through `vacate`, which is
what drops the entry.

## Slotted/Slottable reference

Verbs on objects — the host answers host questions, the candidate
answers candidate questions.

**On the host (`Slotted`):**

| Method | Purpose |
|---|---|
| `occupy(candidate, slot)` / `vacate(slot, candidate)` | The two chokepoints; every occupancy change goes through one of them, and each maintains the candidate's back-reference |
| `occupyAll(candidate, slots)` / `vacateAll(candidate, slots)` | Multi-slot transactional claim / release |
| `findOpenSlotFor(candidate)` | First open compatible slot (single-slot candidates) |
| `getOccupant(slot)` / `getOccupants(slot)` / `getAllOccupants()` | Who is in a slot |
| `isSlotOccupied(slot)` / `isSlotFull(slot)` / `getOccupantCount(slot)` | Slot state |
| `resolveSlot({ detail \| accepts })` | Slot-name resolution |
| `walkOccupants(visit)` | Recursive walker; once-per-unique-occupant |
| `getSlotNames()` / `getSlotSpec(name)` | The slot universe |

**On the candidate (`Slottable`):**

| Method | Purpose |
|---|---|
| `occupiedSlots()` | Full host → slots map, read off the back-reference. A fresh Map with copied slot lists each call — `cleanupOnDestruct` iterates it while vacating |
| `getOccupiedHost()` | The single host, or null; **throws** on multi-host (only Wearable's multi-claim case reaches it, and a gauntlet on each hand of one wearer is still one host) |
| `transferOccupancy(from, to)` | Atomic vacate-then-occupy with rollback; every posture verb swaps the posture-bearing slot through it |
| `fitsSlot(host, slot)` | The candidate-side acceptance test `canOccupy` consults |
| `_noteOccupied` / `_noteReleased` | ⚠ Not caller surface — the host's write into the back-reference, gated to that host |

## Wear / wield / mount failure notes

The slot-claiming verbs (`equip`/`wear`/`wield`, and `mount`) emit a
`slot-occupied { host: StuffRef, slot: string, occupant?: StuffRef }`
note onto the dispatch context when the required slot is already
taken. `host` identifies who owns the slot — the actor for the dressing
verbs, the mount target for `mount`. `slot` is the canonical
body-plan slot name (`'hand:left'`, `'mount:1'`, …). `occupant` is
the current occupant when known, omitted otherwise.

The note rides through the dispatcher's standard auto-escalation
(`declined`) — see
[response-envelope.md § Notes](./response-envelope.md). The
controller's accompanying `Scene.send` carries the human-readable
prose; the note is the machine signal for clients that want to
surface a "your left hand is full (holding the dagger)" affordance
without re-parsing prose.

## Cross-references

- [response-envelope.md](./response-envelope.md) — `slot-occupied`
  note shape; wear / wield / mount audit.
- [embodiment.md](./embodiment.md) — Wearable / Wieldable, body-side
  affordances.
- [posture.md](./posture.md) — Postured + Posed + floor adornments.
- [conveyance.md](./conveyance.md) — Mountable / Drivable + ripple.
- [boundary.md](./boundary.md) — Adornable (Pattern C consumer).
- [race.md](./race.md) — BodyPlan's `slots: SlotSpec[]`.
- [command-spec.md](./command-spec.md) — `default:` field used by
  posture verbs for the `ground` fallback.

## History

**The occupancy back-reference landed** in the world-scan build
(`design/world-scan`, W3). Before it, `occupiedSlots()` answered *which
host holds me?* by walking the whole object registry and looking inside
every `Slotted`'s slots; the metabolism path asked it once per creature
per tick. The forward map on the host is unchanged and still the
persisted side — see § *The occupancy back-reference* above, and
[mql.md § The registry-read grant](./mql.md) for why a reverse-relational
question has no query to fall back on.

**Foldable landed** in the University Avenue crossing build
(`feature/university-avenue-crossing`, commits `ab0867ba` Phase 1B–1D),
introducing the two-state fold/unfold capability, the `FoldingChair`
host, and the `device`-category `fold`/`unfold` verbs. The camp chair —
the crossing guard's relief's chair, seeded deployed and never folded in
practice ("Gus never folds it" is behavior over a real capability, per
the build's no-dead-props law) — was the first driver. Its
guarded-boolean storage was later hoisted onto the shared `BistateMixin`
base (`lib/Bistate.ts`) alongside `Sealable`/`Switchable` in commit
`a99eccb4`.


## History — the furnishing build (2026-07-31)

> ⚠ **A witness implemented by more than one composed layer must
> super-chain.** `onSlotReleased` is implemented by both `MobileMixin`
> (engagedMode) and `PosedMixin` (forgetting which host you rested in). A
> method does not merge — the **outermost** composed layer wins, and
> `Mobile` (in `Character`) sits outside `Posed` (in `Creature`), so Posed's
> witness was silently replaced. Standing up left the sleeper recorded as
> still occupying the bed. `Mobile.onSlotReleased` now calls the parent
> first; any future implementor must do the same.

**`Slotted.occupy` gained `onSlotOccupied`** — the symmetric twin of the
shipped `onSlotReleased`, firing from the same chokepoint so every arming
path reaches it (`occupyAll`, combat's grip swap, persistence
restore). Declared `@hook` on `Slottable`. v1 consumer: `PosedMixin`
records which host's posture slot a body occupies, so an avatar wakes where
it slept — see [furnishing.md](./furnishing.md).
