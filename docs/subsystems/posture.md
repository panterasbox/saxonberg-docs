# Posture

`Postured` (host side) + `Posed` (actor side) + the `Postures`
constants vocabulary. Verb suite: `sit`, `lie`, `kneel`, `stand`.

## The Cast

| Name | Location | Role |
|---|---|---|
| `Postured` | `lib/slot/Postured.ts` | Host capability — exposes posture-bearing slots |
| `Posed` | `lib/character/Posed.ts` | Actor capability — carries `getPosture()` / `setPosture()` |
| `Postures` | `lib/slot/Postured.ts` | Frozen const-object: `Stand`, `Sit`, `Lie`, `Kneel`, `Mounted` |
| `Posture` | `lib/slot/Postured.ts` | Derived type union of `Postures` values |
| `PostureApi` | `api/posture.ts` | Slot+state mechanics (`transferPosture`, `vacatePostureBearingSlots`, `findCurrentPostureBearingSlot`). Pure mechanism — no messaging; callers (controllers) own the narration |

`PosturedMixin` composes on `Stuff & Slotted`. `PosedMixin` composes
on `Stuff` and is composed by `Character`, so every PC and NPC
carries posture state uniformly.

### `restQuality` (the rest multiplier)

`PosturedMixin` carries **`restQuality: number`** (default `1.0`, strict
`> 0` setter, persistent) — how much better than open ground this host is
for recovery. [Metabolism](./metabolism.md)'s coupled recovery reads it
off the host whose posture slot the body occupies
(`getOccupiedHost()?.getRestQuality() ?? 1.0`): the floor / standing is
`1.0`, a bedroll ~1.3×, a four-poster ~2.5× (authored on the furniture
template). It is **not on `SlotSpec`** — the universal slot record stays a
pure structural mechanism; `restQuality` is a behavior field on the
specialized posture-bearing host, the `Vessel.transmissionFactor` pattern
(the same call the encumbrance build made for its coupling factor).

The thermal build added a sibling field on the same host —
**`warmth`** (default `0`), an additive-Kelvin bump to a slot
occupant's effective ambient (a campfire log-seat authors both
`restQuality` and `warmth`). See [thermal.md](./thermal.md).

## Posture-bearing slot

Definition: a slot is *posture-bearing* iff its `SlotSpec` declares
`postures: string[]` non-empty. Worn / held / mount / fixture slots
are NOT posture-bearing. Sit / lie / kneel / stand-on / floor slots
are.

```yaml
# Bed: lie:1 accepts both 'lie' and 'sit'
staticSlots:
  - { name: 'lie:1', accepts: 'SlottableMixin', postures: ['lie', 'sit'] }
```

## Posture vocabulary

```ts
export const Postures = Object.freeze({
  Stand:   'stand',
  Sit:     'sit',
  Lie:     'lie',
  Kneel:   'kneel',
  Mounted: 'mounted',
} as const);
```

Verbs and validators import the constant; slot YAML uses raw
strings. Substrate has no opinion on vocabulary — typo'd values
surface at use time when no verb's posture matches the slot's
`postures` entries. Mod-introduced postures ship their own
constants module.

## Floor adornments

Floors are first-class entities — `Adornment`s on the Location's
`Adornable` surface, composing `Postured`. v1 ships no class-level
default; floor presence is authored per-Location.

The default-floor template (`/idea/surface/default-floor`) declares
one slot:

```yaml
- name: 'ground:1'
  accepts: SlottableMixin
  capacity: 9007199254740991  # UNBOUNDED_CAPACITY (JSON-safe)
  postures: [sit, lie, kneel, stand]
  userFacingDetail: floor
```

Per-Location authoring:

```yaml
# Default Location
adornments:
  floor: { extends: '/idea/surface/default-floor' }

# Lava chamber — only stand
adornments:
  floor:
    extends: '/idea/surface/default-floor'
    staticSlots:
      - { name: 'ground:1', accepts: 'SlottableMixin',
          postures: ['stand'], userFacingDetail: 'floor' }

# Void — no floor adornment at all (sit no-arg fails honestly)
```

The `noDefaultFloor: true` opt-out marker on a Location's seed
suppresses the migration script and any future auto-floor tooling.

## Verbs

The four with-arg posture verbs (`sit X`, `lie X`, `kneel X`,
`stand X`) and `stand X` call `PostureApi.transferPosture(actor,
target, posture, verb)` for the slot+state mechanics; the
controller emits the verb-specific message scene on success. The
asymmetric `stand` no-arg form calls
`PostureApi.vacatePostureBearingSlots` directly (no occupy step —
posture flips to `Stand`, the actor stays free of any slot) and
fires its own "you stand up" scene.

Layering: PostureApi is pure mechanism, no `MessageApi.send`
inside. Narration is verb-specific surface so it lives in the
controller, matching how MountController, GetController,
DropController etc. work. The exception in the codebase is the
movement subsystem, where the message text is settings-driven and
the substrate (`MobileMixin`) owns the send — posture verbs are
the standard non-configurable case.

| Verb | Form | Action |
|---|---|---|
| `sit` | no-arg | `default: 'ground'` → resolves to floor's `ground:1` |
| `sit <X>` | with arg | First slot on X accepting `Postures.Sit` |
| `lie` / `lie <X>` | both | Same shape, `Postures.Lie` |
| `kneel` | no-arg only | `default: 'ground'`, `Postures.Kneel` |
| `stand` | no-arg | Slot-less posture flip — vacates any posture-bearing slot, sets `Stand` |
| `stand <X>` | with arg | Stand on X (e.g., on a chair or table) |

The asymmetry: `sit` / `lie` / `kneel` no-arg occupy the floor slot
(routed through `SlotApi`); `stand` no-arg does not (it's the
implicit "free" posture, no slot needed).

## `default: 'ground'` mechanism

The framework substitutes `'ground'` as if the player had typed it,
then runs MQL resolution + validators. `ground` resolves via the
Detail-keyword pathway (`SlotApi.resolveSlot`) to the floor
Adornment's `ground:1` slot. In a void Location with no floor
Adornment, MQL no-match surfaces "you can't sit on the ground here"
— no controller branching.

See [command-spec.md](./command-spec.md) for the field semantics.

## Cross-references

- [slot.md](./slot.md) — substrate.
- [conveyance.md](./conveyance.md) — Mountable; `Postures.Mounted`
  is what `mount` produces and `dismount` clears.
- [boundary.md](./boundary.md) — Adornable retrofit (floor lives
  here).


## History — the furnishing build (2026-07-31)

**Sleep-as-logout** made posture-slot occupancy durable. See
[furnishing.md](./furnishing.md) § Sleep as logout.

**The posture verbs were unreachable by any player, and had always been.**
Driving the world in a browser to verify sleep-as-logout turned up two
independent gaps, neither of which any unit test could see (they all called
`SlotApi.occupyAll` directly):

1. **Nothing contributed the verbs.** `cmd/posture/{lie,sit,stand,kneel}.yaml`
   and their controllers had shipped since the substrate landed, but a verb
   reaches a giver **only** through `commandContributions` (`Perceiver`
   contributes `look` the same way). An uncontributed view is dead YAML, and
   all four answered *"I don't understand"*. `PosedMixin` now contributes
   them on the `self` surface — the `Respiration` inhale/exhale precedent.
2. **No actor was `Slottable`.** `requiresSlottable` gates all four, and its
   own docstring asserted *"v1 actors are always Slottable via Avatar's
   composition"* — which was never true. `Creature` now composes
   `SlottableMixin` beside the `SlottedMixin` it already had: **`Slotted` is
   the chair's side, `Slottable` is the sitter's.**

Verified in a browser against a live world: `lie bed` → *"You lie down."*,
`stand` → *"You stand up."* (Bare `sit`/`kneel` default their target to
`ground`, so they need a room with the default-floor adornment.)

### Occupancy durability

`PosedMixin` persisted the posture, but `getOccupiedHost()` is a **live
scan** — so the bed was gone on restore and `currentRestQuality()` read
1.0 on the very reconcile meant to pay out. `PosedMixin` now also records
**`restingOnPath`** and **`restingSlot`**, stamped by the new
`onSlotOccupied` witness for **posture-bearing slots only** (worn, held and
mount slots are ignored — this remembers where you rest, not what you
carry) and cleared on release — and `MobileMixin.onSlotReleased` had to be taught to
super-chain, because it sits outside `PosedMixin` and was silently
replacing that witness (see [slot.md](./slot.md) and
[antipatterns.md](../antipatterns.md) § *An optional witness implemented
by more than one composed layer*). `PersistableLogic` re-occupies last, after
placement and fixtures; every failure degrades to the room floor with no
error and no teleport.
