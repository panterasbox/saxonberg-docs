# Conveyance

`Mountable` and `Drivable` mixins; the `mount` / `dismount` verbs;
the conveyance ripple in `Mobile.traverse` that takes occupants
along when their host moves.

## The Cast

| Name | Location | Role |
|---|---|---|
| `Mountable` | `lib/slot/Mountable.ts` | Single mount slot occupant can ride |
| `Drivable` | `lib/slot/Drivable.ts` | Has a controller slot (driver position) |
| `SeatedDrivableMixin` | `lib/slot/Drivable.ts` | Sibling override — controller slot lives on a Containable seat |
| `SlotRef` | `lib/slot/Drivable.ts` | Internal `(host, slotName)` pair; not inter-Stuff contract |

Both compose on `Stuff & Slotted`. `SeatedDrivableMixin` composes
on `Stuff & Drivable & Container`.

## Mountable

Persistent: `mountSlot: string` — defaults to `'mount:1'`. A horse
may use `'back:1'` resolved from its bodyPlan.

Public surface:

- `getMountSlot()` / `setMountSlot(value)` — accessor pair.
- `isMounted()` — slot occupancy.
- `getMountOccupant()` — the rider (or null).

The mount slot itself is registered on the Slotted substrate
(static or body-plan-driven). Mountable doesn't add the slot — it
points at it.

## Drivable

Public surface (interface):

- `isDriven(): boolean`
- `getDriver(): (Stuff & Slottable) | null`
- `getVehicularMode() / setVehicularMode(LocomotionMode | null)` —
  identity ref (path-by-string) for the `LocomotionMode` this
  conveyance engages when driven. Authoring is required:
  `LocomotionApi.resolveHostMode` throws on a driven host with `null`
  vehicularMode rather than silently walk-traversing. See
  [locomotion.md](./locomotion.md#drivablevehicularmode--fail-loud).

The slot-resolution logic is a `protected resolveControllerSlot(): SlotRef`
extension method on the class — not part of the public Drivable
interface (TypeScript interfaces can't declare protected members).

The default `DrivableMixin.resolveControllerSlot` returns
`{ host: this, name: this.controllerSlot }`. `SeatedDrivableMixin`
overrides to find a `role: 'driver'` Containable in the host's
contents.

The field accessor pair `getControllerSlot(): string` /
`setControllerSlot(value)` exists for the persistent-field surface
and is intentionally distinct from the protected
`resolveControllerSlot()` method (they used to share a name; the
collision has been resolved per resolved-decision #4).

The default `controllerSlot` is `'driver:1'` (changed from `'mount:1'`
during the locomotion build to avoid colliding with
`Mountable.mountSlot`'s default for hosts that compose both).

## Vehicle design space coverage

| Vehicle | Pattern | v1 |
|---|---|---|
| Horse, bicycle, motorcycle, magic carpet | Default DrivableMixin (no override) | ✓ |
| Two-seater car, sedan, bus | `SeatedDrivableMixin` override | ✓ |
| Carriage with driver-on-box | Override variant (driver platform is an Adornment) | ✓ |
| Pillion motorcycle | Override + sibling Mountable slot | ✓ |
| Tank, helicopter, sailboat (multi-controller) | `controllerSlot` plural | ✗ — own slate |
| Sailboat with crew (multi-actor coordination) | Beyond Drivable | ✗ |
| Rickshaw, palanquin (driver external) | Different mixin | ✗ |

## Conveyance ripple

Lives in `Mobile.traverse` (not in Mountable / Drivable). After
`ContainmentApi.move(mover, destination)` and `announceArrival`,
the host walks the immediate level of its own slot map and ripples
each occupant according to its capability:

```ts
if (MixinApi.isSlotted(mover)) {
  const seen = new Set<Stuff>();
  for (const [, occupants] of mover.getAllOccupants().entries()) {
    for (const occupant of occupants) {
      if (seen.has(occupant)) continue;
      seen.add(occupant);
      if (MixinApi.isMobile(occupant) && MixinApi.isContainable(occupant)) {
        try {
          await occupant.traverse(exit, mode);
        } catch (err) {
          // rider whose canTraverse vetoes is left behind silently
        }
      } else if (MixinApi.isContainable(occupant)) {
        ContainmentApi.move(occupant, destination);
      }
    }
  }
}
```

**Mobile vs. inert occupants.** A rider on a horse is itself a
`Mobile` — calling its `traverse(exit, mode)` produces the rider's
own arrival narration ("Joe arrives from the west") and, because
`traverse` runs the same ripple, recursively ripples whatever the
rider has in its own slots (so the saddle-on-horse-with-rider-and-
backpack chain just works without an explicit recursion here).
Inert occupants (a saddlebag fixture, a crash-test dummy strapped
to a bicycle) fall back to `ContainmentApi.move` so they ride
along silently — the container model.

**Veto handling.** If a rider's `canTraverse` veto throws, the
ripple swallows the error and the rider stays behind. The host's
traverse never aborts because of a rider's refusal.

**Once-per-occupant within a host.** The local `seen` set
deduplicates a Wearable claiming two slots on the same mover
(boots on foot:left + foot:right ripple once).

**Cycle guard.** The recursive depth is bounded by `traverse`'s
own per-call cost; the slot-occupancy graph is a forest in practice
(a Stuff belongs to one host-of-hosts chain at a time). Saddle-on-
saddle abuse would surface as an excessively deep call stack —
the explicit depth limit was dropped along with the recursive walker
because the per-host single-level walk has nowhere to recurse into
that `traverse()` doesn't itself bound.

**Ride-vs-walk message distinction.** v1 produces the same "arrives
from the west" message regardless of how the occupant got there.
Mode-specific phrasing ("rides" / "walks" / "rolls") consumes
`LocomotionApi.emissionAt(mover)` — the substrate is in place; the
narration polish lands separately.

**Slot-release witness clears engagement.** When a rider dismounts
(`Slotted.vacate` on the mount slot), the `Slottable.onSlotReleased`
witness fires synchronously and `Mobile.onSlotReleased` clears the
rider's `engagedMode` when the vacated host composes the engaged
mode's `conveyanceMixin`. The witness invocation lives inside
`Slotted.vacate` itself so direct callers like
`DismountController.execute` also trigger it. See
[locomotion.md](./locomotion.md#slot-release-witness).

## Verbs

| Verb | Action |
|---|---|
| `mount <X>` | `transferOccupancy` from any current posture-bearing slot to `X.mountSlot`; set `Postures.Mounted` |
| `dismount` | Vacate the mount slot the actor occupies; set `Postures.Stand` |

`mount horse` finds the horse's mountSlot; `mount back` resolves
`back` as a Detail keyword via the § 5.5 Detail-targeted pathway.

## Haulage — pulling a cart

The **inverse** of conveyance: instead of the actor riding *in* the host,
a wheeled `Vessel` (a cart) follows the actor on the ground while staying
a room object. The cost model — the `draftFactor` term folded into
encumbrance, and the move-time gates — lives in
[encumbrance.md § Haulage](./encumbrance.md#haulage--the-cart-the-draft-term);
this is the **relationship**: the coupling, the tow, and the verbs.

**Two mixins** (`lib/slot/`, the conveyance family):

| Name | Role |
|---|---|
| `HaulableMixin` | Cart side, on `Vessel`. Carries `draftFactor` / `handedness` / `passageMode` (the `wheeled` default), `getDraftLoad()`, and the `_hauledBy` back-ref. |
| `HaulerMixin` | Hauler side. Holds the `_hauling` live ref; `hitch` / `unhitch` / `getHauledCart` / `isHitched` / `getHaulDraft`. |

**The coupling is a Pattern-B live ref** — a symmetric **R2.2** pair
(`hauler._hauling` ↔ `cart._hauledBy`) with an **R2.3** self-heal getter
on each side; `hitch`/`unhitch` keep both sides atomic, and either
side's `onDestruct` clears the back-ref. The ref is runtime-only (a
reloaded hauler wakes unhitched). See [ref-shapes.md](../ref-shapes.md).

**Where the capability lands.** `HaulerMixin` composes on **`Character`**
(every PC + NPC-character can pull a cart) — **not** the broad `Creature`
base (a frog / corpse never hauls; the "compose the mixin, not the class
tree" rule). The cart side is `Handcart = HaulableMixin(Vessel)`
(`lib/equipment/`). Draft beasts use **`HaulingCreature`** =
`MountableMixin(PostRegistration(Character))` — a rideable creature that
inherits hauling from `Character` and adds the rider slot.

**The tow** rides the **conveyance ripple region of `Mobile.traverse`**
(alongside the slot-occupant ripple, but keyed separately — the cart is a
room object, not a slot occupant): after the mover moves, if it
`isHauling`, its cart is `ContainmentApi.move`d to the destination,
carrying its cargo as a unit. No encumbrance is read here — the raw move
stays encumbrance-free.

**Animal-hauling composes cleanly** because the three couplings are
orthogonal axes: `player —(mount slot)→ horse —(hauling ref)→ cart
—(container)→ cargo`. You mount the horse (existing mount slot); the horse
hitches the cart (`hitch <cart> to <horse>`); the horse bears the draft on
*its* gauge, you bear nothing. When the **horse** traverses, the ripple
carries you and the tow carries the cart.

> **Ridden-gate seam.** The move-time gates (terrain / breakaway) fire on
> the **host's own self-powered traverse** (the horse is the hauler +
> traverser). A rider-issued `ride <dir>` does *not* reach the haulage
> gate — `exit.canTraverse` rejects passthrough modes at the media gate
> first (a pre-existing locomotion limitation; see
> [locomotion.md](./locomotion.md)). Wiring the gate into ride/drive
> dispatch waits on that fix; the tow and the host-side gate are correct
> and tested.

### Haulage verbs

| Verb | Action |
|---|---|
| `hitch <cart>` | The giver becomes the hauler (self-haul). Hands fill up (can't wield / loose-carry while hauling — guarded in `Wield`/`GetController`, keyed on `isHauling(giver)`). |
| `hitch <cart> to <mount>` | Harness a draft creature (`HaulerMixin`) as the hauler instead; the rider's hands stay free. |
| `unhitch [target]` | Bare = release the giver's own cart; `unhitch <mount>` / `<cart>` releases that one. |

Hitching never fails on weight (you can always grab a handle); the cost
surfaces at move time. Validators `mustBeHaulable` / `mustBeHauler`.

## What v1 doesn't cover

- **Multi-controller vehicles** (tank, helicopter, sailboat) —
  `controllerSlot` is scalar; plural ships when shared-control
  arrives.
- **Driver-external vehicles** (rickshaw, palanquin, dog sled) —
  driver isn't IN the conveyance; needs a different mixin.
- **Multi-actor coordination** (sailboat with crew) — beyond
  Drivable; activity-slate territory.

### ⚠ Known defect: `teleport` does not ripple

**The conveyance ripple and the haulage tow both live in
`Mobile.traverse` only.** `Mobile.teleport` is a bare
`ContainmentApi.move` plus narration, so today:

- **teleport while mounted → the mount stays behind**;
- **teleport while hitched → the cart stays behind**, silently.

This is an **oversight, not a policy** (surfaced 2026-07-31 by the
"can freight ride the TPA?" question — see
freight-slate § *The TPA
question*). The intended rule:

> **Teleport ripples what is *on* you and refuses what you are
> *attached to* — and says why.**

Worn gear and carried contents come along (otherwise teleport strips
the traveller); being **mounted or hitched refuses the ride** with an
honest message, per the enforcement slate's wall-mode-honesty rail.
**Silent failure is the part to kill.**

Note that this is a *ripple* defect only — it is **not** what keeps
vehicles off the TPA network. Two independent gates already do that:
`teleport` is verb-gated `requiresAnimate`, and travel clearance lives
in an **aether-hosted** wallet a vehicle cannot host
([fasttravel.md](./fasttravel.md) § *Identity-bound clearance*).

## Cross-references

- [slot.md](./slot.md) — substrate, Detail-targeted resolution.
- [posture.md](./posture.md) — `Postures.Mounted` integration.
- [spatial.md](./spatial.md) — `Mobile.traverse` integration.
