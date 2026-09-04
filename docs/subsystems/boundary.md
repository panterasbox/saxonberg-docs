# Boundary Subsystem

Everything that *lives on a boundary* — the seams between
containment scopes. Exits gate movement between rooms. Doors gate
exits. The `Boundary` substrate gates cross-room channels (light,
sight, future sound) through a fixture-anchor model. Adornments
attach non-portable Stuff to a host. Sit in `lib/boundary/`.

The companion subsystem is [spatial.md](./spatial.md) — locations,
zones, coordinates, containment, locomotion. Spatial defines the
containers; boundary defines what connects and attaches to them.

Cross-references:

- [spatial.md](./spatial.md) — locations, zones, vessels (the
  containers boundaries connect), coordinates, containment
  chokepoint, locomotion.
- [light.md](./light.md) — the Light & Boundary subsystem on top
  of the substrate here (Window + Door retrofit, propagation
  walks, per-viewer perception).
- [perception.md](./perception.md) — viewer-aware-query pattern.
- [collections.md](./collections.md) — `addX` / `removeX` /
  `hasX` / `getXs` vocabulary the fixture surfaces use.
- [persistence.md](./persistence.md) — scalar-default rule that
  Boundary subclasses' fields honor.

## The Cast

| Type | Kind | Role |
|---|---|---|
| `Exit` | concrete `Idea` | One-way passage between two `Container & Exitable` endpoints. Carries direction, lazy-resolvable destination, optional door, traversal flags, custom messages. |
| `ExitableMixin` | mixin | Explicit exit map + zone-delegated lookup; `addExit` wires `Door.attachedTo` and (for doored exits) `BoundaryApi.attachExistingBoundary`. |
| `ExitableVessel` | concrete class | A Vessel you can enter. `DoorBearingMixin(ExitableMixin(VisibleMixin(AdornableMixin(Vessel))))` — it composes `Adornable` itself (the fixture surface the Door→`BoundaryAnchor` retrofit needs), since the bare `Vessel` base no longer does. Synthesizes `'in'`/`'out'` exits. Migrates the `(vessel, environment)` Boundary anchor pair on `setDoor` / `onMoved`. |
| `DoorBearingMixin` | mixin | Adds a `door: Door | null` field for hosts whose exits are synthesized rather than authored (`ExitableVessel`). Constrained to `Stuff & Exitable`. |
| `Door` | concrete `Thing` subclass | `LockableMixin(SealableMixin(Boundary))`. Shared open/closed **and** locked/unlocked state referenced by exit pairs. Implements all five conduits — `LightConduit`, `LineOfSight`, `MovementConduit`, `SmellConduit`, `SoundConduit` — all gated on `isOpen()`. `attachedTo: Set<Exit>` is the runtime back-reference. |
| `SwitchableMixin` | mixin | Generic binary on/off toggle (`isOn()` / `setOn()`, `switchOn()` / `switchOff()`). The Sealable of the electrical world — a wall switch, a machine, the crossing `Beacon`. Driven by the `switch` / `toggle` verb (in the `device` category). Registered as `Switchable`, `MixinApi.isSwitchable`. |
| `LockableMixin` | mixin | Binary locked/unlocked state, composed onto `Door` **beneath** `Sealable` (`isLocked()` / `setLocked()`, `lock()` / `unlock()`). A locked door refuses traversal before the closed-door gate fires. **A STOPGAP** superseded by build-3's `lib/lock/` real `Lock`+`Key` model. Registered as `Lockable`, `MixinApi.isLockable`. |
| `BistateMixin` | mixin (unregistered) | The shared guarded-boolean substrate under `Sealable` (open/closed), `Switchable` (on/off), and `Foldable` (folded/unfolded). One persisted boolean + a `typeof`-boolean guard, reached via protected `getState()` / `setState(value, label)`. Lives at the `lib/` root (`lib/Bistate.ts`), NOT in `lib/boundary/`. **Deliberately carries no `_mixinName` and is not in the `Mixins` registry** — shared implementation, not a queryable capability; consumers narrow on the concrete axis (`isSealable` / `isSwitchable` / `isFoldable`), never an `isBistate`. |
| `AdornableMixin` | mixin | Container-side surface for non-portable attached Stuff (`getFixtures()` parallel to `getContents()`). Composed onto `Location` and `ExitableVessel` (not the bare `Vessel` base — every fixture consumer narrows on `MixinApi.isAdornable` first). |
| `AdornmentMixin` | mixin | Host-side back-reference (`adornedTo`) and not-portable invariant. Composed by `BoundaryAnchor` and by content fixtures (e.g. `domain/lounge/NeonSign` = `Adornment(Branded(LightSource(Thing)))`). Attached declaratively via the host's `adornments:` instruction field (see *Declarative adornments* below) or imperatively via `addFixture`. |
| `Boundary` | concrete `Thing` subclass | The two-anchor abstraction for cross-room channels. Just `extends Thing` — `Visible` / `Perceptible` come baked into Thing's default composition. Subclasses (`Window`, `Door`) compose `Sealable` for shutter / closed state. |
| `BoundaryAnchor` | concrete `Thing` subclass | `Adornment` Thing — the per-side proxy in each host's `getFixtures()`. Two anchors per Boundary. |
| `Conduit` | TS interface | Channel-shape: `LightConduit`, `LineOfSight`, `MovementConduit`, `SmellConduit`, `SoundConduit`. Boundary subclasses implement (a subset of) these via `getConduits()`. |
| `Window` | concrete `Boundary` subclass | `SealableMixin(Boundary)` implementing `LightConduit + LineOfSight + SmellConduit + SoundConduit`. `baseTransmissivity`, optional one-way `aToBOverride` / `bToAOverride`, `colorTint`. Shutters via `Sealable.open` gate all four channels. Declarative `attachedHosts: [string, string]` identity refs; `setAttachedHosts` resolves hosts lazily and installs anchors. |
| `BoundaryApi` | static API | `attachExistingBoundary({ boundary, hostA, hostB })`, `create({ factory, hostA, hostB })`, `destruct(boundary)`. |

## Locks & keys (`lib/lock/`)

Locking is **not** a `Boundary` concern — it's a credential check an exit/door
performs. A `Lock` value-object (`lib/lock/Lock.ts`) is `{ keyway, technology }`
(a `LockType`: `pin-tumbler` | `keycard`); a **key** is the third
`CredentialKind` — a `KeyCredential` keychain (bearer `{keyway, technology}`
entries + master technologies) held **either** in the implant wallet **or** on
a physical `Key` Thing (`lib/lock/Key`, the `PaymentCard`/`TravelCard`
precedent). the `Lock` value class (`api/credential.ts`, the unified credential
surface) is the callable home: `issueKey` / `issueMasterKey`, and the
**synchronous** `presentsKey(mover, lock)` — a reachable-wallet scan (implant +
carried key) safe to call from a door's `canTraverse`; minting a fresh keyway
(a lock identity) lives on `Lock.mintKeyway()`. **Bearer + re-key:** possession
is access; a lock is revoked by minting a fresh keyway. First consumer: the
`DormDoor` (see
[residence.md](./residence.md)); the substrate is door-agnostic. See
[credential.md](./credential.md) for the credential-family details.

## Exits

`lib/boundary/Exit.ts`. An `Exit` is a first-class `Idea` —
runtime identity, no physical presence in any room's inventory.
Carries:

- `direction: string` — cardinal long-form, semantic label, or
  vessel-synthesized (`'out'`, `'in'`).
- `source: Stuff & Container` — the Exitable this exit leaves
  from.
- `destination` — live ref OR `destinationPath: string`
  (templatePath, lazy-resolved via `StuffApi.singleton`).
- `door: Door | null` — optional; both sides of a bidirectional
  pair reference the same instance.
- Flags: `hidden`, `blocked`, `muffled`, `noFollow`, `oneWay`.
  `hidden` is now **subsumed into the concealment gate** (below).
- Optional Liquid templates: `messageOut` / `messageIn` for
  movement narration.
- `inverse?: Exit` — counterpart on the other side, wired by
  `addBidirectionalExit` or by the mutual-exit verifier.
- `media: string[]` — locomotion media this exit admits (`'ground'`,
  `'water'`, etc.). Empty list = the **ground pace family**
  (`walk`/`sneak`/`run`) — widened from walk-only when the care↔speed
  axis landed, since anywhere you can walk you can sneak or run; non-ground
  media (climb/swim/fly) and wheeled conveyance still require an explicit
  `media` declaration, so old content stays backcompat-safe. See
  [locomotion.md](./locomotion.md#exitmedia).

### `Exit.hidden` — subsumed into the concealment gate

`Exit` now composes `ConcealableMixin` (`Exit extends
ConcealableMixin(Idea)`, the `LocomotionMode` mixin-on-`Idea` precedent).
The legacy `hidden` boolean — a dead-end "never displayed" flag with **no
reveal path** — is a thin view over the concealment band: `isHidden()`
re-derives as `isConcealed()`, `setHidden(true)` raises the band to
`ConcealmentLevels.hiddenDefault()` (the `concealment.hiddenDefaultLevel`
dial, default `'hidden'`), `setHidden(false)` clears to `'obvious'`. At
construction `ExitOptions.hidden: true` maps onto the same default, and an
explicit `concealment` band on the seed **wins** over the legacy flag. An
`Exit` carries no `templatePath` of its own, so it overrides
`getDiscoveryKey()` to a durable synthetic `` `${source.path}#exit:<dir>` ``
handle (so a discovered secret door stays discovered across re-clones).
`getObviousExits()` still drops a hidden exit (it reads `isHidden()`) — but
the **reveal path now exists**: `Exitable.obviousExitsFor(viewer)` lists a
concealed exit once the viewer has discovered it (or perceives it), and
`search` turns it up. This closes the bar's documented sealed `north →
office` dead-end. Full detail in
[concealment.md](./concealment.md).

`Exit.canTraverse(mover, mode?)` rejects when `blocked`, when an
attached door is **locked**, when an attached door is shut, or — when
`mode` is supplied — when `allowsMode(mode)` returns false (mode's
medium not in `media`). The result is a `TraversalGuard` carrying `gate`
/ `mode` / `context` fields so locomotion controllers can compose
verb-templated rejection prose without re-parsing `reason` (`gate:
'blocked' | 'locked' | 'door' | 'exitMode' | 'bodyPlan' | 'posture' |
'enablement' | 'capability' | 'noConveyance'`). Pre-locomotion callers
omit `mode` and skip the mode gate — additive, no breakage.

The **lock gate runs before the closed-door gate** so a locked door
reports `'locked'` rather than `'closed'`. It reads entirely off
`this.door` (already in hand) — the destination is **never resolved**,
so a locked gate pointing at an unbuilt or dangling destination path is
safe to veto (this is how the crossing's north gate can be seeded
locked while its far side is unbuilt). The rejection prose caps the
leading character of `door.getPresentation()` (which already carries the
article) rather than re-prefixing `"The "`, avoiding a double-article
(`"The the university gate is locked."`). See *Locking* under Doors.

Door routing still consults `door.isOpen()` directly (rather than
going through `MovementConduit.canPassThrough`) — they return the
same answer. Routing through the conduit is deferred to a future Door
subclass that varies on traversal mode.

### Deferred destinations — `DeferredDestinationExit`

`lib/boundary/DeferredDestinationExit.ts` — the base for an exit whose
**destination `Stuff` is faulted in on traversal** while the exit itself
stays **eager**. The reconstitution seam for content that doesn't exist
until you walk toward it: an elastic building's floors/rooms, a
procedurally-generated level, a per-player instanced space.

The name says *destination*, not *lazy exit*, deliberately — the exit is
present and describable the moment you're in its source room:

- **Eager exit.** The Exit object lives in the source room's exit map, and
  its `getDirection()` + `getDestinationTemplatePath()` are read by
  `look` / a future map to render the edge **without materializing the far
  side**. The template path is stored **accurately** (the destination's real
  class template, never a placeholder), so the edge reads honestly before
  it's been walked. Describing an exit never resolves anything.
- **Deferred destination.** The live destination faults in only on
  `resolveDestination()` — the base runs the subclass's
  `computeDestination()` hook, caches the live ref (an instance ref), and
  re-materializes after a reap. For **multi-instance** destinations (many
  clones sharing one template) the template path names the *class*, not the
  clone, so resolution goes through the hook, never path lookup.

The base owns the fault-in lifecycle; subclasses supply only
`computeDestination()` + their own `canTraverse` (passability varies —
a key check vs. a reachability check). Consumers: the dorm's `DormDoor`
(a unit's room) and `FloorStairExit` (a floor's corridor) — see
[residence.md](./residence.md) — and `obj/LotGateExit` (a sold lot's
yard, hung on the street by `PlatWarren`; see
[smallholding.md](./smallholding.md)).

`LotGateExit` is the case where the eager path is **exact** rather than a
class name: each lot's room is minted at its own identity, so the stored
`destinationTemplatePath` is that identity. That matters beyond honesty —
it is the path `CartesianLocation.addExit` resolves the destination ZONE
from when deciding whether a non-cardinal direction is admissible.

#### Non-cardinal exits are a ZONE boundary, not a naming choice

`CartesianLocation.addExit` refuses a non-cardinal direction when source
and destination resolve to the **same** zone. Content hits this the
moment a street wants a gate per lot: `lot-1` is not a compass bearing,
and it cannot be one (a second lot would collide on `north`).

The refusal is the right answer and the fix is upstream — the
destinations belong in a zone of their own, because N lots cannot share
one grid coordinate. Two traps when you author that zone:

- It must be a **`SpatialZone`** (in practice a `CartesianZone`).
  `ZoneApi.resolveZoneForPath` returns only spatial zones and walks
  straight past a `FolderZone`, so a folder leaves every room back in the
  parent's zone — the exact failure you were fixing.
- It must be an **authored template row**, not a minted identity.
  `resolveZoneForPath` walks template ancestry in Mongo, and a minted
  path has no row in `domain`. One authored branch zone covers N minted
  children; a per-child zone cannot be minted at all.

#### `addExit` is async

`Exitable.addExit(exit)` returns `Promise<boolean>`. The async return
type is what makes `CartesianLocation`'s cardinal-only-intra-zone
check possible — the subclass override awaits
`ZoneApi.resolveZoneForPath` (which may clone a zone Stuff on first
reference). The base implementation does only sync work; the async
type signature is for subclass overrides. `addBidirectionalExit` is
also async because it `await`s the two `addExit` calls internally.
Callers must `await`.

### `ExitableMixin` lookup — explicit only

`ExitableMixin` (`lib/boundary/Exitable.ts`) is composed by
`CartesianLocation`, `SphericalLocation`, and `ExitableVessel`.
Exits are **authored explicitly on every room** — a room connects to
exactly what its own template declares, on both sides. The zone does
**not** synthesize exits from grid adjacency; every template is
self-describing. `getExit(direction)` resolves:

1. **Explicit map.** The authored exit, or `undefined`.
2. **Subclass hook.** `ExitableVessel.getExit` overrides for
   `direction === 'out'`.

`getObviousExits()` is the explicit exits, `!hidden` — the viewer-blind
seam kept for the physics/propagation walks. The **viewer-aware**
`obviousExitsFor(viewer)` is what `look` and the perception paths consult:
a concealed exit shows iff the viewer has discovered or perceives it (see
[concealment.md](./concealment.md)).

**No auto-reciprocal.** `applyExits` installs only the edge it declares
(`bidirectional` defaults to **false**); the return trip is a separate
entry on the destination's template. `bidirectional: true` is an
explicit opt-in that also installs the reciprocal — used where the two
exits share one physical `Door` (whose boundary anchors must be wired
once; the door *is* the shared "both sides"). `verifyOutboundExits`
still links the two authored one-way exits of a doorway as inverses
when both are loaded.

`getExitDoors()` collects every non-null `exit.door` reachable
through `getObviousExits` — used by MQL so a player can target a
door by keyword without it living in inventory.

### Bidirectional exits

`Exitable.addBidirectionalExit(other, direction, opts?)` installs
both sides in one call:

- For cardinal `direction`, the opposite is inferred via
  `NavigationApi.invertDirection`. For semantic labels the caller
  passes `opts.opposite` explicitly.
- A shared `door` reference installs on both sides.
- `forward.inverse = back` and `back.inverse = forward` wired so
  `Mobile`'s arrival-message resolver can reach
  `exit.inverse?.direction`.
- **After the Boundary retrofit**: when `opts.door` is non-null,
  `addBidirectionalExit` also calls
  `BoundaryApi.attachExistingBoundary({ boundary: opts.door,
  hostA: this, hostB: other })` to install the door's anchor pair.

One-way exits use `addExit` directly and leave `inverse`
undefined.

### Declarative exits: `applyExits` instruction field

`exits` is an **instruction field** on `ExitableMixin` (declared via
`exits: { instruction: true }` in `fieldMeta`) — its YAML data is a
declaration applied to produce the runtime `exits: Map<string, Exit>`
collection, not stored as a value. The applier consumes a
`Record<string, ExitInstruction>` and installs each entry:

```yaml
# A CartesianLocation template's data:
exits:
  north:
    destination: /eternal/duncan-hall/lounge
  portal:
    destination: /elsewhere
    bidirectional: true
    opposite: portal
  cellar:
    destination: /eternal/duncan-hall/cellar
    door: /eternal/duncan-hall/cellar-door
```

The applier:

- Resolves `destination` and optional `door` paths lazily via
  `StuffApi.singleton` (clones on first need).
- Defaults `bidirectional` to **false** — installs only the edge it
  declares. `bidirectional: true` is an explicit opt-in that also
  installs the reciprocal (its inverse inferred via
  `NavigationApi.invertDirection` for cardinals, or the caller-supplied
  `opposite` for non-cardinal labels).
- Per-direction idempotency: a matching existing exit (same
  destination, same door) is a no-op; a mismatch throws with a
  diagnostic naming both seed paths.
- Calls `addBidirectionalExit` for the bidirectional case (which
  also installs the door's anchor pair when `door:` is present), or
  `addExit` for the one-way / explicit-`bidirectional:false` case.

`ExitInstruction` carries the full exit shape: `destination`, optional
`door`, `bidirectional`, `opposite`, `hidden`, `blocked`, `muffled`,
`noFollow`, `oneWay`, `messageIn`, `messageOut`, `media`. Per the
declarative exit spec (see [templates.md](./templates.md)) and
`feedback_property_vs_instruction_fields`. No paired getter for the
spec — the runtime `exits: Map<string, Exit>` collection has its own
established API (`getExit`, `addExit`, …) which is the only public
surface for reading exits at runtime.

### Mutual-exit verification

`ExitableMixin.verifyOutboundExits()` walks the
`_pendingVerify` set — exits flagged at `addExit` time as
needing inverse-wiring. Settled exits (wired, oneWay, blocked,
non-cardinal, no resolvable destPath) are evicted as the
verifier observes them. Idempotent. Fires from
`postRegister` (load-time) and from `Mobile.traverse`
(traversal-time fallback).

### Lazy Exit destination resolution

An Exit can be authored with a live `destination` ref or a
`destinationPath` (templatePath string). The synchronous
`destination` getter consults the singleton cache first; if the
destination isn't loaded the getter throws — async-aware
callers (`Mobile.traverse`) `await
exit.resolveDestination()` first.

## Exit-kind templates (`/stuff/idea/exits/<kind>`)

An **exit kind** is an authored template a room's `exits:` entry
references via `kind:` — the authorable *nature of the passage*
(traverse messages, locomotion `media`, `wheelPassable`, concealment
defaults), distinct from a `Door` (the operable barrier object):

```yaml
exits:
  north:
    destination: /world/terminus/counting-houses/circle-parlor
    kind: /stuff/idea/exits/archway        # authored defaults for this edge
    messageOut: "{{ mover }} ducks under the arch."   # per-site override
```

Mechanics (the identity doctrine's first consumer — see
[ref-shapes.md § Identity, lineage, and backing](../ref-shapes.md)):

- `applyExits` **clones** the kind (`StuffApi.clone`; the template's
  `class:` may name an `Exit` subclass) — one instance per edge, two
  for a `bidirectional` entry. The clone hydrates the kind's authored
  fields (`Exit.persistentFields` is the hydration allowlist; exits are
  never actually persisted — no host captures them).
- The installing room completes identity via **`Exit.bind(opts)`** —
  participant-gated (`FromMixin(Mixins.Exitable)` + a party-to-the-edge
  `where`: the caller must be the bind options' source or destination),
  `@Final @Unshadowable`, one-shot (re-bind throws). `bind` is
  **delta-aware**: per-site entry fields override the kind's hydrated
  defaults; unset fields keep them.
- A bare `new Exit()` (no options) is the **unbound** state the clone
  pipeline constructs through; `isBound()` reports it, an unbound
  orphan is ordinary cullable clutter (no live-room residency veto),
  and its `getDiscoveryKey()` is `undefined`.
- **Identity payoff:** a kind-cloned exit's `templatePath` IS the kind
  (`/stuff/idea/exits/archway`) — the population is addressable
  (`findAllByTemplatePath`, the MQL path seed), and a CMS edit to the
  kind re-hydrates its clones via the standard content go-live. The
  *instance* half of an exit's identity stays the relational discovery
  key (`<sourcePath>#exit:<dir>`).
- Entries without `kind:` construct a raw `Exit` exactly as before — a
  bare passage has nothing authored, so it earns no row.

Seeded kinds: `/stuff/idea/exits/archway`, `/stuff/idea/exits/stair`
(`wheelPassable: false` — the haulage residue). Demonstrator: the
counting-houses banking-hall → circle-parlor archway.

## Doors

`lib/boundary/Door.ts`. Composition:
`LockableMixin(SealableMixin(Boundary))`. Boundary supplies `Visible +
Perceptible + Thing`. The Containable composition (from Thing) is what
enables broken / unhinged doors to become inventory items. The two
boolean fields (`open` from Sealable, `locked` from Lockable) are
distinct and coexist on the Door.

The same physical door is referenced by potentially many Exits
(canonical case: forward + back). The `attachedTo: Set<Exit>`
field is the runtime back-reference, populated by:

- `ExitableMixin.addExit` when the exit's `door` is non-null
- `ExitableMixin.addBidirectionalExit` indirectly
- `ExitableVessel`'s synthesized exit factories when
  `vessel.door` is non-null

…and unwired by `removeExit`, `Exit.onDestruct`,
`vessel.setDoor` cache invalidation, and `Door.detach()`.

### Conduit registry

A Door advertises three conduits — `LightConduit`,
`LineOfSight`, `MovementConduit` — all gated on `isOpen()`:

```ts
public override getConduits(): readonly Conduit[] {
  return [lightConduitFor(this), lineOfSightFor(this), movementConduitFor(this)];
}

public transmissivity(_from, _to): number { return this.isOpen() ? 1 : 0; }
public canSeeThrough(_from, _to): boolean { return this.isOpen(); }
public canPassThrough(_from, _to, _mode): boolean { return this.isOpen(); }
```

A closed Door blocks light, sight, and movement uniformly. The
Light propagation walk consumes the `LightConduit` via the
boundary path; `Exit.canTraverse` consults `isOpen()`
directly (same answer as the `MovementConduit`).

### Detach choreography

`Door.detach()` walks BOTH back-references:

1. Existing `attachedTo` — for each Exit, clear its `door` ref.
2. `super.detach()` (Boundary) — clear both anchor slots and
   remove the anchors from each host's `getFixtures()`.

Then `ContainmentApi.move(door, location)` plants the
broken/removed door somewhere. Reinstall reverses:
`ContainmentApi.move(door, null)` plus
`addBidirectionalExit(other, dir, { door })` (which also
installs the new anchor pair).

`Door.onDestruct` is inherited from Boundary — it calls
`detach()` (now also clearing attachedTo) and then destructs the
captured anchors.

### `Door.getKeywords()`

Overrides Perceptible to union the explicit keyword list with
the tokens of the door's `shortDescription`, so a door authored
with only `shortDescription: 'heavy oak door'` is targetable as
`oak` / `door` without re-listing.

### Locking

`lib/boundary/Locked.ts`. `LockableMixin` composes onto Door **beneath**
`Sealable`, adding a second boolean axis:

- `isLocked()` / `setLocked(value)` — the predicate-getter / noun-setter
  inter-Stuff contract pair. `setLocked` rejects non-booleans with a
  `TypeError` (a malformed template `locked: 1` crashes loudly at
  hydrate rather than being silently coerced).
- `lock()` / `unlock()` — the idempotent action verbs.

The enforcement lives in `Exit.canTraverse` (see *Exits* above): the
lock gate fires **before** the closed-door gate, reads only `this.door`
(never resolving the destination), and reports `'locked'`. The `lock` /
`unlock` verbs (`platform/idea/cmd/boundary/LockController.ts` /
`UnlockController.ts`, mirroring `Close` / `Open`) resolve any reachable
`Lockable` — a direct hit (`lock oak door`) or a direction match (`lock
north`, door fetched from `via.exit.getDoor()`) via
`MqlApi.effectiveTarget`.

**⚠ STOPGAP.** `LockableMixin` carries **no key/credential model** — the
crossing's north gate is seeded permanently locked and soft-walled in
dialogue (Gus), so `lock` / `unlock` are minimal admin / no-key verbs. It
is superseded by build-3's `lib/lock/` real model (a re-keyable `Lock`
value-object a door carries, plus `Key` riding `CredentialWalletMixin` /
the `Lock` value class — "the door checks a key, not identity"). This boolean is
a degenerate special case of that (a lock nobody holds a key for). **At
reconcile toward `lib/lock/`:** retire this mixin + the `'locked'`
`Exit.canTraverse` gate and re-express the gate as either a plain
`blocked` exit or a keyless `Lock`. Do NOT grow it into a second lock
system, and do NOT fold it into the shared `BistateMixin` base — it is
leaving the boolean world. See commit `17add9a5`.

### `DoorBearingMixin`

`lib/boundary/DoorBearing.ts`. Adds `door: Door | null`
(runtime-only, NOT in `fieldMeta`'s persistent entries), `getDoor()`, default
`setDoor()`. Bound to `Stuff & Exitable`. Composed by
`ExitableVessel` so its synthesized `'in'`/`'out'` exits can
pull the carried door at template/clone time.

`ExitableVessel` overrides `setDoor` to invalidate the
synthesized-exit caches AND to migrate the door's
`(vessel, environment)` Boundary anchor pair (see "ExitableVessel
anchor migration" below).

**`DoorBearing` is NOT the same as `BoundaryAnchor`** —
`DoorBearing` is a host-side mixin holding one Door reference;
`BoundaryAnchor` is a `Thing` subclass that proxies a Boundary
on one side. Both can be present on `ExitableVessel` for
different roles.

### `ExitableVessel` anchor migration

A vessel-door is a Boundary on the runtime triple
`(vessel, environment)`. As the vessel moves, the boundary
follows.

- `ExitableVessel.onMoved(_from, to)` — destructs the existing
  anchor pair via `Boundary._detachAndDestructAnchors()` (the
  subclass-bypass seam that doesn't touch Door's `attachedTo`),
  then re-installs the pair on `(vessel, to)`.
- `ExitableVessel.setDoor(door)` — destructs the old door's
  anchor pair, updates the door reference, installs the new
  door's pair on `(vessel, currentEnv)`.

So a wardrobe with an open Door reads room ambient inside; a
closed Door reads ZERO inside; moving the wardrobe to a different
room migrates the boundary so the inside reads the new room's
ambient.

## Adornable / Adornment

`lib/boundary/Adornable.ts` and `lib/boundary/Adornment.ts`.

`Adornable` (composed onto `Location` and `ExitableVessel`) gives a
Container a fixtures collection parallel to `contents`. Fixtures
are NOT inventory; they don't show up in `look`-list output by
default. The not-portable invariant is enforced at
`ContainmentApi.move`: attempting to move an attached `Adornment`
rejects with `ContainmentError` until the host calls
`removeFixture` first.

**Slotted underpinning (post-retrofit).** `Adornable` composes the
`Slotted` substrate (`lib/slot/Slotted.ts`) and overrides the
universe surface to derive from its live fixture map (Pattern C in
[slot.md](./slot.md)). Fixtures are keyed by synthetic slot names
(`fixture:1`, `fixture:2`, …) generated by an instance-local
counter; authors who want meaningful names pass an explicit
`slotName` to `addFixture(f, slotName)`. The runtime storage is
`Map<string, Stuff & Adornment>` rather than the older `Set`, but
the **public API surface is unchanged** — `addFixture` /
`removeFixture` / `hasFixture` / `getFixtures` /
`getFixtureBoundaries` / `getFixtureLightSources` all preserve
their pre-retrofit shape.

`Adornment` composes `Slottable` so adornments pass
`MixinApi.isSlottable`. The `adornedTo` back-reference stays as
the parent pointer the slot framework doesn't carry.

Surface (canonical collection vocabulary —
[collections.md](./collections.md)):

```ts
addFixture(f: Stuff & Adornment, slotName?: string): boolean;
removeFixture(f: Stuff & Adornment): boolean;
hasFixture(f: Stuff & Adornment): boolean;
getFixtures(): readonly (Stuff & Adornment)[];
getFixtureBoundaries(): Boundary[];      // dedupes via anchor → boundary
getFixtureLightSources(): (Stuff & Adornment)[];
```

### Declarative adornments: `applyAdornments` instruction field

`adornments` is an **instruction field** on `AdornableMixin` (declared
via `adornments: { instruction: true }`) — the `applyExits` /
`applyProps` precedent applied to fixtures. Its YAML data is an
array of `AdornmentSpec` entries (a bare `template` path, or
`{ template, slot }` for a meaningful slot name). The Hydrator's
Phase-2 dispatch calls `applyAdornments`, which **clones** each
template (fixtures are per-instance — no singleton dispatch like
`applyProps`), guards that it composes `AdornmentMixin`
(`MixinApi.isAdornment`, else a configuration-error throw naming the
path), and attaches it via `addFixture`. There is no paired getter —
the live fixtures are read through `getFixtures()`; the spec is
discarded after Phase 2.

Fixtures are runtime-only (not persisted; `addFixture` sets no
`environment`, matching `BoundaryApi.attachExistingBoundary`), so
`applyAdornments` re-runs and rebuilds them on every hydrate — the
same reconstruct-on-boot model the `BoundaryAnchor` wiring uses. This
is the declarative seam that retired the imperative-only fixture
wiring: a seed can now author wall décor (e.g. the `NeonSign` branded
light-emitting fixtures on Dave's Bar back-bar wall) as pure data.

```yaml
adornments:
  - { template: /world/lounge/thing/neon-veshko, slot: sign:veshko }
  - /world/lounge/thing/neon-aevex            # bare path → auto slot
```

**Floors are Adornments.** The default-floor template
(`/idea/surface/default-floor`) composes `Adornment + Postured`
and lives as an Adornment on the Location's Adornable surface.
`sit ground` routes via the floor's `userFacingDetail: 'floor'`
slot to the floor's `ground:1` posture-bearing slot. See
[posture.md](./posture.md) for the full ground-targeting story.
Trapdoors-as-Adornments-on-floors compose the existing
Adornable-of-Adornable pattern — no Boundary-substrate change.

`Adornment` is the host-side back-reference (`adornedTo`) and the
not-portable invariant. Composed by `BoundaryAnchor`; future
fixture types (sconces, decorations) compose it too.

`AdornableMixin.onDestruct()` walks `fixtures` and destructs
each via `StuffApi.destruct`. A `BoundaryAnchor` being destructed
clears its slot in the boundary; if the host on the other side
is still alive, that side's anchor stays put and the boundary
lingers half-attached until explicitly destructed.

## Boundary Substrate

`lib/boundary/Boundary.ts`, `lib/boundary/BoundaryAnchor.ts`,
`lib/boundary/Conduit.ts`.

A `Boundary` connects two `Adornable` containers and gates
channels (light, sight, movement, future sound) between them.
The runtime triple is `(boundary, anchorA, anchorB)` — the
Boundary itself plus a `BoundaryAnchor` Thing in each host's
`getFixtures()`.

```
       Boundary
       /       \
  anchorA    anchorB
     |          |
   hostA      hostB         ← Adornable containers (rooms / exitable vessels)
```

`Boundary` just `extends Thing` — `Visible` and `Perceptible`
come baked into Thing's default composition, giving it identity,
a description body, and keyword-addressable status. Because it
also picks up `Containable` from Thing, a Boundary detached from
both sides can become inventory (same shape as a salvaged door).
Subclasses (`Window`, `Door`) compose `Sealable` for shutter /
closed state.

`BoundaryAnchor` is `AdornmentMixin(Thing)` — sits in
`getFixtures()`, knows its boundary and side (`'A'` or `'B'`).

### Conduits

A `Conduit` is a TypeScript interface a Boundary subclass
implements via wrapper objects returned from `getConduits()`.
Each conduit covers one channel with directional methods that
take `BoundarySide` arguments:

```ts
interface LightConduit    { conduitKind: 'light';
  transmissivity(from: BoundarySide, to: BoundarySide): number;
}
interface LineOfSight     { conduitKind: 'sight';
  canSeeThrough(from: BoundarySide, to: BoundarySide): boolean;
}
interface MovementConduit { conduitKind: 'movement';
  canPassThrough(from: BoundarySide, to: BoundarySide, mode: string): boolean;
}
interface SoundConduit    { conduitKind: 'sound'; }   // reserved for v2
```

Conduits MUST NOT cache — the boundary's own state (e.g.,
Sealable `isOpen`) participates in transmissivity and changes at
runtime.

### `BoundaryApi`

```ts
BoundaryApi.attachExistingBoundary({ boundary, hostA, hostB });
// Constructs the two anchors, wires anchorA/anchorB on the boundary,
// calls hostA.addFixture(anchorA) and hostB.addFixture(anchorB).

BoundaryApi.create({ factory, hostA, hostB });
// Convenience: StuffApi.create(factory) + attachExistingBoundary.

BoundaryApi.destruct(boundary);
// StuffApi.destruct(boundary) → boundary.onDestruct()
// detaches anchors from hosts and destructs them → boundary destroys.
```

Asymmetry guards: rejects `hostA === hostB` and rejects
re-installing on a boundary that already has anchors (would
silently orphan the previous pair).

### Naming: `BoundaryAnchor` vs `DoorBearing`

The two roles do not overlap. `DoorBearing` is a host-side
**mixin** giving an Exitable a single `door` field (used by
`ExitableVessel` for synthesized exits). `BoundaryAnchor` is a
`Thing` subclass + `Adornment` that sits in `getFixtures()` as
the per-side proxy of a `Boundary`. After the Door retrofit,
`ExitableVessel` participates in BOTH — `DoorBearingMixin` for
the field, anchor-pair for the runtime fixture installation. Do
not unify.

## Window

`lib/boundary/Window.ts`. The first concrete Boundary user
beyond Door. `SealableMixin(Boundary)`. Implements `LightConduit
+ LineOfSight` via wrapper objects from `getConduits()`.

Persistent state (per the [persistence.md](./persistence.md)
scalar-default rule):

- `baseTransmissivity: number` (default 1.0) — symmetric
  pass-through factor when open.
- `aToBOverride: number | null` — one-way override for A→B (set
  to `0` for the dark side of one-way glass).
- `bToAOverride: number | null` — same for B→A.
- `colorTint: ColorTag | null` — stained glass.
- `attachedHosts: [string, string] | null` — identity refs. Two
  templatePaths of the hosts this Window connects. The setter
  resolves both paths via `StuffApi.singleton` and installs the
  anchors via `BoundaryApi.attachExistingBoundary`. Per the declarative
  attachedHosts spec (see [templates.md](./templates.md)).
- `open: boolean` (from Sealable, was `isOpen`) — shutter state.

The structured runtime API `getDirectionalOverrides()` /
`setDirectionalOverrides()` reconstructs / decomposes the
`{ aToB?, bToA? } | null` shape from the two stored scalars.

### Declarative wiring via `attachedHosts`

A templated Window declares the two host paths in its YAML:

```yaml
class: /lib/boundary/Window
hydratorClass: /platform/idea/persistence/PersistentHydrator
data:
  baseTransmissivity: 0.9
  attachedHosts:
    - /eternal/duncan-hall/lobby
    - /eternal/duncan-hall/freshman-dorm-A
```

`setAttachedHosts([pathA, pathB])` is a setter with side effects (per
[location.md § The setter-with-side-effects pattern](./location.md#the-setter-with-side-effects-pattern)):
stores the pair, resolves both hosts lazily, calls
`BoundaryApi.attachExistingBoundary`. Idempotent on re-set with the
same pair (in either order — boundaries are undirected). Throws on
half-attach corruption or mismatched re-attach.

The check `MixinApi.isAdornable(hostA) && MixinApi.isAdornable(hostB)`
gates the attach so a Window can't be wired to a non-Location host.
Per `ref-shapes.md` § the identity ref — the runtime keeps the original
string array; persist-back writes it back unchanged. No marshaller.

### Exit-kind templates (`/stuff/idea/exits/<kind>`)

An **exit kind** is an authored template a room's `exits:` entry
references via `kind:` — the authorable *nature of the passage*
(traverse messages, locomotion `media`, `wheelPassable`, concealment
defaults), distinct from a `Door` (the operable barrier object):

```yaml
exits:
  north:
    destination: /world/terminus/counting-houses/circle-parlor
    kind: /stuff/idea/exits/archway        # authored defaults for this edge
    messageOut: "{{ mover }} ducks under the arch."   # per-site override
```

Mechanics (the identity doctrine's first consumer — see
[ref-shapes.md § Identity, lineage, and backing](../ref-shapes.md)):

- `applyExits` **clones** the kind (`StuffApi.clone`; the template's
  `class:` may name an `Exit` subclass) — one instance per edge, two
  for a `bidirectional` entry. The clone hydrates the kind's authored
  fields (`Exit.persistentFields` is the hydration allowlist; exits are
  never actually persisted — no host captures them).
- The installing room completes identity via **`Exit.bind(opts)`** —
  participant-gated (`FromMixin(Mixins.Exitable)` + a party-to-the-edge
  `where`: the caller must be the bind options' source or destination),
  `@Final @Unshadowable`, one-shot (re-bind throws). `bind` is
  **delta-aware**: per-site entry fields override the kind's hydrated
  defaults; unset fields keep them.
- A bare `new Exit()` (no options) is the **unbound** state the clone
  pipeline constructs through; `isBound()` reports it, an unbound
  orphan is ordinary cullable clutter (no live-room residency veto),
  and its `getDiscoveryKey()` is `undefined`.
- **Identity payoff:** a kind-cloned exit's `templatePath` IS the kind
  (`/stuff/idea/exits/archway`) — the population is addressable
  (`findAllByTemplatePath`, the MQL path seed), and a CMS edit to the
  kind re-hydrates its clones via the standard content go-live. The
  *instance* half of an exit's identity stays the relational discovery
  key (`<sourcePath>#exit:<dir>`).
- Entries without `kind:` construct a raw `Exit` exactly as before — a
  bare passage has nothing authored, so it earns no row.

Seeded kinds: `/stuff/idea/exits/archway`, `/stuff/idea/exits/stair`
(`wheelPassable: false` — the haulage residue). Demonstrator: the
counting-houses banking-hall → circle-parlor archway.

## Doors are wired transitively, not via `attachedHosts`

`Door` is intentionally NOT extended with `attachedHosts`. Doors are
wired by `Exitable.applyExits` when an exit declares `door:` — the
existing `addBidirectionalExit({ door })` path installs the door's
anchor pair as a side effect. This keeps the door's "I'm part of
exit topology" identity primary and the boundary anchor pair
secondary. The slate ratifies the distinction; future Boundary
subclasses follow Window's shape (declare `attachedHosts` directly)
unless their identity is tied to an Exit.

`transmissivity(from, to)`:

- Returns 0 if `!isOpen` (shutters closed).
- Returns the relevant directional override if non-null, else
  `baseTransmissivity`.

`canSeeThrough(from, to)` — `transmissivity(from, to) > 0`.

Template authoring: Window is template-loadable like Door
(`class: '/lib/boundary/Window'`,
`hydratorClass: '/platform/idea/persistence/PersistentHydrator'`). Seed
code calls `BoundaryApi.attachExistingBoundary` to install on
two rooms.

## Switchable

`lib/boundary/Switchable.ts`. A generic binary on/off toggle — the
Sealable of the electrical world, the same surface shape as open/close
over Sealable but on the on/off axis:

- `isOn()` / `setOn(value)` — the predicate-getter / noun-setter
  inter-Stuff contract pair.
- `switchOn()` / `switchOff()` — the idempotent action verbs.

Per `feedback_boolean_field_naming` the field, setter, and YAML key use
the noun form (`on`); only the predicate getter takes the `is` prefix.
Reads naturally at every site: `beacon.setOn(true)`, `beacon.isOn()`,
`data: { on: true }`.

The global `switch` / `toggle` verb (`cmd/device/switch.yaml` +
`platform/idea/cmd/device/SwitchController.ts` — the **`device` category**, not
`boundary`) targets any `Switchable` in reach rather than a specific
class: `switch beacon on`, `switch lamp off`, or `toggle lamp` to flip to
the opposite of its current state.

The shipped consumer is the crossing's `Beacon` (`obj/Beacon.ts`, a
`Switchable(Propertied(Detailed(Thing)))` pedestrian signal — on = WALK,
off = STOP, the walk/stop meaning is prose over the on/off state; the
beacon gates nothing). The build's lamppost was later cut to prose, so
`Beacon` is the sole live Switchable.

### The shared `BistateMixin` base

`Switchable`, `Sealable` (open/closed), and `Foldable`
(folded/unfolded) are the same shape — one persisted boolean, a
predicate getter, a noun setter that rejects non-booleans loudly, two
idempotent action verbs — differing **only** by name. The one genuinely
shared piece, the guarded boolean, is factored into `BistateMixin`
(`lib/Bistate.ts`, at the `lib/` root because the concept spans
spatial / boundary / slot with no single subsystem home — the
`lib/reserve.ts` precedent). Each toggle mixin is then a thin naming
layer: `class SwitchableMixin extends BistateMixin(Base)` casting `this`
to the internal `BistateInternal` (`getState()` / `setState(value,
label)`) shape.

`BistateMixin` is **deliberately unregistered** — it carries no
`_mixinName` and is absent from the `Mixins` registry. It is shared
*implementation*, not a contract surface: consumers narrow on the
concrete axis (`isSwitchable` / `isSealable` / `isFoldable`), never an
`isBistate`. Registering it would invent a capability nobody queries and
leak the base into the author-facing mixin roster. (`LockableMixin` is
NOT part of this family — it is a stopgap leaving the boolean world for
`lib/lock/`, so it keeps its own inline guarded boolean rather than
extending `BistateMixin`.)

## Persistence Notes

- Exits: `direction`, `destinationPath`, flags. The `door` Stuff
  reference goes via custom `persistenceHandler`. Every exit is
  explicit and persisted — the zone no longer derives any.
- Doors: `open` (from Sealable) plus `locked` (from Lockable) — both
  boolean scalars. The `attachedTo` Set and the anchor pair are
  runtime-only.
- Switchables: `on` (from Switchable, stored via `BistateMixin`'s
  guarded boolean) — a single boolean scalar.
- Boundaries: subclass-specific (Window's transmissivity +
  overrides + colorTint). Anchors are runtime-only; v1 anchor
  wiring is hydrate-only via seed code calling
  `BoundaryApi.attachExistingBoundary`.
- Adornments: `adornedTo` is a Stuff cross-reference, follows
  the `Containable.environment` shape (custom
  `persistenceHandler`). Composing classes that need attachment
  to survive restart own that.

## History

**Exits became explicit-only** in the Terminus Terminal build
(`feature/terminus-terminal`, commits `f142178c` + `84f30cdc`). The
`CartesianZone`/`SpatialZone` layer previously *derived* cardinal exits
from grid adjacency (and `applyExits` auto-installed a reciprocal for
cardinal directions); `getExit` fell through to a `deriveExit()` synthesis
path. That was removed: every exit is now authored explicitly on both
sides of a connection, `applyExits` installs only the edge it declares
(`bidirectional` defaults **false**; `bidirectional: true` opts into the
reciprocal for a shared `Door`), and `deriveExit` is gone. The motivation
was legibility — any room template now names exactly what it connects to,
readable without simulating the grid. The zone's remaining job is to
*enforce* connectivity invariants, not to invent edges. Content that
relied on derivation (the Terminus hub, the EU campus) migrated to
explicit-both-sides exits in the same build.

**`Switchable` and `Lockable` landed** in the University Avenue crossing
build (`feature/university-avenue-crossing`, commits `ab0867ba` Phases
1B–1D): `SwitchableMixin` for the crossing `Beacon`, and `LockableMixin`
on `Door` for the permanently-locked north gate soft-walled by Gus. The
shared **`BistateMixin`** base (`lib/Bistate.ts`) was extracted in
`a99eccb4`, factoring the guarded boolean out of `Sealable` /
`Switchable` / `Foldable`. `Lockable` was marked a **stopgap** in
`17add9a5` — superseded by build-3's `lib/lock/` real `Lock`+`Key`
model; the keyed-credential lock is the future, and this boolean is to be
retired (not grown) at reconcile.

## Cross-References

- [spatial.md](./spatial.md) — locations, zones, vessels,
  containment, locomotion. The containers boundary objects
  attach to.
- [light.md](./light.md) — Light propagation walk consumes the
  `LightConduit` and `getFixtures()` surfaces.
- [perception.md](./perception.md) — viewer-aware queries.
- [persistence.md](./persistence.md) — scalar-default rule the
  Boundary subclasses honor.
- [antipatterns.md](../antipatterns.md) — `ContainmentApi.move`
  over raw `setContainer`; pre-asserted casts; instanceof
  vs. virtual methods vs. cast-by-invariant.

## `hasSpatialDestination()` — an exit that leads nowhere

The sandbox build added an `@hook` to `Exit`:

```ts
public hasSpatialDestination(): boolean { return true; }
```

True for every ordinary exit. An exit subclass that fully applies its
own traversal (`applyTraversal`) may name **no room at all** — the
sandbox's `SandboxCrossingExit` is the shipped case: crossing onto the
wire moves no body, and its `destinationPath` is a presentation label
naming the wire, not a place.

The distinction matters to anything that walks the exit **graph**
structurally instead of traversing it. The vision flux walk is the
first such consumer and skips exits that answer false; **sound,
pathfinding and `reachable` have the same shape and will want the same
check.** Found live: the light walk followed the wardrobe's
`/home` label, landed on the HomeZone *Idea* — which has no
`getContents` — and `look` threw for everyone in the room.

See [sandbox.md](./sandbox.md).
