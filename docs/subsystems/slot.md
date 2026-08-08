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
| `Slottable` | `lib/slot/Slottable.ts` | Marker mixin — anything that can sit in a slot. Carries `fitsSlot(host, slot)` with a default `() => true`; Wearable/Wieldable override |
| `SlotSpec` | `lib/slot/Slotted.ts` | Per-slot declaration (name, accepts, capacity, postures, userFacingDetail, `bodyPart`, `covers`). `bodyPart`/`covers` are optional `body.*` references to anatomy (Vitals) — a slot is its own axis that *references* anatomy where it has a home; see [vitals.md](./vitals.md) |
| `UNBOUNDED_CAPACITY` | `lib/slot/Slotted.ts` | Sentinel = `Number.MAX_SAFE_INTEGER`; JSON/BSON-safe substitute for `Infinity` |
| `BodyPlanSlotsMixin` | `lib/slot/BodyPlanSlots.ts` | Sibling provider — Pattern B, derives universe from species → bodyPlan |
| `Foldable` | `lib/slot/Foldable.ts` | Host capability — two-state fold/unfold; a folded host refuses its posture-bearing slots (gates `canOccupy`) |
| `SlotApi` | `api/slot.ts` | Cross-cutting helpers (multi-slot transactional ops, inverse lookups, slot resolution, conveyance ripple walker) |

`Slotted` composes on `Stuff` (no `Container` prereq). Composing
`Slottable` doesn't constrain a host — it just marks a Stuff as
slot-occupant-eligible.

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
`obj/command/device/`). `FoldController` additionally refuses to fold a
host whose slots are occupied — you can't fold a chair someone is
sitting on. The concrete host is `FoldingChair` (`obj/FoldingChair.ts`
= `Chair` + `Foldable`, its own file so seeds resolve it by path); it
seeds **deployed** (`folded: false`). See [posture.md](./posture.md) for
the `Chair`/`sit` posture-slot side this gates.

## Capacity

`SlotSpec.capacity` defaults to 1. Authored values:

- `1` (default) — chairs, mount slots, worn-clothing slots.
- `> 1` — benches (4), queen bed (2).
- `UNBOUNDED_CAPACITY` (= `Number.MAX_SAFE_INTEGER`) — floor's
  `ground:1`. **Don't use `Infinity`** — it doesn't round-trip
  through JSON/BSON. The sentinel constant is JSON-safe and
  `isSlotFull` math stays trivial.

`isSlotFull(slot)` returns `count >= capacity`. `isSlotOccupied`
returns `count > 0`.

## Mutation surface

- `occupy(candidate, slot)` — throws on unknown slot, full slot,
  double-occupy, or type mismatch.
- `vacate(slot, candidate)` — removes a specific occupant; returns
  null if the candidate wasn't present. Throws on unknown slot.
- `vacateSole(slot)` — convenience for single-capacity slots;
  throws if multi-occupant.
- `SlotApi.occupyAll(host, candidate, slots)` — multi-slot atomic
  claim with rollback on partial failure.
- `SlotApi.transferOccupancy(candidate, from, to)` — vacate-then-
  occupy with rollback. Used by every posture verb to swap the
  actor's posture-bearing slot atomically.
- **`tryReleaseFromSlots(item)`** — take `item` off this host
  entirely, every slot it occupies, as **one decision**. Returns
  `{released: false, dumpedKJ}` (the occupant refused; **nothing** was
  vacated) or `{released: true, vacated}` (`0` = it was not in any slot,
  which is a no-op rather than a failure).

  **Every verb that removes something from a body goes through this** —
  `remove`, `unwield`, `drop`, `give`, `put`. It exists because leaving
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
  `Slotted.vacate` (not `SlotApi.vacate`) so direct callers like
  `DismountController.execute` also trigger it. Future witnesses
  (polymorph revert on dismount, status-clear, etc.) compose the same
  optional-method shape.

## Detail-targeted resolution

Slots have **canonical internal names** (`hand:left`, `mount:1`,
`sit:1`) and **optional user-facing detail keywords** (`back`,
`seat`, `floor`). The internal name keys the slot map; the detail
keyword is the noun the player types.

A slot's `userFacingDetail` references a keyword on the host's
`DetailedMixin` map. `SlotApi.resolveSlot(host, { detail: 'back' })`
returns the slot name whose `userFacingDetail` matches.

```yaml
# Horse — back:1 has userFacingDetail 'back'; the horse's
# DetailedMixin defines a 'back' detail too.
slots:
  - { name: 'back:1', accepts: 'SlottableMixin', userFacingDetail: 'back' }
```

`mount horse` finds `back:1` via host resolution (the only mount-
eligible slot). `mount back` resolves `back` as a Detail keyword,
then `SlotApi.resolveSlot({ detail: 'back' })` finds the slot.
Same slot, two paths.

`setStaticSlots` validates that no two slots on the same host share
a `userFacingDetail`.

## SlotApi reference

| Method | Purpose |
|---|---|
| `occupyAll(host, candidate, slots)` | Multi-slot transactional claim |
| `vacateAll(host, candidate, slots)` | Multi-slot vacate |
| `findOpenSlotFor(host, candidate)` | First open compatible slot (single-slot candidates) |
| `findOccupiedHost(candidate)` | Single host or null; throws on multi-host |
| `findOccupiedSlots(candidate)` | Full host → slots map |
| `resolveSlot(host, { detail \| accepts })` | Slot-name resolution |
| `walkOccupants(root, visit)` | Recursive walker; once-per-unique-occupant |
| `transferOccupancy(candidate, from, to)` | Atomic vacate-then-occupy with rollback |

`Slottable.getOccupiedHost()` delegates to `findOccupiedHost`. The
two-step API is "use the convenience instance method by default;
go through SlotApi when you need the verbose form."

## Wear / wield / mount failure notes

The slot-claiming verbs (`wear`, `wield`, `mount`) emit a
`slot-occupied { host: StuffRef, slot: string, occupant?: StuffRef }`
note onto the dispatch context when the required slot is already
taken. `host` identifies who owns the slot — the actor for `wear` /
`wield`, the mount target for `mount`. `slot` is the canonical
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
path reaches it (`SlotApi.occupyAll`, combat's grip swap, persistence
restore). Declared `@hook` on `Slottable`. v1 consumer: `PosedMixin`
records which host's posture slot a body occupies, so an avatar wakes where
it slept — see [furnishing.md](./furnishing.md).
