# Boundary Subsystem

Everything that *lives on a boundary* — the seams between
containment scopes. Exits gate movement between rooms. ⭐⭐ **A way's
boundary is ONE object and each place's exit is a SIDE of it** (the
boundary build, 2026-11): a door, a window, a curtain, a gate between two
places is shut, locked and named the same from both, and what it lets
through is a table of channels on its row. Adornments attach
non-portable Stuff to a host. Sit in `lib/boundary/`.

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
| `Exit` | concrete `Idea` | One-way passage between two `Container & Exitable` endpoints — and one SIDE of the boundary standing in it (`getBoundary()`, null for an open way). Carries direction, lazy-resolvable destination, traversal flags, custom messages; `wayState()` / `wayRecord()` are what every screen shows. |
| `ExitableMixin` | mixin | Explicit exit map + zone-delegated lookup; `applyExits` attaches each authored `door:` side to the one boundary row; `removeExit` detaches the side; `edgeRefusal` is the hook `Location` answers with the zone's edge rules. |
| `ExitableVessel` | concrete class | A Vessel you can enter. `ExitableMixin(AdornableMixin(AtmosphericMixin(Vessel)))` — ⭐ **`Atmospheric` since the base-class narrowing build: a thing you can go inside is a place with air** (see [biome.md](./biome.md)). Carries the authored `interiorVolume` the envelope runs on, and overrides `envelopeApplies` + `openExteriorOpenings` because a room's version of each is wrong about a vehicle. Synthesizes `'in'`/`'out'` exits, which carry no boundary: ⭐ **its own `Sealable` is its door** — a shut vessel drops its `out` from the walks' neighbours (`getObviousNeighbours`), so no light, sound or smell comes in. Resets the weather-locality memo on a move — a place that goes places has a new address. |
| `Door` | concrete `Thing` (`platform/thing/Door`) | `LockableMixin(platform Boundary)` — a boundary with a **bolt**: a door, a gate, a hatch, a gangway-to-be. `closesBehind` (a door on a weight or a latch); ⭐ `freeFrom` (the place whose side opens with no key). |
| `SwitchableMixin` | mixin | Generic binary on/off toggle (`isOn()` / `setOn()`, `switchOn()` / `switchOff()`). The Sealable of the electrical world — a wall switch, a machine, the crossing `Beacon`. Driven by the `switch` verb (in the `device` category). Registered as `Switchable`, `MixinApi.isSwitchable`. |
| `LockableMixin` | mixin | ⭐ **A keyway AND a bolt** (`lib/boundary/Lockable.ts`), composed onto `Door` (over the platform twin's `Sealable`), onto the gangway's `BerthExit`/`AshoreExit` (which delegate to the ship) and water's `Gate`. The bolt is `isLocked()` / `setLocked()` / `lock()` / `unlock()`; the keyway is `getLock()` / `opensFor(mover)` over `lib/lock/Lock`. `canPass = !isLocked() || opensFor(mover)`, so a key-holder passes a locked door and `unlock` is how you let everyone else through. `getLock()` is a policy seam (the keyed family reads its warren's keyway); the starting bolt is the row's `locked:`. Registered as `Lockable`, `MixinApi.isLockable`. |
| `BistateMixin` | mixin (unregistered) | The shared guarded-boolean substrate under `Sealable` (open/closed), `Switchable` (on/off), and `Foldable` (folded/unfolded). One persisted boolean + a `typeof`-boolean guard, reached via protected `getState()` / `setState(value, label)`. Lives at the `lib/` root (`lib/Bistate.ts`), NOT in `lib/boundary/`. **Deliberately carries no `_mixinName` and is not in the `Mixins` registry** — shared implementation, not a queryable capability; consumers narrow on the concrete axis (`isSealable` / `isSwitchable` / `isFoldable`), never an `isBistate`. |
| `AdornableMixin` | mixin | Container-side surface for non-portable attached Stuff (`getFixtures()` parallel to `getContents()`). Composed onto `Location` and `ExitableVessel` (not the bare `Vessel` base — every fixture consumer narrows on `MixinApi.isAdornable` first). |
| `AdornmentMixin` | mixin | Host-side back-reference (`adornedTo`) and not-portable invariant. Composed by content fixtures (e.g. `domain/lounge/NeonSign` = `Adornment(Branded(LightSource(Thing)))`) and the floor. Attached declaratively via the host's `adornments:` instruction field (see *Declarative adornments* below) or imperatively via `addFixture`. |
| `Boundary` | `lib/boundary/Boundary` (base) + `platform/thing/Boundary` (twin) | ⭐ **The edge object.** Two weak sides (`getSides`, `sideOf`, `otherSide`), a channel table (`channels: {open, shut}`) read by `transmissivity(channel, from, to)` / `transmissivityInto(channel, side)`, an `opening`, `stateWord()`, the keyword fold. The twin composes `Sealable` and `fixed` (a grating that never shuts). |
| `Conduit.ts` | vocabulary | `BoundarySide` (`A`/`B`), the closed `BOUNDARY_CHANNELS` (`movement · light · sight · sound · smell` — one per walk that reads it), `ChannelTable`, `BoundaryOpening`. The five conduit interfaces it used to hold retired with the boundary build. |
| `Window` | concrete `Boundary` subclass | `SoilableMixin(platform Boundary)`: passes light, sight, sound and smell and **never a body**. `baseTransmissivity`, one-way `aToBOverride` / `bToAOverride`, `colorTint` / glazing (the light walk multiplies `transmittanceColour` in), grime. Authored as `door:` on the exits facing it on both sides. |
| `BoundaryApi` | static API | `attach(boundary, ...sides)` (refuses a third side, an exit already on another boundary, the same exit twice), `detach(...sides)`, `mintKeyway()`. |

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
(a lock identity) lives on `BoundaryApi.mintKeyway` — **minting is an act on the
world** (it reaches `SecurityApi.uuid`), so it is the Api's; **issuing is the
lock answering about itself**, so `issueKeyTo` / `opensFor` stay instance
methods. **Bearer + re-key:** possession
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
- `edgeMinutes: number | null` — ⭐ how long this leg takes, in game
  minutes. ⚠ **Nothing in the kernel reads it**: ordinary movement stays
  instantaneous, and only the transport pack's Journey spends it. That
  is deliberate — *moving around must not feel like netlag* — and its
  consequence (the cost surface is opt-in) is named in
  [logistics.md](./logistics.md).
- `media: string[]` — locomotion media this exit admits (`'ground'`,
  `'water'`, etc.). Empty list = the **ground pace family**
  (`walk`/`sneak`/`run`) — widened from walk-only when the care↔speed
  axis landed, since anywhere you can walk you can sneak or run; non-ground
  media (climb/swim/fly) and wheeled conveyance still require an explicit
  `media` declaration, so old content stays backcompat-safe. See
  [locomotion.md](./locomotion.md#exitmedia).

⭐ **An exit is a thing you can name and look at** (harbors review 3).
`Exit` composes `Visible` + `Perceptible` over `Concealable`, and MQL
binds a direction to the EXIT itself — there is no `via.exit`. Its name
is its direction (`presentationPhrase`) and its keywords are the authored
ones plus its direction plus its short forms (`n`, `u` — `getKeywords`,
reading `NavigationApi.shortFormsOf`) — both ⚠ **derived**, and named so
on the methods. MQL reads an exit only through that interface; `go n`
binds the north exit because the exit answers to `n`. It describes itself: an authored `longDescription`
(the berth way's *"A gangway runs up to {{ ship }}'s deck."*), else
`plainWay()` — *a way north; the oak door is closed* — which never names
the far side (the map's evidence firewall). A verb that needs the door
on a way reaches it through `MqlApi.effectiveTarget`; `Stuff.isExit()`
is how a reader asks, since `Exit` sits on the locomotion import graph.

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

⭐ **A ship's two ways** (harbors) are `DeferredDestinationExit`s too,
both `LockableMixin` over it: `BerthExit` (`platform/idea/BerthExit`,
the `exits/berth` row) is the way aboard the ship herself puts on the
landing she makes fast at, named for her (`go hesper`), and gone when
she casts off; `AshoreExit` (on her entrance) leads to the landing she
lies at. Both read her gangway's lock (`lock ashore`, `unlock hesper`);
asleep, the berth way reads the lock she was left with. Both answer
`isMapped() → false` — a way that comes and goes with a ship is never
written to a map. There is no third derived *alongside* way (the
`AboardExit` that maritime derived was deleted): `Exit.alongsideNow()`
on the berth way answers her while she stands and lies at that landing.
See [ship.md](./ship.md).

`LotGateExit` is the case where the eager path is **exact** rather than a
class name: each lot's room is minted at its own identity, so the stored
`destinationTemplatePath` is that identity. That matters beyond honesty —
it is the path `Location.addExit` resolves the destination ZONE
from when deciding whether a non-cardinal direction is admissible.

#### Non-cardinal exits are a ZONE boundary, not a naming choice

`Location.addExit` refuses a non-cardinal direction when source
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
type is what makes `Location`'s cardinal-only-intra-zone
check possible — the subclass override awaits
`ZoneApi.resolveZoneForPath` (which may clone a zone Stuff on first
reference). The base implementation does only sync work; the async
type signature is for subclass overrides. `addBidirectionalExit` is
also async because it `await`s the two `addExit` calls internally.
Callers must `await`.

### `ExitableMixin` lookup — explicit only

`ExitableMixin` (`lib/boundary/Exitable.ts`) is composed by
`Location`, `SphericalLocation`, and `ExitableVessel`.
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

⚠ **Living in no contents, a door also had no light to be seen in** —
`VisionModality.canSee` reads the target's container, so every attached
door rendered as *"something"* (`open door` → *"You open something."*).
A door is now seen in the light of the viewer's room when that room's
obvious exits carry it — which covers a one-way door with no anchors.
Found by the rubber build's browser drive (the stove house, 2026-10-18).

#### ⭐⭐ `getObviousNeighbours()` — the five guards, once

`getObviousExits()` answers *which exits are here*; the propagation
walks need *which live rooms do my exits actually reach*, and every one
of them carried its own copy of the same five guards before they
reached for a neighbour:

1. an exit may name **no room at all** — the sandbox wardrobe passage
   names the WIRE, and walking it lands on a non-`Container`;
2. its destination template may have **many live clones** (a Warren hub
   exit names `/world/lounge/location/lounge` once a satellite exists)
   and the singleton lookup **throws** on it;
3. `getDestination()` may throw for its own reasons;
4. the far side may not be a `Container`;
5. the far side may have been **reaped mid-walk**, and a destroyed
   proxy answers every call with `undefined`.

⚠⚠ Each is a real incident, dated in `VisionModality`'s comments —
guards 2 and 5 each took `look` down for a whole room and the presence
fan with it. The list existed in **six** copies: the three modalities,
the audience gather, and `Atmospheric`'s two openings counters, whose
own comments admitted they were *"copied from the light walk"*. Six
chances to be missing one.

⭐ So it lives on the host that owns the exits, and the claim that makes
about every `Exitable` is simply true. `opts.doors` is the one axis
callers genuinely differ on: the propagation walks **skip** a doored
exit (the boundary conduit carries it, with its own transmissivity),
while the two openings counters want one whose door is **open**. A test
per guard, because putting them in one place is only worth something if
the one place is right.

⚠ Two callers keep their own reader on purpose: mine air reads
`getExits()` — *all* of them, because a hidden heading still has air in
it — and the forage census admits a destination that is **not** a
`Container`. See
[location-graph.md § Routing](./location-graph.md).

### Bidirectional exits

`Exitable.addBidirectionalExit(other, direction, opts?)` installs
both sides in one call:

- For cardinal `direction`, the opposite is inferred via
  `NavigationApi.invertDirection`. For semantic labels the caller
  passes `opts.opposite` explicitly.
- A `boundary` (opts) is attached to both sides — one door, two sides.
- `forward.inverse = back` and `back.inverse = forward` wired so
  `Mobile`'s arrival-message resolver can reach
  `exit.inverse?.direction`.

One-way exits use `addExit` directly and leave `inverse`
undefined.

### Declarative exits: `applyExits` instruction field

`exits` is an **instruction field** on `ExitableMixin` (declared via
`exits: { instruction: true }` in `fieldMeta`) — its YAML data is a
declaration applied to produce the runtime `exits: Map<string, Exit>`
collection, not stored as a value. The applier consumes a
`Record<string, ExitInstruction>` and installs each entry:

```yaml
# A Location template's data:
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
`onCreate` (load-time) and from `Mobile.traverse`
(traversal-time fallback).

### Lazy Exit destination resolution

An Exit can be authored with a live `destination` ref or a
`destinationPath` (templatePath string). The synchronous
`destination` getter consults the singleton cache first; if the
destination isn't loaded the getter throws — async-aware
callers (`Mobile.traverse`) `await
exit.resolveDestination()` first.

### ⭐⭐ `conditional` — is this the KIND of way that closes?

`Exit.isConditional()`, authored as `conditional: true` on the exit
**kind** row (a ford is a ford wherever it is laid), projected onto
`StoredEdge.conditional` and read by the router and the Cartographer.

⚠ It does **not** say the way is shut right now — `isBlocked()` says
that, and a conditional exit refreshes it at the traverse, which is
where a river belongs. It says the way is *the kind that closes*, which
is a fact about the kind and therefore authorable. The index is a
projection of authored rows and cannot know the water level; what it
can carry is the caveat a route plan owes its reader — *this way
crosses the ford at Kestrel; it is not always passable.*

⭐ It is on the base `Exit`, not on the transport pack's `FordExit`,
for two reasons: the projection and the Cartographer both read it
through the base shape, and **a second conditional class — a tidal
causeway — then needs no kernel edit**. Every exit can answer *am I the
kind that closes*; the honest answer for almost all of them is no.

⚠ `lint:location-graph` enforces it **by shape**: a kind row whose
class defines `refreshCrossing` (the conditional exits' shared protocol,
called from their async `canTraverse`) must declare it.
Deliberately not *"extends `FordExit`"* — a kernel gate must not
enumerate a pack's classes. The honest residue: a conditional class
that closes by some other means plans with no caveat until the shape
test is widened.

⭐ It also turned out to be a **risk** axis rather than a cost one —
the first place in this game where a fast uncertain way can be weighed
against a slow sure one. Nobody designed that; it fell out of
projecting one flag.

## Exit-kind templates (`<root>/idea/exits/<kind>`)

⭐⭐ **Every exit in the world is a clone of a kind row.** There is no
`new Exit(...)` branch left anywhere in the tree — an entry that names
no `kind:` clones the platform's **`/platform/idea/exits/passage`**, a
bare passage that authors nothing. That is the point: an exit built in
TypeScript could say nothing an author had written, and the code that
built one kept apologising for it in comments.

⚠ `passage` authors an EMPTY `media` deliberately. An exit with no
media admits the ground pace family (walk / sneak / run) with no
modality lookup; writing `media: [ground]` out loud would make every
unremarkable doorway depend on the mode roster being warm.

**`Exitable.installExit(kindOrExit, opts, configure?)` is the ONE
install path** for a code-minted exit: clone the kind row, run
`configure` (whatever the subclass's constructor used to take — a floor
number, a unit key, a warren reference), `bind` it to this edge,
install. The ROOM must be the caller, because `bind` is
participant-gated to a room party to the edge.

Passing an existing exit instead of a kind path RE-installs it — the
case of **an exit that moves with its host**: a vessel's `in`/`out`
pair and a sandbox crossing's passage are minted once at
`onCreate` and rebound as the host travels. That is what
`Exit.rebind(opts)` exists for, and it is the only sanctioned way past
`bind`'s "already bound" refusal.

An **exit kind** is an authored template a room's `exits:` entry
references via `kind:` — the authorable *nature of the passage*
(traverse messages, locomotion `media`, `wheelPassable`, concealment
defaults), distinct from a `Door` (the operable barrier object):

```yaml
exits:
  north:
    destination: /world/terminus/counting-houses/circle-parlor
    kind: /stuff/idea/exits/archway        # authored defaults for this edge
                                           # (absent → /platform/idea/exits/passage)
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

`platform/thing/Door.ts`. Composition: `LockableMixin(Boundary)` over the
platform twin, so `Sealable` (open/shut) and `Lockable` (keyway + bolt)
coexist on the one object. The Containable composition (from Thing) is what
lets a broken or unhinged door become an inventory item.

⭐⭐ **One door, two sides.** Each place's exit is a side of the door
(`exit.getBoundary()`); the door knows its sides (`sideOf`, `otherSide`). An
authored door is a row singleton: every side that authors `door: <row>`
attaches to the one instance through `BoundaryApi.attach`. Nothing owns
across the pair — the sides are weak refs both ways, so a destroyed exit's
slot heals to empty and a destroyed door reads as an open way.

### ⭐ A door that swings shut behind you — `closesBehind`

An authored flag (default `false`): a workshop door on a weight, or a
latched front door. Shut, it lets a body through (`Door.transmissivity
('movement')` answers 1) and is shut again after them; everything else
reads it shut — light, sight, smell, sound and the envelope's
open-doorway count — which is the point: a warm room keeps its heat with no
routine to close it. ⚠ **The bolt is not read there**: `Exit.canTraverse`
asks the lock first and refuses whoever the key does not open for, so a
key-holder walks through their own latched front door and a stranger is
refused at it. The potter's shop was the first; every keyed door a holding
or a warren hangs is one.

### ⭐ The free side — `freeFrom`

The identity path of the PLACE whose side opens with no key (`''` = keyed
from both sides). `Door.opensFor(mover)` is *the key fits, or the mover is
standing in `freeFrom`* — read off where the mover IS, never off which exit
it asked. A thumb-turn inside, a keyhole out: a guest let into a house is
never trapped and leaving your own house needs no key; the inside can also
`lock` and `unlock` keyless. The pump house is authored free from inside;
the code that mints a holding's doors stamps the inside room. (Owner's call,
2026-11-09 — before the boundary build the plans got the same effect by
authoring the way back with no door at all, which left a lock you could walk
round.)

### A fixed grating — `fixed`

On the platform twin: always open, and `close` refuses with the reason
(`Sealable.canClose`, the twin of `canOpen`). ⚠ No kind row: a grating is
`fixed: true` with a channel table on any boundary row — the shipped
`grating` kind had no consumer, and `lint:reachability` refuses a row
nothing reaches (documents sweep, 2026-11-10).

### Locking

> ⚠ **Since the boundary build (2026-11) the keyed exit family below is
> gone**: `KeyedDoorExit` is deleted and `FrontDoorExit` / `DormDoor` keep
> only their deferred destination — the lock is the DOOR the holding or the
> warren hangs in the way (§ Keyed ways). What follows is the lock model,
> which is unchanged; read "the keyed family" as "the keyed doors".

⭐⭐⭐ **A lock is a KEYWAY and a BOLT, and for most of this project it
was neither.** `lib/boundary/Lockable.ts` holds both facts; the lock
build (2026-10-09) is where they met.

**What was wrong.** There were two lock models and they were strictly
disjoint:

| | what it had | what it lacked |
|---|---|---|
| `LockableMixin` on `Door` | a **bolt** — a boolean anybody could throw | any key, credential or title check |
| `Lock` on three Exit subclasses | a **keyway** — `{keyway, technology}` + `opensFor` | a bolt, so the door was *permanently* locked |

The bolt-bearing `Door` Things carried no keyway, and the
keyway-bearing exits (`residence/src/idea/KeyedDoorExit.ts`,
`FrontDoorExit.ts`, `eternal-university/.../DormDoor.ts`) carry no
`Door` at all. ⚠ So `lock`/`unlock` shipped **afforded by nothing** and
the reachability sweep held them that way on purpose: conferring either
verb over either half would have afforded a verb that lies — `lock front
doors` would have locked a hall's doors for every player alive — and the
keyed family could never be left open for a guest.

**The model.** A real door has both facts and they are independent:
which key it accepts (the keyway — its identity; re-keying mints a fresh
one and old keys stop matching), and whether the bolt is currently
thrown. So:

```
canPass(mover)  =  !isLocked()  ||  opensFor(mover)
```

⭐ **The bolt refuses, the key excuses.** A key-holder never has to
`unlock` to get through, which is exactly how the keyed exits have
always behaved — so **unlocking is for letting everyone ELSE through**,
the capability the old model had no way to express.

The surface, and which half is which:

- `isLocked()` / `setLocked(value)` / `lock()` / `unlock()` — **the
  bolt.** State, no authority; the Exit family and the provisioning
  controllers move it directly. `setLocked` rejects non-booleans with a
  `TypeError` (a malformed template `locked: 1` crashes loudly at
  hydrate rather than being silently coerced).
- `opensFor(mover)` — **the authority.** A synchronous scan of the
  mover's reachable wallet (implant keychain, then a carried physical
  `Key` — never one lying in the room), safe to call from a traversal
  gate. The verb controllers ask it before touching the bolt;
  `Exit.canTraverse` asks it before refusing passage.
- `getLock()` — ⭐ **a policy seam.** The default builds the `Lock`
  value-object from this host's own `keyway` + `lockTechnology` fields;
  a host whose keyway is owned elsewhere overrides it. All three keyed
  exits do, reading the warren's cache, so **a holding's identity stays
  in one place and re-keying it re-keys every door at once.**
- **The starting bolt is the row's** — `locked: true` on the keyed door
  and the dorm door; a plain `Door` starts unbolted. (A `boltedByDefault()`
  hook did this while the keyed family were classes; it lost its last
  overrider when they became rows and was removed, 2026-11-10.) ⚠ The
  default is a field default, never a `postRegister` write: the Hydrator
  reflects the persisted `locked` in after construction, so the player's
  last act wins, where a lifecycle write would re-bolt the door every boot.

⚠ **An empty keyway opens for NOBODY.** A row authoring `locked: true`
with no keyway is sealed against the whole world — which is how the
university gate kept working with its content row untouched: *shut,
chained, and locked — the gown's, not the town's.*

> ⭐⭐ **What this model does NOT do, and where it is tracked.** A lens
> pass over the shipped design
> (lock-slate) **fails lens 1
> outright** — turning a key exercises no Discipline and nothing is
> derivable — and lens 6, because a key is cloned from nothing in a
> realm that has smelting and smithing. It also found that **you cannot
> defeat a lock**, so no lock story is makeable (lens 2), that
> `issueMasterKeyTo` has no production caller and no seat to hold it,
> and that nothing records who holds a key to a given door. ⭐ The
> Discipline that answers the first two is already designed —
> `lockcraft`, on policing-slate.

**Enforcement** lives in `Exit.canTraverse` (see *Exits* above): the
lock gate fires **before** the closed-door gate, reads only `this.door`
(never resolving the destination), and reports `'locked'`.

**The verbs.** `lock`/`unlock`
(`platform/idea/cmd/boundary/LockController.ts` /
`UnlockController.ts`, mirroring `Close`/`Open`) are afforded by
`MobileMixin.self`, beside `open`/`close` — the same gesture at the same
object. Each resolves any reachable `Lockable` through
`MqlApi.effectiveTarget`, which walks three rungs: the named object, the
exit's `Door`, and ⭐ the **exit itself** — the rung the keyed family
needed, since it carries its lock with no `Door` Thing. A mover with no
fitting key gets `controller-rejected:no-key`, and ⚠ the key is asked
**before** the bolt is read, so a stranger learns only that they have no
key rather than that the door is already locked.

> ⛔⛔ **`open north` and `close north` were broken for years**, and the
> lock build's drive is what found it. Resolving a boundary verb by
> DIRECTION threw inside MQL — *Converting circular structure to JSON* —
> because `MqlLogic.consensusVia` and the resolver's chain-walk dedupe
> key both compared a match's `via` by serializing it, and a direction's
> `via` holds a live `Exit` whose `Boundary.anchorA` points back at it.
> Both compare by identity now. ⚠ Nothing could see it:
> `OpenController`'s own test names the door by KEYWORD, so the
> direction branch of `effectiveTarget` had never run in either
> controller. Pinned by
> `platform/idea/api/__tests__/MqlLogic.circularVia.test.ts`.

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
`environment`), so
`applyAdornments` re-runs and rebuilds them on every hydrate. This
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
not-portable invariant. Composed by the floor, sconces, signs and
other fixtures.

`AdornableMixin.onDestruct()` walks `fixtures` and destructs
each via `StuffApi.destruct`.

## Boundary Substrate

`lib/boundary/Boundary.ts` — the base every boundary class extends (nothing
instances it; rows name the platform twin, `Door` or `Window`).

- **Sides.** `sideA` / `sideB`, weak instance refs to the two exits; A is
  the side attached first. `getSides()`, `sideOf(exit)`, `otherSide(exit)`.
  Written only by `BoundaryApi` (the `_attachSide` / `_detachSide` seams
  are gated to `BoundaryLogic`, as is `Exit._setBoundary`).
- **Channels.** `channels: { open: {…}, shut: {…} }` on the row, values
  0..1 per channel; a row names only what differs from the class default
  (all 1 open, all 0 shut — a door's numbers). ⚠ The vocabulary is CLOSED
  (`BOUNDARY_CHANNELS`) and an unknown key or an out-of-range value throws
  at hydrate: a misspelt `ligth:` must not read as dark. A channel exists
  where a walk reads it; `water` is absent until its first reader.
- **`transmissivity(channel, from, to)`** — the one question every walk
  asks; reads the open or shut table by `isOpen()`. `transmissivityInto
  (channel, side)` asks it for what flows INTO a side's place;
  `colourInto(side)` for the light's colour (white unless a `Window` is
  glazed). `passesBodiesWhenOpen()` is the window test.
- **`opening`** — `{ width, height }` in metres (default 1 × 2): inside a
  zone it must fit the face (§ The edge rules).
- **`stateWord()`** — `open` / `closed`; `Door` adds `locked`, which wins
  because the traverse refuses a locked door before a shut one.
- **`getKeywords()`** — authored keywords ∪ the short description's
  tokens (DERIVED, named at the method), so `heavy oak door` answers to
  `oak`. ⭐ The boundary is a candidate in MQL's `here` pool beside its way
  (once — deduped), so `wash window` / `open door` bind it and `open
  north` reaches it through `MqlApi.effectiveTarget`.
- **`canEvict`** — vetoes while a side's source room lives.

### The kinds — rows, not classes

`/platform/thing/boundaries/{door,gate,hatch,curtain}`: a door
(all open / all shut, 1 × 2 m), a gate (2 × 2, shut it still shows and
sounds through), a hatch (1 × 1 — the Delight creamery's cellar steps), a
curtain (drawn, it passes a body, sound and smell and blocks sight — the
bathhouse doorway). An authored boundary `extends:` one; ⭐ **a new kind of way is one
row and no code.** A window into a place is a `Window` row.

### `BoundaryApi`

`attach(boundary, ...sides)` takes each exit as a side — refusing a third
side, an exit already on another boundary, and the same exit twice;
idempotent for a side already attached. `detach(...sides)` lets each go.
`mintKeyway()` for a lock. That is all: attaching is the one two-object
act; everything a boundary does afterwards is a method on it.

## Authoring a way's boundary — `door:`

The exits-map key stays `door:` and names a boundary ROW of any kind. ⭐ A
way with a boundary is authored **either as `bidirectional: true` on one
side** (the far side authors nothing for that direction) **or as the same
`door:` on both**:

```yaml
# the bank
exits:
  pumphouse: { destination: …/pumphouse/location/house, door: …/thing/pump-house-door }
# the pump house
exits:
  out: { destination: …/wharfside/bank, door: …/thing/pump-house-door }
```

⚠ **D4 — the sides agree, or the doorless side is a wall.** Whichever side
installs second checks the far place's single way back
(`_reconcileBoundaryPair`): if one carries a boundary and the other none,
the boundary-less side is BLOCKED and the author is told through
`DiagnosticApi` (*a way is one boundary with two sides*). Never a silent
half-lock where leaving ignores the bolt — the pump house and the foundry
yard were both that before the build.

## The edge rules — every way inside a zone is true to the grid

Inside one zone every room is one cell, so every shared face is the same
size, and a way between two rooms of the zone is where two cells MEET
(`Location.edgeRefusal`, the `Exitable.edgeRefusal` hook — **every way**,
owner's call 2026-11-10; it was doored ways only at first):

1. the far place is the **adjacent** cell that way;
2. the far side's way back, if it has one, is the **opposite** direction;
3. the way's **opening** fits the face, and a boundary hung in it fits the
   way.

⭐ **The opening is the WAY's** (`Exit.getOpening()`, from the way's kind
row or authored per site; `null` = the whole face). An empty archway has a
size too, and what carries what through a way (size and carrying) asks the
way. A boundary's own `opening` is the object's size; it must fit.

⭐⭐ **A track — the long walk over open ground** (`kind:
/platform/idea/exits/track`, `Exit.spansOpenGround()`). The sparse country
(Rejection's outlying claims, Hearts Delight's lanes, Waponi's volcano)
places rooms several cells apart on purpose — coords are metres, and the
drilling geology is laid out on them. A track's far place lies **on its
bearing** (the ground bearing rounds to its compass point, within 22.5°;
`up`/`down` move the height that way), it may climb, **no room stands on
the straight line between**, and **nothing hangs in it** (a boundary needs a
shared face). The way back is still the opposite word. *A compass word over
open country is a bearing, never a promise of a straight road.*

Across zones — or between a holding's own frame and another — a way is a
**threshold**: the grids share no face and nothing is checked. A way that
breaks a rule installs **unbuilt** (*nothing lies that way yet*) and the
diagnostic names the row, the direction and the reason. `lint:locations`
(`wayEdges`) checks rules 1–2 and the track rules from the rows' `coords:`
and `kind:` in CI; rule 3 needs resolved `extends:` chains and the zone's
cell size, so it is the runtime's. The sweep that turned the rule on found
67 ways: ~25 were plain mistakes (a wrong sign on y, an off-by-one cell,
two rooms on one cell, a way back named the wrong way), fixed; the rest were
the sparse country, made `track`s on their true bearings (a few direction
words and three rooms re-aimed — listed in the build's sweep commit).

## Keyed ways — the minter mints the door

A door made by code is minted and **owned** by the code that made it
(`{ref: 'instance', lifetime: 'owned'}` maps):

- **A holding's interior keyed doors** — the plan names the way in as
  `kind: /system/residence/idea/exits/keyed-door` (a plain way now);
  `HoldingWarren.hangKeyedDoors` (after `wake`) mints
  `/system/residence/thing/boundaries/keyed-door` (bolted, latched), keys it
  to the parcel, attaches it to that way AND the way back, and makes the
  inside free. `refreshKeyway` re-keys every door it hung.
- **An entry door** — `CirculationWarren.hangEntryDoor(key, row, side,
  freeFrom?)`, called by `BuildingWarren` / `DormWarren` / `PlatWarren` from
  `entryEdgeFor` (the corridor or lane side) and `wireHubExit` (the entry
  room's way out, the free side). One door per holding, kept across its
  sleep; re-stamped with the keyway when `refreshProvisioned` rebuilds the
  cache. Rows: `residence/thing/boundaries/keyed-door` (a unit's front
  door), `duncan-hall/thing/dorm-door`, `residence/thing/boundaries/lot-gate`
  (a gate, standing open).
- ⚠ **The gangway is not yet a boundary.** `BerthExit` and the deck's
  `AshoreExit` both delegate their lock to `Ship.gangwayLocked`, so they are
  already one lock with two faces; making it a minted boundary is the
  follow-up (boundary-slate § 1). `Exit.wayState()` reads an exit's own lock
  when it has no boundary, so the screens show it.

## A way's state on the wire — `WayRecord`

`Exit.wayState()`: no boundary → `open`; a boundary that passes no body
even open → `window`; else the boundary's word, `locked` > `shut` > `open`.
`Exit.wayRecord()` → `WayRecord { direction, state, crossesZone,
destination?, boundary?: { stuffId, name, primaryKeyword? } }`
(`@saxonberg/types`; `StuffExitRecord` and `WhereaboutsExit` are aliases).
Both producers — the room card's `exits` subscription and the whereabouts
push — emit it, so the navigator, the chips, the phone bar and `look`'s
exits line (*north (the oak door, locked)*, *west (the west window)*)
cannot disagree.

## Window

`platform/thing/Window.ts` — a boundary of glass between two places. A
window **into a place** is a way: `door:` on the exit that faces it, on
both sides; `look west` sees the hillside through it, `go west` is refused
*the west window is not a way through*. ⭐ A window onto the **sky** is not
a boundary at all — it is glass in the place's `enclosure:`
([construction.md](./construction.md)); a window with nothing behind it is
prose. The glazier's hut's west window is the first (the Rejection hut moved
to (1,2) so its doorway and its window each join adjacent cells).

Movement is 0 open or shut; the other four channels are the glass —
`baseTransmissivity`, or a one-way override for what flows FROM side A or
B, scaled by grime; nothing while the shutter is shut. Glazing / `colorTint`
colour the light (`transmittanceColour`). A window's grime rides each
room's record keyed `way:<direction>` (the fixtures capture slice).

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

The global `switch` verb (`cmd/device/switch.yaml` +
`platform/idea/cmd/device/SwitchController.ts` — the **`device` category**, not
`boundary`) targets any `Switchable` in reach rather than a specific
class: `switch beacon on`, `switch lamp off`, or a bare `switch lamp` to
flip it (the `toggle` word is deleted: one view, one word).

The shipped consumer is the crossing's `Beacon` (`obj/Beacon.ts`, a
`Switchable(Propertied(Thing))` pedestrian signal — on = WALK,
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
`lib/reserve/Reserved.ts` precedent). Each toggle mixin is then a thin naming
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

- Exits: `direction`, `destinationPath`, flags. The boundary reference
  is runtime (re-attached as the exits map applies). Every exit is
  explicit — the zone no longer derives any.
- Doors: `open` (from Sealable) plus `locked` (from Lockable) — both
  boolean scalars, re-seeded from the row (or re-bolted by the minter) at
  stand-up; only a window's grime (`runtimeState`) rides the room's
  record, keyed `way:<direction>`.
- Switchables: `on` (from Switchable, stored via `BistateMixin`'s
  guarded boolean) — a single boolean scalar.
- Boundaries: the channel table, the opening, and subclass fields
  (Window's transmissivity, overrides, glazing).
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

The harbors build (MR !511) added `BerthExit`/`AshoreExit` (P4–P6), made
both unmapped in review round 1 (`4cce826da`), and made an exit a Visible
thing MQL binds directly (review 3); master's `AboardExit` stayed deleted
at the paint merge (`5cfd1a5db`).

**One shared edge, each side the exit** — the boundary build
(`reqs/boundary`, 2026-11, MR !559; requirements and plan retired at its
sweep). The four models of a way — a `Door` hung on an exit, the
`BoundaryAnchor` pair surfacing a window on each room, exits with a lock of
their own (`KeyedDoorExit`/`FrontDoorExit`/`DormDoor`), and a `Window` that
wired its own hosts (`attachedHosts`) — folded into one `Boundary` whose
sides are the exits. Retired with it: the five conduit interfaces,
`DoorBearingMixin`, `Door.attachedTo`, `Adornable.getFixtureBoundaries`,
`Exitable.getExitDoors`. Added: the channel table, `freeFrom`, the edge
rules, `WayRecord`. Decisions the build made beyond its plan: the boundary
stays an MQL candidate beside its way (object verbs — `wash window`,
`glaze` — bind it); keyed doors latch (`closesBehind`, the bolt is the lock
gate's alone); the lot gate stands open; a direction word in a boundary's
name is the way's, not folded into its keywords; a way's prose is
parenthetical (*through the front doors (open)*) because the way cannot know
its boundary's number; a shut vessel drops its synthesized `out` from the
walks. Review round 1 (owner): the opening and the edge rules are the WAY's
and hold for every way, and the `track` kind was added for the sparse
country (67 ways swept to 0). Verified by the blocking drive
`packages/wire/tests/boundary.dirty.wire.test.ts` (5/5) and the browser walk
`e2e/tests/boundary-walk.spec.ts` (3/3), which found `map` crashing on a
deferred way (fixed in `Perceiver`). Left: the gangway as a minted boundary
and the rest of boundary-slate.

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

## ⭐⭐ The two far-side gates — `unpublished` and `unbuilt` (2026-10)

`TraversalGate` gains two words for facts about the FAR side, and both
run **before the lock gate** so a wall reads as a wall rather than as a
locked door. Both follow the lock gate's own stated rule: **they never
resolve the destination.**

| gate | what it means | read from |
|---|---|---|
| `unbuilt` | the destination names a row that does not exist | a transient flag on the `Exit` |
| `unpublished` | the far side's parcel says its content is not open | `ParcelApi.isPathPublished` — sync, longest-prefix |

### A dangling exit is a DIAGNOSTIC, not a boot crash

`StuffApi.singleton` throws *Template not found* for a missing row, and
eager rooms come from each pack's `boot:` list — so **one mistyped
destination anywhere in content took the whole world down**, wrapped as
*"failed to clone"*, from a stack trace naming the framework rather than
the row. An author could not see it coming and could not read it when it
arrived.

`_applyExitSpec` now installs the direction **unbuilt** instead: `look`
still names it (the room reads as authored), `_destinationPath` is kept,
`canTraverse` answers *"Nothing lies that way yet."*, and
`DiagnosticApi` tells the author which row and which direction on their
own channel.

⭐ **It heals.** An existing unbuilt stub is **replaceable**: create the
row the author meant, re-hydrate, and the real exit lands where the stub
was. Without that the idempotency check would compare the stub's
now-resolvable destination against the incoming one, find them equal,
return a no-op — and the fix would look like it had not worked, which is
worse than the original defect. ⚠ The reverse is refused: a **working**
exit is never traded for a stub.

⚠ Unhook from the exits map **before** destructing the stub.
`StuffApi.destruct` does not remove it, so the map would keep handing
out an inert exit and every read of it would no-op to `undefined`.

⚠⚠ Caught **narrowly** — by the one message `StuffApi.clone` throws for
a missing row, with everything else rethrown. A failure inside a
destination's own `onCreate` is a different fault and must stay loud;
swallowing it would turn a broken room into a mysteriously unreachable
one. And by a **catch rather than a pre-check**: a `Template.findByPath`
before every `singleton` adds a store round-trip per authored exit on
the hydration hot path (199 edges at boot) and makes exit installation
depend on a reachable store — which took six suites red on
`isConnected is not a function`, the same hazard a live server hits
during early boot.

⚠ `Exit._unbuilt` is **transient**: a fact about the world as loaded,
re-derived by the next hydrate. Persisting it would outlive the defect
it describes.

See [location-graph.md](./location-graph.md) and
[parcel.md](./parcel.md).
