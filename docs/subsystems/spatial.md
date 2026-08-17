# Spatial Subsystem

How Stuff relates spatially: the **containment** chokepoint, **surface**
placement, and the **locomotion** that moves actors between rooms — the
substrate now in `lib/spatial/` (Container, Containable, Mobile,
Surfaced, Sealable). The room/coordinate/zone **geometry** that used to
live here has moved to `lib/location/` — see the sibling doc below.

Sibling docs cover related ground without overlap:

- [location.md](./location.md) — the room/coordinate/zone **geometry**
  (`Location`, `CartesianLocation`, `SphericalLocation`, the coordinate
  mixins, `CartesianZone`, `SphericalZone`, `ZoneApi` resolution) plus
  the Warren elastic-graph (MultiLocation) substrate. All in
  `lib/location/`.
- [zone.md](./zone.md) — the Zone-hierarchy roots (`Zone`,
  `SpatialZone`, `FolderZone`) and the field-inheritance walk. The
  hierarchy lives in `lib/zone/`; the concrete coordinate zones
  (`CartesianZone`, `SphericalZone`) live in `lib/location/`.
- [templates.md](./templates.md) — clone pipeline,
  `ZoneApi.resolveZoneForPath`, the folder/leaf invariant on `domain`.
- [state-model.md](./state-model.md) — the universal `zone` field
  stamped on every Stuff, why it follows template path rather than
  current container.
- [messaging.md](./messaging.md) — Scene composer, sensor routing.
  Movement narration sits on top of this.
- [prose.md](./prose.md) — the Liquid templating used by
  `Exit.messageOut` / `messageIn` and the Mobile default settings.
- [light.md](./light.md) — the Light & Boundary subsystem that
  layers on top of this one. `Door` is now a `Boundary` (closed
  doors block light, not just movement); `Adornable` composed onto
  `Location` and `ExitableVessel` is what hosts `BoundaryAnchor`
  fixtures.
- [properties.md](./properties.md), [mixins.md](./mixins.md) — the
  general mechanics that let mixins carry persistent fields.

## The Cast

The room / coordinate / zone classes (`Location`, `CartesianLocation`,
`SphericalLocation`, the coordinate mixins, `CartesianZone`,
`SphericalZone`, `ZoneApi`) live in `lib/location/` — see
[location.md](./location.md); the `Zone` hierarchy roots are in
[zone.md](./zone.md). This doc's cast is the containment / movement
substrate in `lib/spatial/`:

| Type | Kind | Role |
|---|---|---|
| `Vessel` | top-level branch | A *container-object* — a thing that holds things, at any scale (bag → cart → ship). Carries Thing's describable-physical baseline directly (`Visible` + `Perceptible` + `Tangible`) so a describable container needs no re-added `Visible`, plus Container + Containable + `Atmospheric` ([biome.md](./biome.md)) — but is **not** a `Thing` subtype (a ship isn't pocketable); carry/drag/ride is emergent from mass vs. a bearer's capacity ([encumbrance.md](./encumbrance.md)), never a type flag. Sibling of Thing / Location / Idea / Agent / Shadow; lives in `lib/stuff/`. Carries a `transmissionFactor` field (the encumbrance attenuation, default 1.0). `Adornable` is **not** on the base — it lives on `ExitableVessel` (the only subclass needing fixtures). Pure containers (Box, Backpack) do NOT compose Atmospheric and are skipped by the outward-walking biome chain. |
| `ContainerMixin` | mixin | Inventory side: `addContainable` / `removeContainable` / `getContents`. |
| `ContainableMixin` | mixin | Lives-inside side: `environment`, `setContainer`. |
| `SurfacedMixin` | mixin | Surface placement: the auxiliary `restingOn` pointer + `canRest`. |
| `MobileMixin` | mixin | Locomotion: `traverse` (async)/`teleport` and movement narration. |
| `SealableMixin` | mixin | Binary `open` state (predicate `isOpen()`). Used by `Door` and `Window`; reusable for chests, trapdoors, envelopes. Generic, stays in `lib/spatial/`. |
| `ContainmentApi` | static API | The single public surface for moving Stuff between containers. |
| `NavigationApi` | static API | Direction normalization, aliases, grid offsets, inverses. |

## Class Hierarchy

The room/zone **geometry** tree (`Location` → `CartesianLocation` /
`SphericalLocation`; the `Zone` / `SpatialZone` / `CartesianZone` /
`SphericalZone` hierarchy) is in [location.md](./location.md) and
[zone.md](./zone.md). The branches this doc covers:

```
Stuff (one of seven top-level branches — see architecture.md)
  ├── Idea
  │     └── Exit                    (data + canTraverse() guard, lazy destination)
  ├── Location                      (Adornable + Container — the container host; concrete rooms in location.md)
  ├── Thing                         (ContainableMixin(Stuff))
  │     ├── Boundary                (Visible + Perceptible)            ← see light.md
  │     │     ├── Window            (Sealable + Light/Sight/Smell/Sound Conduits; `attachedHosts` identity refs)
  │     │     └── Door              (Sealable + Light/Sight/Movement/Sound/Smell Conduits)  ← retrofit
  │     └── BoundaryAnchor          (Adornment)                         ← see light.md
  ├── Vessel                        (Tangible + Atmospheric + Container + Containable)
  │     └── ExitableVessel          (DoorBearing + Exitable + Visible + Adornable)
  └── Agent                         (Avatar / NPC / vehicle layer Mobile + … on top)
```

The fundamental split for spatial relationships:

- **Locations are containers but not containables** (rooms don't
  live anywhere).
- **Things are containables but not containers** (or, in Vessel-light
  cases, both — a chest *could* be modeled as a Thing-with-Container,
  but anything with navigable interior is a Vessel).
- **Vessels are both** — that's their distinguishing trait. Mobile
  places.
- **Doors are Things** with an attach/detach relationship to Exits,
  not a pure Containable role.
- **Agents are containable + mobile** — they live inside a Location
  and can travel.

Capability checks (`isContainer`, `isContainable`, `isExitable`)
cross all seven branches; class-identity checks (`instanceof`) are
for role.

## Containment

Source: `lib/spatial/Container.ts`, `lib/spatial/Containable.ts`,
`api/containment.ts`.

The most security-hardened slice of the codebase. Any code path that
can mutate "what's inside what" goes through one chokepoint, by design.

### The chokepoint

Three lockdown decorators stack across two methods to make
`ContainmentApi.move` the **only** legitimate way to change inventory.

`Container.addContainable` / `removeContainable`
(`Container.ts:98-113`) are:
- `@Final` — no subclass override (out-of-sync inventory is catastrophic)
- `@Unshadowable` — no shadow bypass
- `@CallSecurity(CalledFromSetContainer)` — reachable only from inside
  `Containable.setContainer`

`Containable.setContainer` (`Containable.ts:91-104`) is:
- `@Final`, `@Unshadowable`
- `@CallSecurity(FromContainmentApi)` — reachable only from
  `ContainmentApi.move`

The result is a hard guarantee: every change to `environment` /
`inventory` flows through `ContainmentApi.move`, which means every
change runs the invariant gates and the witness hooks below.

Detach is `ContainmentApi.move(item, null)`. A direct
`setContainer(null)` is rejected by policy.

### `ContainmentApi.move` pipeline

`ContainmentApi.move(item, to)` (`containment.ts:73-119`):

1. **Pre-flight invariants** (when `to !== null`):
   - Exitables can only land inside other Exitables. (Closes the
     "carry a chest with someone in it" exploit.)
   - Exitables cannot cross zones via containment. The `zone` field
     follows whichever template spawned the item; moving it into a
     foreign zone would silently desync.
2. **Veto hooks** (declaration-of-care order: item, source, dest):
   `canMove` on the item, `canRemoveContainable` on the source,
   `canAddContainable` on the destination. First veto wins, throws
   `ContainmentError`.
3. **State mutation** through `setContainer`. The chokepoint runs the
   three cross-object updates atomically:
   `oldContainer.removeContainable(this)` →
   `newContainer.addContainable(this)` → `this.environment = container`
   (`Containable.ts:94-104`).
4. **Notification hooks** (post-mutation, never veto):
   `onContainableRemoved` on source, `onContainableAdded` on
   destination, `onMoved(from, to)` on the item.

Witness hooks are optional methods on the relevant interface — declare
the ones you want, omit the rest. `containment.ts:157-165` dispatches
them via `typeof === 'function'` so a shadow defining a hook
participates without a `MixinApi.hasMixin` precheck.

### `ContainmentError`

Thrown for **programmatic** contract violations: invariant breaks and
hook vetoes from internal code. User-facing commands (`go`, `get`,
`drop`) validate beforehand and produce friendly messages — the API
never returns boolean success flags. See
[antipatterns.md](../antipatterns.md) for the
"don't manually call setContainer + addContainable" rule.

### `zone` is NOT restamped on move

By design (`containment.ts:64-69`). The authoritative source for zone
membership is the `domain` template path at clone time, not the
current container. Cross-zone movement of Exitables is blocked at
pre-flight; non-Exitable Stuff (an Avatar walking from a Narnia room
into a Caves room) keeps its original `zone` reference, which is
the right answer — see [state-model.md § Stamped-on-Stuff Fields](./state-model.md#stamped-on-stuff-fields).

### Detached Stuff (`environment === null`)

A `Stuff & Containable` whose `environment === null` is **detached**
— not in any container, not anywhere in the world. Detachment comes
up in three normal situations: a Stuff just constructed via
`StuffApi.create` but not yet placed; a door that has been removed
from its Boundary anchor; an item whose container was destroyed
mid-frame.

Detachment is a normal state, not an error. Subsystems that walk
container chains, route messages, or compute perception MUST handle
the null-env case without throwing. The matrix below is canonical;
new code that touches a detached input has to land somewhere on it.

| Subsystem | Site | Behavior on null env |
|---|---|---|
| MQL scope walks | `api/mql/resolver.ts:165, 520, 817, 836` | Silently skip the detached Stuff; resolver continues with what's left. Empty results are normal. |
| MQL scope-walk helper | `api/mql/scope-walk.ts:116, 147` | Returns `[]` when the giver has no environment. |
| MQL predicates | `api/mql/predicates.ts:61, 65` | `inLocation` / `peers` return `false`. |
| Command scoping | `lib/command/CommandGiver.ts:367` | A detached giver's environment-bucket is empty; recency stack reflects only `self` + `inventory`. |
| Perception (canSee) | `lib/perception/modalities/VisionModality.ts:148` | `VisionModality.canSee` returns `false` for a detached target. The shadow seam still fires for per-viewer overrides. |
| Mudlog routing | `api/message.ts:237, 411` | `MudlogApi.peers` walks no further. `messageContainer` warns once and returns; nothing is delivered. |
| Boundary (ExitableVessel) | `lib/boundary/ExitableVessel.ts:121, 161, 185` | `getExit` returns `undefined` for a detached vessel. The vessel is still reachable through its interior. |
| Light source notification | `lib/perception/LightSource.ts:156-168` | A detached LightSource emits no notifications. |
| Containment move | `api/containment.ts:107` | Detached → null is a no-op; detached → present follows the regular path with no `from` to remove from. |
| Mobile traversal | `lib/spatial/Mobile.ts:286` | A detached mover can `traverse`; no leaving-message fires (no `previous` to address). |
| Login | `obj/Login.ts:125` | If an avatar is detached at login time, the login frame announces "you are nowhere" and routes to `/void`. |

By behavior class: silently-skip / empty-result for the MQL stack and
command scoping; `false` for `canSee`; `undefined` for boundary
queries; warn-and-return for `messageContainer`. **Nothing throws.**
Code that throws on null-env is a bug — file the regression as a
matrix-invariant violation. Regression tests live in
`lib/spatial/__tests__/Containable.nullEnv.test.ts`.

### Declarative-content `container:`

`ContainableMixin` declares `container: { instruction: true }`
plus an `applyContainer(path)` Phase 2 applier. When the source
template's `data` block carries a `container: /some/singleton-target`
string, the Hydrator resolves the target via `StuffApi.singleton(path)`
and moves self into it via `ContainmentApi.move`.

**Compare-and-move idempotency.** `applyContainer` no-ops when the
current container's templatePath matches the declared path; otherwise
the move fires. The shape supports both fresh-clone placement (current
container is null → declared) AND `Avatar.restore()` re-move semantics
(current is `/A`, declared is `/B` → move) with no flag.

**Singleton-target constraint (v1).** The target template's class
MUST compose `SingletonMixin`. The invariant is enforced at template-
save time by `TemplateApi.validateSingletonContainerTarget`, fired
through the `DomainHook` aroundSave alongside the folder/leaf
validator. A non-singleton target throws at save with a clear
diagnostic naming both the source and target paths.

This is a known v1 limitation. Two reasons to relax it later:

1. **Class redefinition.** A template that saved when its target
   composed `SingletonMixin` may not still satisfy the check after a
   class refactor; an eager save-time check is stricter than the
   actual runtime requirement.
2. **Non-singleton container use cases (multirooms).** Legitimately,
   multiple live instances of "the same" container template are
   useful (multirooms, instanced dungeons). The eager check
   forecloses on this without language for addressing a specific
   instance.

The lazy direction: `applyContainer` already uses `StuffApi.singleton(path)`,
which surfaces the same diagnostic class at first-hydrate time when
the path doesn't resolve to exactly one instance. Removing the eager
validator would defer the check to that natural site, accepting the
"fail at first clone instead of at save" trade-off in exchange for
not over-constraining authoring. When the multiroom story lands,
this constraint should be re-evaluated alongside the addressing
scheme (templatePath alone is 1:1; multirooms need richer keys).

For populating a Container with children declaratively (the inverse
direction), see `PopulatesMixin` and the `populates:` instruction
field — covered in the [containment](#containment) section's
[Hydrator contract](../subsystems/templates.md#the-hydrator-contract)
cross-reference.

## Surfaced and surface placement

`SurfacedMixin` (`lib/spatial/Surfaced.ts`) is the sibling-marker
mixin for Stuffs that support items resting on them — tables,
shelves, the surface of a floor when authored as such. The
on-vs-in distinction is structurally modeled (an apple on a desk
and an apple in a chest live in different relationships) without
conflating with the existing Container's interior contents.

### The auxiliary-pointer model

Containment stays hierarchical and exclusive: every Containable
has exactly one `container` (or none). "Resting on" is an
**orthogonal optional pointer** on the item, not a replacement.
An apple on a desk in a room has `container = room` AND
`restingOn = desk` — both relationships are real, neither is
conjured by the other.

| Scenario | `container` | `restingOn` |
|---|---|---|
| Apple on a desk in a room | the room | the desk |
| Apple in a chest in a room | the chest | null |
| Apple on the floor in a room (Surfaced floor) | the room | the floor |
| Apple in inventory | the actor | null |
| Apple on a desk inside a chest | the chest | the desk |

The "apple in the room" intuition is preserved — the room's
`getContents()` includes the apple directly, not through any
indirect "items on items in here" walk. The fact that a desk is
supporting it is auxiliary information.

A desk-with-drawer composes BOTH `Container` (for the drawer-
as-part, with `container = desk`) AND `Surfaced` (for the apples
on top, with `container = room` and `restingOn = desk`). The two
collections are independent and non-overlapping.

### `Containable.restingOn`

`Containable` carries `_restingOn: (Stuff & Surfaced) | null`
as a **runtime-only instance ref**. The accessor
`getRestingOn()` returns the ref with an R2.3 self-heal (clear
on destructed supporter); the privileged setter
`_setRestingOn(surface)` is gated by
`@CallSecurity(FromContainmentApi) @Final @Unshadowable` —
reachable only from `ContainmentApi.placeOn` /
`ContainmentApi.move`.

Not persisted: on server restart, an apple's container is
preserved (the apple is still in the room) but the on-surface
relationship resets. The tradeoff is intentional. An identity-ref
templatePath stamping would persist cross-restart, but only
resolves unambiguously for singleton supporters — which
constrains the natural sandbox case of multiple identical
chairs / tables authored in a single area. The cross-restart
loss is small (items reappear in their container, just without
on-surface precision); when content earns persistent
on-surface state, that build picks the right shape (likely
an instance ref with stuffId stamping at save time).

### `ContainmentApi.placeOn(item, surface)`

The on-surface analogue of `move`. Resolves the surface's
container as the target environment, runs `canRest`, calls
`move(item, env)`, then stamps the auxiliary `restingOn`
pointer:

```typescript
ContainmentApi.placeOn(apple, desk);
// internally:
//   1. targetEnv = desk.getContainer()         // the room
//   2. assert desk.canRest(apple)              // host gate
//   3. ContainmentApi.move(apple, targetEnv)   // ordinary move
//   4. apple._setRestingOn(desk)               // restamp
```

`move`'s own contract is unchanged. The hidden invariant `move`
enforces: when the container actually changes, any existing
`restingOn` is cleared. So picking an apple up off a desk (a
move into the actor's inventory) clears `restingOn`
automatically; the on-surface state isn't carried into
inventory.

`placeOn` throws on programmatic-contract violations (surface
has no environment; surface rejects via `canRest`). User-input
failures are handled upstream by the `mustBeSurfaced` /
`mustBePutTarget` validators and the `PutController`'s
pre-flight `canRest` check (which produces friendly prose).

### `Surfaced.getResting()`

Lazy walk; no maintained forward collection. The surface walks
its own environment's contents and filters by
`getRestingOn() === this`. For typical room sizes the walk is
cheap; surfaces with very many resting items are content design
that hasn't earned a forward index yet.

### `Surfaced.userFacingDetail`

MQL keyword bridge, mirroring `SlotSpec.userFacingDetail` on
the slot subsystem. An author declares `userFacingDetail:
tabletop` on a Surfaced host and `put apple on tabletop`
resolves "tabletop" to the host via the Detailed-keyword path.
Pure MQL plumbing; surfaces don't gain Slotted semantics.

### `Surfaced.canRest(item)`

Per-host veto. Defaults to `true`; authors override for shape-
specific gates (fragile shelf rejects heavy items, sloped
surface rejects round items, wax tabletop rejects hot items).
Item-side gates intentionally don't exist — the authoring
intuition is host-side ("this surface rejects X") not item-side
("this item refuses Y").

### Surface-resting presentation: `ContainmentApi.looseContents`

Items appear in their enclosing container's contents listing
naturally (the apple is in the room; `room.getContents()`
includes it). But a room-contents *listing* shouldn't repeat an
item that's already represented by the surface it rests on — the
back-bar's bottles read "on the back-bar," not as loose room
clutter beside the patrons.

`ContainmentApi.looseContents(items)` is the shared presentation
filter: given a contents snapshot, it drops any item whose
`getRestingOn()` is itself in the set. It's a pure static (no
mutation of the walk) applied by **three** callers so the rule is
uniform: `look` and `sense` (the room branch) and the inspection
card (`Container.contents`).

The complementary half is the **drill-in**: examining a
`Surfaced` host (`look back-bar` / `sense back-bar`) reveals what
rests on it via an "── On it:" line built from
`Surfaced.getResting()`. So the resting items are out of the room
view but one examination away — the discovery path that keeps the
stock reachable without cluttering the room. (The earlier
count-aware "a desk, with a red apple on it" inline suffix —
once a dead `DescribeApi` helper — remains unbuilt; this is the
listing-partition shape that actually shipped, driven by the
crafting build's back-bar. See [crafting.md](./crafting.md).)

A concrete **`Surface`** class (`lib/spatial/Surface.ts`,
`SurfacedMixin(DetailedMixin(Thing))`) is the generic fixture you
set things on — a shelf, counter, table, or the bar's back-bar —
the first authored consumer of the surface substrate.

### Verbs: `put`, `give`

`put X in Y` calls `ContainmentApi.move`. `put X on Y` calls
`placeOn`. The verb's YAML carries `prepositions: [in, on]` on
the target arg; the preposition lands on `model.target.prep` and
the controller branches on it. With no preposition, the
controller infers from target capability (`Container` only →
`in`; `Surfaced` only → `on`); ambiguous (composes both) rejects
with a "put it in or on X?" prompt.

`give X to Y` is inter-Agent transfer via `ContainmentApi.move`
into the recipient's general Container. Items don't land in a
hand slot — see [embodiment.md § Hand slots are for activities,
not storage](./embodiment.md). The recipient is gated by the
`mustBeAgent` validator (target `instanceof Agent`).

## Locations, Coordinates, Zones

Moved out of `lib/spatial/` and into `lib/location/`. Rooms
(`Location` / `CartesianLocation` / `SphericalLocation`), the coordinate
mixins (`CartesianCoordinatesMixin` / `SphericalCoordinatesMixin`), and
the concrete spatial zones (`CartesianZone`, `SphericalZone`) — together
with `ZoneApi` resolution and the setter-with-side-effects pattern — are
documented in [location.md](./location.md). The base Zone hierarchy
(`Zone` / `SpatialZone` / `FolderZone`) is in [zone.md](./zone.md).

## Exits, Doors, Adornable, ExitableVessel

Moved out of `lib/spatial/` and into `lib/boundary/`. The full
architectural reference for these — exits, doors (now `Boundary`
subclasses), the `Adornable` / `Adornment` fixture surface, and
`ExitableVessel` (which composes `DoorBearing` and migrates the
vessel-door's `(vessel, env)` Boundary anchor pair on `setDoor` /
`onMoved`) — lives in [boundary.md](./boundary.md). Spatial keeps
the containers; boundary holds what connects them.

Locations and `ExitableVessel`s compose `AdornableMixin` so their
`getFixtures()` collection can host `BoundaryAnchor`s; that is
the only Boundary detail that lives in this doc. (A bare `Vessel`
is **not** Adornable — fixtures are an `ExitableVessel` concern.)
## Locomotion: `MobileMixin`

`lib/spatial/Mobile.ts`. The mover's side of movement. Composed by
Avatar today; future NPCs and vehicles will compose it too.

Base constraint: `MixinConstructor<Stuff & Containable>` — a mobile
thing that cannot be contained is nonsensical.

### Two modes: `traverse` and `teleport`

**`traverse(exit, mode)`** — exit-driven movement under a named
locomotion mode (short name: `'walk'`, `'climb'`, `'swim'`, …). `mode`
is required at the API. Pipeline:

1. **Mode-gate:** `exit.canTraverse(this, mode)` — checks blocked /
   door / `Exit.allowsMode(mode)` (mode's medium ∈ exit's `media`).
   Throws `ContainmentError` on rejection (programmatic-violation
   policy; player-input paths pre-check via
   `LocomotionApi.canTraverseExit`).
2. **Traversal vetoes:** `canTraverse` on the mover, `canExit` on the
   source room, `canEnter` on the destination room. First veto throws
   `ContainmentError`.
3. **Departure narration:** `announceDeparture(source, exit)`.
4. **State change:** `ContainmentApi.move(mover, destination)` —
   which fires its own containment-layer hooks (`canMove` /
   `canRemoveContainable` / `canAddContainable` / etc.).
5. **Arrival narration:** `announceArrival(destination, exit)`.
6. **Conveyance ripple:** for each occupant of the mover's slots,
   recursively `traverse(exit, mode)` if Mobile+Containable, else
   `ContainmentApi.move(occupant, destination)`. See
   [conveyance.md § Conveyance ripple](./conveyance.md#conveyance-ripple).
7. **Traversal post-hooks:** `onExited` (source), `onEntered`
   (destination), `onTraversed` (mover).

The `go` verb routes through `LocomotionApi.defaultModeFor(actor)` —
a three-tier chain (explicit setting → bodyplan default → `'walk'`)
— rather than the older `resolveSetting` shape. Literal mode verbs
(`walk`, `climb`, `swim`, `fly`, `ride`, `drive`) extend
`LocomotionControllerBase` with a one-line `modeName()` override. See
[locomotion.md](./locomotion.md).

**`teleport(destination, opts?)`** — Exit-less
move. Default narrates departure + arrival; pass `{ silent: true }`
to suppress both. Login spawning uses `silent: true`: a newly cloned
avatar shouldn't be announced as "vanishing from nowhere" or
"appearing out of thin air" before the player has even seen the room.

### Movement-message resolution

`announceDeparture` / `announceArrival` compose a Scene at
`act.move` (with Exit) or `act.move`
(without). The body resolution is a precedence chain:

`resolveDepartureMessage`, `resolveArrivalMessage`:

1. **`Exit.messageOut` / `messageIn`** — Liquid template, simplest
   override. `{{ mover }}` is bound to the mover's `Mml.actor`. Used
   by the vessel's synthesized exits for "Alice enters the wardrobe."
2. **Per-room hook:** `from.getDepartureMessage?(mover, exit)` /
   `to.getArrivalMessage?(mover, exit)` — returns
   `{ self?, peers? }`. Anything missing from the return value falls
   back to step 4. Same shape but `getTeleportOutMessage` /
   `getTeleportInMessage` for the no-Exit case.
3. **Mobile defaults:** `defaultDepartureSelf` etc. — read settings
   from the mover via `resolveSetting` (the cross-host helper from
   [shell-environment.md](./shell-environment.md)). The defaults are
   declared as a `static settings` schema on the mixin
   (`Mobile § settings`) — eight keys covering self/peers × depart/
   arrive × walk/teleport, with Liquid variables `{{ mover }}` and
   `{{ direction }}`.

The schema-on-mixin pattern means an Avatar (which composes
EnvironmentMixin) can override individual keys via the `settings`
command, while NPCs and vehicles render at the schema default through
`resolveSetting`'s non-Environment fallback.

`dispatchMovementScene` sends the resolved
bodies: `toSelf` only when the mover is itself a `Sensor` (a future
vehicle carrying passengers might not be); `toPeers` always.

### Witness hooks summary (movement)

| Hook | On | Fires from |
|---|---|---|
| `canTraverse(via)` | mover | `Mobile.traverse` (pre) |
| `canExit(mover, via)` | source room | `Mobile.traverse` (pre) |
| `canEnter(mover, via)` | dest room | `Mobile.traverse` (pre) |
| `onExited(mover, via)` | source room | `Mobile.traverse` (post) |
| `onEntered(mover, via)` | dest room | `Mobile.traverse` (post) |
| `onTraversed(via)` | mover | `Mobile.traverse` (post) |
| `canMove(to)` | item | `ContainmentApi.move` (pre) |
| `canRemoveContainable(item)` | source container | `ContainmentApi.move` (pre) |
| `canAddContainable(item)` | dest container | `ContainmentApi.move` (pre) |
| `onContainableRemoved(item)` | source container | `ContainmentApi.move` (post) |
| `onContainableAdded(item)` | dest container | `ContainmentApi.move` (post) |
| `onMoved(from, to)` | item | `ContainmentApi.move` (post) |

The traversal layer fires from `Mobile.traverse` and is exit-aware.
The containment layer fires from `ContainmentApi.move` and runs
regardless of whether an Exit was involved (so `teleport` and
`StuffApi.clone`-then-place still trigger the containment hooks).

### Conveyance ripple

After `ContainmentApi.move(mover, destination)` and
`announceArrival`, `Mobile.traverse` walks the immediate level of
the mover's slot map and ripples each occupant by capability:
Mobile occupants `traverse(exit, mode)` so they announce their own
arrival; non-Mobile occupants fall back to `ContainmentApi.move`
silently (the container model). A veto on a rider's `canTraverse`
leaves them behind without aborting the host. The chain self-
recurses through each Mobile occupant's own `traverse()` call, so
the saddle-on-horse-with-rider-with-backpack case just works.

The ripple makes mounts work: a horse moving carries any rider in
its mount slot, and a saddle on a horse with a rider in the saddle
ripples through both layers. See
[conveyance.md](./conveyance.md) for the full story.

### `engagedMode` and the slot-release witness

`MobileMixin` carries a runtime-only `_engagedModePath: string | null`
field (NOT in `fieldMeta`'s persistent entries — a reloaded actor wakes up
unengaged). `getEngagedMode()` resolves it via the singleton cache;
`setEngagedMode(mode)` stores `mode.getTemplatePath()`.
`isEngagedIn(mode | name)` is polymorphic — accepts either the
singleton or a short-name / full-path string.

`LocomotionApi.engageAround(actor, mode, exit, action)` is the
canonical engagement scope: it sets engagedMode, runs the inner
traversal, then conditionally clears engagedMode based on
`isTransientEngagement(mode, exit)` — passthrough modes (ride / drive)
stay set; walk / vehicular modes clear; climb / swim / fly clear if
the destination doesn't still expose the relevant enablement.

`Mobile.onSlotReleased(host, slotName)` is the witness invoked
synchronously by `Slotted.vacate`. The default body clears
engagedMode when the mode is passthrough AND the vacated host
composes the engaged mode's `conveyanceMixin`. A dismounting rider's
engagement clears automatically without controller-side bookkeeping.
See [locomotion.md § Engagement lifecycle](./locomotion.md#engagement-lifecycle).

### Location floors

Floors are first-class entities — `Adornment`s on the Location's
`Adornable` surface, composing `Postured` (see
[posture.md](./posture.md)). v1 ships no class-level default;
floor presence is authored per-Location in the `adornments` map.

```yaml
# Default Location includes the default floor:
adornments:
  floor: { extends: '/idea/surface/default-floor' }

# Voids omit it, marked with the `noDefaultFloor` opt-out so the
# migration script doesn't auto-add one:
data:
  noDefaultFloor: true
```

## Direction Vocabulary: `NavigationApi`

`api/navigation.ts`. The canonical direction table for cartesian
zones. Spherical and vessel directions live outside this table —
callers treat an `undefined` result as "not cardinal, try as a
semantic label."

The 10 cardinals: `north`, `south`, `east`, `west`, the four diagonals
(`northeast` / `northwest` / `southeast` / `southwest`), `up`, `down`.
Each has:

- A long-form name (canonical).
- Single-letter and two-letter aliases (`n`, `ne`, `u`, ...).
- A `[dx, dy, dz]` offset. **`y` grows north** (matches map-up
  convention); **`z` grows up** (standard).
- A canonical inverse (`north ↔ south`, `up ↔ down`, etc.).

Public methods (all on `NavigationApi`):

- `normalizeDirection(input)` → canonical `CardinalDirection` or
  `undefined`. Trims and lowercases.
- `invertDirection(direction)` → canonical inverse, or `undefined` for
  non-cardinals.
- `directionOffset(direction)` → `[dx, dy, dz]` triple, or `undefined`.
- `isCardinalDirection(direction)` → boolean.
- `cardinalDirections()` → readonly array of all 10.

Used heavily: `CartesianLocation.addExit` validates against
`isCardinalDirection`; `CartesianZone.getNeighbor`
goes through `normalizeDirection` and `directionOffset`;
`Exitable.addBidirectionalExit` uses `invertDirection` for inferred
opposites; `MobileMixin`'s arrival resolver uses `invertDirection`
for "arrives from the south."

## Loading and Lifecycle

Three runtime mechanisms that operate across the spatial layer at load,
traversal, and destroy time. They all rest on the singleton index in
`StuffApi` (the `byTemplatePath` Map).

### Singleton infrastructure

`StuffApi.singleton(path)` is a generic cache-or-clone tool. It walks
`byTemplatePath`, returns the unique live instance for `path` if any,
otherwise routes through `clone()`. Throws on multi-instance
collision (caller mixed `clone()` and `singleton()` on a class that
doesn't compose `SingletonMixin`).

`SingletonMixin` (`lib/stuff/Singleton.ts`) is the enforcement layer.
Composing classes refuse a second `clone()` for the same templatePath
— the pre-flight in `clone()` checks `MixinApi.hasMixin(cls,
Mixins.Singleton)` against `byTemplatePath`. This makes the framework
itself the single source of truth for "is there already an instance
for this path?", not the call site.

Today, `CartesianZone` and `SphericalZone` compose SingletonMixin —
each zone template has at most one live instance. `ZoneApi` gets path
caching for free via `StuffApi.singleton` and no longer carries its
own bookkeeping. Content authors who need "this Location is unique
by path" (Town Square, the Vault) compose SingletonMixin on their
specific subclass; the base `Location` is deliberately not singleton
so that future `MultiLocation` patterns (procedural deserts,
self-rearranging chambers) can produce many instances per template.

### Destroy choreography

`StuffApi.destruct(stuff)` runs `stuff.onDestruct()` and then
`stuff.destroy()`. Spatial-side `onDestruct` implementations:

- **`Location.onDestruct()`** detaches from the owning Zone
  (`zone.removeLocation(this)`), clearing coordinate-keyed
  indexes (CartesianZone grid, SphericalZone focusIndex).
  Concrete subclasses inherit. `ExitableMixin.onDestruct`
  (lib/boundary/) chains here after handling the exit-side
  teardown — see [boundary.md § Doors](./boundary.md#doors) and
  [boundary.md § Exits](./boundary.md#exits).
- **`Zone`:** refuse to destruct while non-empty — caller drains
  rooms first. `CartesianZone` additionally clears
  `derivedCache`.
- **`AdornableMixin`** (composed on Location and ExitableVessel) walks
  `getFixtures()` and destructs each. See
  [boundary.md § Adornable / Adornment](./boundary.md#adornable--adornment).

## Persistence Notes

The spatial subsystem is mostly auto-persistent through
`fieldMeta`'s persistent entries:

- `CartesianCoordinatesMixin`: `coordinates`.
- `SphericalCoordinatesMixin`: `coordinates`, `radius`.
- `SealableMixin`: `isOpen`.
- `Zone` subclasses: `name` (and `cellSize` on Cartesian).
- `Vessel`: `transmissionFactor` (the encumbrance attenuation; default
  1.0) — plus whatever its mixins contribute.

One intentional non-persistent:

- **`Containable.environment`** is NOT in `fieldMeta`'s persistent entries (see
  `Containable.ts:71-76`). It's a reference to another Stuff; the
  classes that compose Containable must declare a custom
  `persistenceHandler` to choose how to serialize the reference.
- **`zone`** on every Stuff is runtime-only today. The authoritative
  source is the `domain` template path — see
  [state-model.md § Stamped-on-Stuff Fields](./state-model.md#stamped-on-stuff-fields).

## Cross-References

- [architecture.md § Class Hierarchy](../architecture.md#class-hierarchy)
- [architecture.md § Available Mixins](../architecture.md#available-mixins)
- [antipatterns.md](../antipatterns.md) — `ContainmentApi.move` over
  raw `setContainer`, `creature.travel()` over `creature.move()`.
- [templates.md § Clone Pipeline](./templates.md#clone-pipeline) —
  zone resolution at clone time.
- [state-model.md § Paths and Collections](./state-model.md#paths-and-collections)
- [messaging.md](./messaging.md) — Scene composer used by Mobile.
- [prose.md](./prose.md) — Liquid templating used by `Exit.messageOut`
  and the Mobile default settings.
- [shell-environment.md § resolveSetting](./shell-environment.md) —
  cross-host setting resolution used by Mobile defaults.
- [light.md](./light.md) — the Light & Boundary subsystem on top of
  Door, Adornable, and the per-room walks.
- [perception.md](./perception.md) — viewer-aware-query pattern.
