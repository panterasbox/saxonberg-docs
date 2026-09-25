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
| `PosedMixin` (the verbs) | `lib/character/Posed.ts` | Slot+state mechanics ON the actor since the OO sweep (`actor.transferPosture`, `actor.vacatePostureBearingSlots`, `actor.currentPostureBearingSlot`; the mutators are gated to the posture controllers — the narrow-entry pattern — and sealed). Pure mechanism — no messaging; callers (controllers) own the narration |

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

⭐⭐ **Every Location has a floor, and nobody authors it** (ground build,
2026-09-23). The canonical `ground:1` slot is a **class-level default** on
`FloorMixin` — a `Floor` row that authors no `staticSlots` gets exactly
this, which is why `forge-floor`, which authored none, went from pourable-
but-not-sittable to sittable:

```yaml
- name: 'ground:1'
  accepts: SlottableMixin
  capacity: 9007199254740991  # UNBOUNDED_CAPACITY (JSON-safe)
  postures: [sit, lie, kneel, stand]
  userFacingDetail: floor
```

`Location.ensureFloor()` runs at `postRegister` — the one lifecycle every
clone AND every `create` passes through — and clones
`TemplatePaths.defaultFloor` unless the room already has a floor or opts
out. So a warren-minted room gets its floor from the same line an authored
one does.

⚠⚠ **A floor's `ground` / `floor` keywords are on the CLASS, not the row.**
`FloorMixin.getKeywords()` unions them. This is not belt-and-braces
tidiness: the MQL scope walk pools a thing's own `getKeywords()`, and
`pushDetails` gives a detail the pool `[<its id>]` and **never its authored
`keywords:`** — so `default-floor`'s `details.floor.keywords` was dead text,
and attaching that row to all 184 Locations would have fixed `look floor`
and left bare `sit` broken. `lint:ground` clause (d) also holds every floor
row to carrying both words in its own list, so a row reads honestly alone.

Per-Location authoring, three rungs, none of them required:

```yaml
# Rung 1 — a floor ROW, for a floor with DETAILS (a worn track, a gutter)
adornments: [{ template: /world/terminus/.../crossing-paving, slot: floor }]

# Rung 2 — three words on the room, for a floor that is just a material
floor: { material: /stuff/idea/material/rock/granite, worked: true }

# Rung 3 — nothing at all. The default floor resolves its own material:
#   authored → the room's floor.material → the ground beneath (a
#   GroundSource) → the room-kind default → the plain default.

# Lava chamber — only stand. An authored slot set is obeyed EXACTLY,
# including one narrower than the canonical spec.
adornments: [{ template: /stuff/thing/surface/lava-floor, slot: floor }]
```

⚠ **`noDefaultFloor: true` on a Location opts out, and it is REAL now** —
it was documented here, in `spatial.md` and in `default-floor.yaml` for two
builds while **no code read it**. The void row uses it. ⭐ And it is a
different question from `onGrade`: *is this place standable* vs *does the
ground continue beneath the floor*. Conflating them would hand a
sky-exposed flying-only room an **earth** floor (you could sit down on the
sky) and floor an open water column when only its **bed** has ground under
it. Existence defaults to yes; mid-air and mid-water declare the opt-out.
Neither exists in the game yet — the seam and its test ship so the builder
who arrives finds them.

## Verbs

The four with-arg posture verbs (`sit X`, `lie X`, `kneel X`,
`stand X`) and `stand X` call `actor.transferPosture(
target, posture, verb)` for the slot+state mechanics; the
controller emits the verb-specific message scene on success. The
asymmetric `stand` no-arg form calls
`actor.vacatePostureBearingSlots()` directly (no occupy step —
posture flips to `Stand`, the actor stays free of any slot) and
fires its own "you stand up" scene.

Layering: the posture methods are pure mechanism, no `MessageApi.send`
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
(routed through `Slotted/Slottable`); `stand` no-arg does not (it's the
implicit "free" posture, no slot needed).

## `default: 'ground'` mechanism

The framework substitutes `'ground'` as if the player had typed it,
then runs MQL resolution + validators. `ground` resolves via the
Detail-keyword pathway (`resolveSlot`) to the floor
Adornment's `ground:1` slot. In a void Location with no floor
Adornment, MQL no-match surfaces "you can't sit on the ground here"
— no controller branching.

See [command-spec.md](./command-spec.md) for the field semantics.

## ⚠⚠ The gate that named an id that cannot exist

From the OO sweep until the world-scan build's drive (2026-09), the six
posture verbs **did not work over the wire at all**. `PosedMixin`'s
mutators are gated to the posture controllers — the narrow-entry
pattern, correct — but every one of those controllers is a **default
export**, whose module id is the bare path. The policy list appended
`#StandController` to each, naming an id nothing can ever have, so
`sit` / `stand` / `lie` / `kneel` / `mount` / `dismount` denied every
caller.

⭐ **225 unit tests passed either way**, because they call the mixin
methods directly, where `SelfOnly` admits them. Only a real dispatch
goes through the controller, and only the live drive does a real
dispatch. `lint:gates` refuses the suffixed form on a default export
now — see [call-security.md](./call-security.md) and
lint-family.md.

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
`occupyAll` directly):

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
