# Spatial Subsystem

How Stuff relates spatially: the **containment** chokepoint, **surface**
placement, and the **locomotion** that moves actors between rooms — the
substrate now in `lib/spatial/` (Container, Containable, Mobile,
Placing, Sealable). The room/coordinate/zone **geometry** that used to
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
| `Holder` | `lib/stuff/Holder` | ⭐ **A thing that holds things and is itself held** — `ContainerMixin(Thing)` and nothing else. Minted by the base-class narrowing (2026-09-30) as the rung `Vessel` had been standing in for: the container chain is a **chain, not a fork** — `Location` (holds, is not held) → **`Holder`** (holds, is held) → `Vessel` (…and is ownable and concealable) → `ExitableVessel` (…and has air and a door). An immovable container — `Stock`, `BankCounter`, `DepotCounter`, `CheckRack`, `ConsignmentShelf`, `TpaTerminal` — extends `Holder`, not `Vessel`, because it is not a good. ⚠ Deliberately **twin-less**: nothing should clone a bare holder. Boundary vs `Receptacle`: a holder takes DISCRETE things, a receptacle takes BULK ([bulk.md](./bulk.md)). |
| `Vessel` | `lib/stuff/Vessel` | A *container-object* — a thing that holds things, at any scale (bag → cart → ship). Carries Thing's describable-physical baseline directly (`Visible` + `Perceptible` + `Tangible`) so a describable container needs no re-added `Visible`, plus Container + Containable — ⭐⭐ **and NOT `Atmospheric`: a bag is not a place.** The mixin composed here until the base-class narrowing build, giving a backpack, a till, a jar, a rack, a footlocker, a handcart, a bank counter and a barge each their own temperature, pressure, humidity, wind and biome; 37 rows over 15 composers, and none ever authored one of those fields. It is on `ExitableVessel` now ([biome.md](./biome.md)) — *a thing you can go inside is a place with air* — which is also the only place anybody's `context.location` can ever be a vessel, since a rider occupies a SLOT and stands in the room. A plain vessel is now a transparent step in the outward biome walk, exactly like a Box. ⭐ Since the base-class narrowing it is `ChattelMixin(ConcealableMixin(Holder))` — **a vessel is a holder that is also a GOOD**, so it carries exactly the two mixins that came off the `Thing` root, and `ContainerMixin(Good)` and `Vessel` are now the same mixin set. (An older reading of this row called it a top-level branch and *not* a `Thing` subtype; it traces through `Thing` and always did — *you can't pocket a ship* is a mass gate, not a type gate.) Carry/drag/ride stays emergent from mass vs. a bearer's capacity ([encumbrance.md](./encumbrance.md)), never a type flag. Lives in `lib/stuff/`. Carries a `transmissionFactor` field (the encumbrance attenuation, default 1.0). `Adornable` is **not** on the base — it lives on `ExitableVessel` (the only subclass needing fixtures). Pure containers (Box, Backpack) do NOT compose Atmospheric and are skipped by the outward-walking biome chain. |
| `ContainerMixin` | mixin | Inventory side: `addContainable` / `removeContainable` / `getContents`. |
| `ContainableMixin` | mixin | Lives-inside side: `environment`, `setContainer`. |
| `PlacingMixin` | mixin | **Placement**: the members this host offers (`placements`), the lazy `getPlaced` read, and the `canPlace` veto. Was `SurfacedMixin` until 2026-09-28. |
| `Placement` | singleton Idea | A **way of sitting**, as a row: `on` · `in` · `from`. Rows under any root; warmed by `PlacementCatalogue`. |
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
  ├── Thing                         (Wet + Visible + Detailed + Perceptible + Tangible + Containable)
  │     ├── Good                    (Chattel + Concealable)            ← portable matter: owned, hidden
  │     ├── Boundary                (Visible + Perceptible)            ← see light.md
  │     │     ├── Window            (Sealable + Light/Sight/Smell/Sound Conduits; `attachedHosts` identity refs)
  │     │     └── Door              (Sealable + Light/Sight/Movement/Sound/Smell Conduits)  ← retrofit
  │     ├── BoundaryAnchor          (Adornment)                         ← see light.md
  │     └── Holder                  (Container) — holds, and is held
  │           └── Vessel            (Chattel + Concealable) — a holder that is a Good
  │                 └── ExitableVessel  (Atmospheric + DoorBearing + Exitable + Adornable)
  └── Agent                         (Avatar / NPC / vehicle layer Mobile + … on top)
```

The fundamental split for spatial relationships:

- **Locations are containers but not containables** (rooms don't
  live anywhere).
- **Things are containables but not containers** — until they extend
  `Holder`, which is the rung that adds `Container` and is where a chest,
  a counter or a shelf belongs. ⭐ The chain does not fork: a thing with a
  navigable interior is an `ExitableVessel`, which is a `Vessel`, which is
  a `Holder`. ⚠ And the taxonomy is deliberately NARROW — four rungs, each
  adding exactly one claim. A fifth container branch is a design
  conversation, not a convenience.
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
direction), see `StagedMixin` and the `props:` instruction
field — covered in the [containment](#containment) section's
[Hydrator contract](../subsystems/templates.md#the-hydrator-contract)
cross-reference.

## Placement — where inside its container a thing sits

⭐⭐ **A container's contents are partitioned by where inside it a
thing sits, and the partition is named by the preposition a player
types.** A mug is **on** a desk; a steak is **in** a compartment with
its own air; a ham hangs **from** a hook. One relation, one
vocabulary, and the word IS the name — which is what lets somebody
who has learned `on` use `from` with no explanation.

`PlacingMixin` (`lib/spatial/Placing.ts`) is the host side: a Stuff
that offers one or more **placements**. `ContainableMixin` carries the
item side: the host it sits on and the member it sits under.

⚠ This was called `SurfacedMixin` until 2026-09-28 and modelled only
the `on` case (`restingOn` + `canRest`). The rename is not cosmetic —
the model had one instance of a general relation and no word for the
relation, so the second instance could not be expressed at all. See
the history note at the foot of this doc.

### The auxiliary-pair model

Containment stays hierarchical and exclusive: every Containable has
exactly one `container` (or none). A placement is an **orthogonal
optional pair** on the item, not a replacement. An apple on a desk in
a room has `container = room` AND `placement = { host: desk, name:
'on' }` — both relationships are real, neither is conjured by the
other.

| Scenario | `container` | `getPlacement()` |
|---|---|---|
| Apple on a desk in a room | the room | `{ desk, 'on' }` |
| Apple in a chest in a room | the chest | `null` |
| Apple on the floor in a room (a Placing floor) | the room | `{ floor, 'on' }` |
| Apple in inventory | the actor | `null` |
| Apple on a desk inside a chest | the chest | `{ desk, 'on' }` |
| Ham hanging from a hook in a room | the room | `{ hook, 'from' }` |

⭐ **The host is always a SIBLING in the same contents list** — that is
the structural insight the whole design rests on, and it is why the
persisted form is an *index into the container's own slice*
(`ContentPlacement`, below) rather than a reference.

The "apple in the room" intuition is preserved: the room's
`getContents()` includes the apple directly, not through any indirect
walk. That a desk is holding it is auxiliary.

A desk-with-drawer composes BOTH `Container` (the drawer-as-part,
`container = desk`) AND `Placing` (the apples on top, `container =
room`). The two collections are independent and non-overlapping.

### ⚠⚠ Two composition refusals, enforced at registration

`PlacingMixin.__validateComposition__` throws on first registration of
a concrete class (`assertComposable`, dispatched from
`StuffApi.register`) — never from the types, because the types cannot
express "not this mixin" and a refusal asserted from them is a refusal
that never runs. Both are proven by registering a class and catching
the throw, with a control class that must register cleanly.

1. **A Placing host MUST compose `ContainableMixin`.** The host has to
   live somewhere for the lazy `getPlaced()` walk to have an
   environment. ⭐ This is what keeps `Location` out: a room is not a
   thing you put things on — its floor is.
2. **A Placing host must NOT compose `ExitableMixin`.** A thing you can
   go inside cannot also be something you put things on, because
   *which exits does each region afford* is a question the model
   refuses to answer. Put a `Fitting` or a `Chamber` in its contents
   instead (that is how a boot works), or write the class.

### The `Placement` vocabulary — a member is a ROW

`platform/idea/Placement.ts`, warmed by `PlacementCatalogue` (the
`ReadingCatalogue` self-warming shape, boot entry in the platform
pack). Rows live at `<root>/idea/Placement/<name>` under **any** root,
so a capability pack ships a way of sitting with no kernel edit.

| field | read by |
|---|---|
| `name` | the catalogue index; `PlacingMixin.getPlacements()` |
| `prepositions` (primary first) | `resolvePlacement`, `put`'s offers |
| `encloses` | `Containable.getEnclosingScope`, `canPlace` |
| `prose` (one Liquid template) | `put`, `dry` |
| `heading` | the `look` / `sense` drill-in |

Three ship: `on` (`[on, onto]`), `in` (`[in, into]`, **`encloses:
true`** — the only one), `from` (`[from, on]`).

⭐ **`from` lists `on` as a SECONDARY word**, which is how `put ham on
hook` reaches a `from`-only host and gets told *"You hang the ham from
the hook"* — the world teaching the word without a tutorial. A
member's PRIMARY word always wins its own key
(`lint:placement-words` refuses two members claiming one primary).

⚠ **A cold catalogue degrades to the shipped behaviour, never to
silence.** A member with no live row answers to its own name
everywhere, so the worst case is *the model before the vocabulary*
rather than an unaddressable host. The consequence: a member's
SECONDARY words are its row's claim, so `put ham on hook` needs the
roster warm.

### The cost of a new way of sitting

> **One row, plus one word on every verb whose argument accepts a
> placement host.**

Today that is `put` and `dry`, and
`pnpm -C packages/server lint:placement-words --list` prints the
roster — which arguments accept a placement host, which members each
accepts, and which it does not. ⚠ The roster matters as much as the
gate: a verb that *should* accept a member and silently does not
refuses every such target **at the binder**, which no controller test
can see.

The gate holds the three things that are always wrong: a member with
no preposition, two members claiming one primary word, and `put` — the
universal placement verb — not accepting a member's primary word. It
deliberately does NOT check that every preposition on such an argument
is a member: `butcher <carcass> at <block>` says *where you do it*,
not *how it sits*, and there is no way to tell a verb's own grammar
from a stale placement word by inspection.

### `Containable`'s side: the pair, and the enclosing scope

`_placementHost` is an **instance (live) ref** (`{ ref: 'instance',
lifetime: 'weak' }`), so the R2.3 self-heal runs in the proxy get
trap: a host destructed since the last set reads as no placement at
all, whatever `_placementName` says. An identity ref by templatePath
was rejected because it resolves unambiguously only for singleton
hosts, which would constrain several identical tables in one hall.

- `getPlacement(): { host, name } | null` — normalises the pair.
- `_setPlacement(host, name?)` — gated
  `@CallSecurity(FromContainmentApi) @Final @Unshadowable`; reachable
  only from `ContainmentApi.place` / `.move`.
- ⭐ `getEnclosingScope(): Stuff | null` — **what stands between me
  and my container, for air, sight and reach.** The placement host
  when the member `encloses`, else the container. Read by
  `PerceptionLogic.canReach`, by `Thermal`'s **holder** read
  (`ambientScopeOf` → `enclosingCoolbox`: what you are IN outranks the
  room), and as the **step** of `Thermal`'s **air** walk.

⭐ Its two readers ask different questions, and the difference is one
step:

- *what holds me* — `Thermal.ambientScopeOf`, one hop, no walk;
- *what air reaches me* — `Thermal.airScopeOf`, which steps outward
  through this method until something is `Atmospheric`.

They were one call until `AtmosphericMixin` left `Vessel` in the
base-class narrowing build and a bag stopped pretending to be weather.
⚠⚠ A **worn** bag's container is the wearer (a `Creature` is a
`Container`), so bag → carrier → room is two hops — which is why the air
read is a walk and not a second step. See
[thermal.md](./thermal.md) § *Two questions, one step apart*.

**The pair IS persisted**, by the container's slice.
`ContentPlacement { hostIndex?, placement? }` in
`lib/persistence/PersistenceSlice.ts` records the host's **index
within the same contents list** plus the member name;
`PersistableLogic`'s placement pass re-`place`s on restore. (An older
version of this doc said the relation reset on restart. That stopped
being true when the slice learned to record it.)

### `ContainmentApi.place(item, name, host)`

The placement analogue of `move`, and it **calls** `move` — the
dependency runs one way.

```typescript
ContainmentApi.place(apple, 'on', desk);
// internally:
//   1. targetEnv = desk.getContainer()          // the room
//   2. desk.canPlace(apple, 'on')               // host veto → reason
//   3. ContainmentApi.move(apple, targetEnv)    // ordinary move
//   4. apple._setPlacement(desk, 'on')          // restamp
//   5. if Thermal, restamp()                    // see below
```

⚠ **`move`'s signature and meaning are unchanged.** It takes no
member, picks no default, and has no options bag. The one thing it
does with placement is the invariant it always had: when the container
actually changes, the pair clears. Picking an apple up off a desk into
inventory clears it automatically.

⭐ **Step 5 is not optional.** `move` is a no-op when the container is
unchanged (a mug moving from one desk to another in the same room), so
nothing would restamp — and a member that encloses means the thing's
ambient scope just changed.

`place` throws `ContainmentError` on programmatic-contract violations
(no environment; `canPlace` vetoes). User-input failures are handled
upstream by `PutController`, which reads the veto's `reason` into
prose.

### `Placing.getPlaced(name?)`

Lazy walk; no maintained forward collection. The host walks its own
environment's contents and filters on `getPlacement()`. With a `name`,
only that member's items. For typical room sizes the walk is cheap;
hosts with very many placed items are content design that hasn't
earned a forward index.

### `Placing.canPlace(item, name): VetoResult`

Per-host gate, replacing the old boolean `canRest`. Default body, in
order: the name is not offered → `no-such-placement`; the member
`encloses` and the host is a closed `Sealable` → `shut`; else ok.
Authors override for shape-specific gates (a fragile shelf rejects
heavy items, a wax tabletop rejects hot ones). ⭐ The `reason` is what
the verb reads into its refusal, so name it for a player.

Item-side gates intentionally don't exist — the authoring intuition is
host-side (*"this host refuses X"*), not item-side.

### `Placing.userFacingDetail`

MQL keyword bridge, mirroring `SlotSpec.userFacingDetail`. An author
declares `userFacingDetail: tabletop` and `put apple on tabletop`
resolves the keyword to the host via the Detailed-keyword path. Pure
MQL plumbing; placement hosts don't gain Slotted semantics.

⚠ Nothing reads it today beyond that bridge — `lib/ground/Floor.ts`
authors it and no consumer exists. Recorded on the narrowing slate.

### Rung 1: the containment read reaches the wire (cold-storage build)

The model has always known how each item sits and whether it is itself
a holder; the client was sent only `{ displayName, quantity,
primaryKeyword }` per contained item and threw the rest away, so a
fridge-with-a-freezer read as two anonymous boxes. Rung 1 stops the
discard. **Rung 2 — the recursive nesting view — is the next build; this
is the honest bytes plus a flat per-item read, no expand tree.**

Three subscribable fields carry it (the projection seam is
`MqlSubscriptionApi.projectFields`, § [card-surface.md](./card-surface.md)):

| field | mixin | on | read |
|---|---|---|---|
| `placement` | `Containable` | `REF_FIELDS` | the member name (`on`/`in`/`from`) this item sits under, or omitted when loose |
| `holds` | `Container` **and** `Placing` | `REF_FIELDS` | `true` — this thing is itself a holder. A capability, not a count (`static`, never fires), true for an empty container; a thing composing both reads one `holds` |
| `placed` | `Placing` | `DETAIL_FIELDS` | the items placed on this host, grouped by member with the member's own `Placement.getHeading()` — the card half of `look`'s drill-in prose |

`placement` is name-keyed on the field bus, so `ContainmentApi.place`
fires a `placement` `FieldChangedEvent` after `_setPlacement` (and the
`move` invariant fires it on the clear). `Container.contents`
`dependsOnFields: ['contents', 'placement']` so a placement change
inside an unchanged container still wakes the open container card, which
re-projects its children with the fresh placement. ⭐ `Chest` composes
`PlacingMixin` for a lid — so `look chest` reads *on the lid* (`placed`)
vs *in the box* (`contents`) rather than one flat list.

### Affordances live on Stuffs, not Details

`DetailedMixin` gives a Stuff addressable sub-parts for *description
only* — `look at door's handle` resolves and reads, but a Detail is
never itself an object a verb can target. **A sub-part that needs a
verb that DOES something — accepts things, holds things, can be picked
up — earns its own Stuff.** `userFacingDetail` (above) is the one
sanctioned bridge: it lets MQL resolve a keyword against the affording
host's real mixin (`Placing`, and `Slotted`'s own field of the same
name); the Detail never gains the capability itself. A bookshelf with
several genuine shelves is several `Placing` Stuffs, one per shelf —
not one Detail-with-its-own-Placing — keeping the substrate honest
about which sub-parts are actually interactive (graduated from the
affordance-verb slate, 2026-09).

### Presentation: `Container.getLooseContents(items?)`

Items appear in their enclosing container's contents naturally (the
apple is in the room). But a room *listing* shouldn't repeat an item
already represented by the host it sits on — the back-bar's bottles
read "on the back-bar", not as loose clutter beside the patrons.

`getLooseContents` drops any item whose placement host is itself in
the set. Three callers share it so the rule is uniform: `look`,
`sense`, and the inspection card (`Container.contents`).

⭐ **It is a METHOD, not an Api static, and that is the rule not an
accident**: it reads one container's own list and nothing else, and
the inspection card is a FIELD PROJECTION rather than a query, so the
rule has to be callable without MQL. See
[architecture.md § MQL is a VIEW over the model](../architecture.md) —
*anything expressible in MQL must also be expressible by function
call.* A `:loose` predicate is welcome as one line that calls this;
it may not replace it.

⚠ One clause of it is filed as a question rather than defended: the
`ids.has(host)` guard is set-relative and fires only when the caller
hands in a snapshot that already dropped a host while keeping its
contents — damage control for a lossy perception filter, not a
containment rule. →
mql-predicate-parity.

### The drill-in

Examining a placement host lists what is on it, **one line per member
it offers**, with the heading off the member's own row:

```
── On it: a shaker, a muddler and a strainer.
── Hanging from it: a prime cut of meat.
```

So a hook says *Hanging from it* where a shelf says *On it*, with
nothing in `look` or `sense` knowing the difference. A member with no
live row falls back to `With it`.

### The concrete hosts

**`Fitting`** (`platform/thing/Fitting.ts`,
`PlacingMixin(Good)`, `fixedInPlace = true`) is the
bare fixture things are placed on — a shelf, counter, table, rail,
hook, the bar's back-bar. ⭐ A row decides WHICH member it offers
(`placements: [from]` for a hook) and how airy it is, so a drying
rack, a meat hook, a cheese shelf and a wire line are all this class.

⚠ It was `Surface` until 2026-09-28; the rename is because the class
is no longer only about surfaces.

`Oven`, `Hearth` and `Campfire` compose `PlacingMixin` directly — a
range is a firebox you put a loaf IN and a plate you stand a pot ON,
and both limbs feed the same thermal couple.

### ⭐ `ContainmentApi.reachableFrom(actor)`

Everything `actor` can act on, **on-person first**: what they wear or
wield (`Slotted` occupants), then what they carry (`Container` contents),
then what shares their location, minus the actor. Deduped by `stuffId`;
held-before-floor ordering is part of the contract, so a first-match
consumer prefers your own gear over what happens to be lying about.

⚠⚠ **It applies NO perception**, deliberately. This is engine
bookkeeping, the same licence system mode takes. A *player* asking what
they can reach must be answered by the `reachable:[…]` MQL query, which
applies honest fog, recognition-relative naming and via-attribution.
Prefer the query whenever a viewer is involved.

⚠ **It is not `CommandController.reachableMarks(giver)`**, which is *what
you could aim something at* and therefore excludes the actor themselves.
The two differ by exactly one member, and the difference is the point.

⭐ **Why it exists:** the old `ContainmentApi.findReachable` was deleted
into MQL's `reachable` seed, which lives behind a sealed subdir only
`api/mql.ts` may import from. That left non-controller callers — a brain,
a logic singleton — with no sanctioned route, and **eleven** of them
wrote the two hops by hand. Deleting a finder is only half the job; the
other half is leaving the replacement findable from where the callers
are. See [antipatterns.md § Rebuilding the two-leg reach by
hand](../antipatterns.md).

### Verbs: `put`, `give`

⭐ **`put` resolves over N OFFERS**, not two branches. The controller
builds the ways this target can be put into or onto, in listing
order:

- **region zero** — the container's own interior — when the target is
  a plain Container and not a body. It is NOT a member (a chest has no
  compartments, it just holds things), but it borrows its words and
  its prose from the `in` row when the catalogue has one. That is what
  makes *one region and many regions are the same thing* literal.
- **one per member** the target's row offers (`placements:`).

A typed preposition matches a member's PRIMARY word first, then any
secondary — which is how `put ham on hook` resolves `from`. No match
names what the host does take; two matches ask which. With no word,
a single offer is taken and several ask.

Dispatch: region zero → `ContainmentApi.move`; a member →
`ContainmentApi.place(item, name, target)`. ⚠ A shut container is
refused **at the verb** with reason `shut`, never in `move` — brains
and restocks legitimately move goods into closed cupboards — and the
target stays BOUND, because a region you can name and be refused from
teaches, while one that vanishes from the parser reads as a bug.

The sentence is the MEMBER's: one Liquid template on its row,
rendered per audience, so `from` says *hang* with nothing in the
controller knowing the difference.

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

⭐ **`MobileMixin` also affords the `teleport` VERB**, beside `go` and
`goto` — the TPA reform moved it off `AuthorMixin`, where it had been a
wizard tool, on the grounds that *teleportation is a way of moving and
belongs to whatever moves*. The verb is the kernel's and the teleport
NETWORK is the `tpa` pack's; they meet over the `TravelNode` shape
(`lib/travel/TravelNode.ts`) and neither imports the other, so a
platform-only boot still teleports the people entitled to. Forks, in
order: free movement inside an extent you hold → a travel node's
timetable or ride → the anchored spell. See
[fasttravel.md](./fasttravel.md).

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
| `canMove(to)` | item | `ContainmentApi.move` (pre) — ⚠ class invariants only, see below |
| `canRemoveContainable(item)` | source container | `ContainmentApi.move` (pre) |
| `canAddContainable(item)` | dest container | `ContainmentApi.move` (pre) |
| `onContainableRemoved(item)` | source container | `ContainmentApi.move` (post) |
| `onContainableAdded(item)` | dest container | `ContainmentApi.move` (post) |
| `onMoved(from, to)` | item | `ContainmentApi.move` (post) |

The traversal layer fires from `Mobile.traverse` and is exit-aware.
The containment layer fires from `ContainmentApi.move` and runs
regardless of whether an Exit was involved (so `teleport` and
`StuffApi.clone`-then-place still trigger the containment hooks).

### ⚠⚠ `canMove` is a class invariant, never "a person can't take that"

It is the lowest-level veto in the system, and the tempting one. It fires
inside `ContainmentApi.move` — *the* chokepoint, which a remodel, a
`place`, a room being rebuilt and an author rearranging scenery all go
through. A veto there says the move is **impossible**. It also takes no
actor, so it cannot reason about who is doing the moving even if you
wanted it to.

What you almost always mean is narrower: **an agent cannot pick this up
or carry it off.** That is `fixedInPlace` — a persistent, authorable
state on `ContainableMixin`, read by the verbs that model *a person
taking a thing* (`get` refuses with `fixed-in-place`; `hitch` is already
gated by `HaulableMixin` composition). The wall TV is *mounted*, not
*immovable*, and the two are different facts.

Being a per-instance field is the other half of the argument: whether a
given screen is bolted to the wall or standing on a counter is the ROW's
business, and a class-level veto could never say. `Screen.canMove`
(refusing every destination but a `Location`) was the one production
override in the tree; it is gone, and `canMove` now has **no production
users** — the seam stays for a genuine class invariant, and
`witness.test.ts` holds its contract.

A structural invariant is the case that *does* belong at the chokepoint,
and there is one: an attached `Adornment` throws rather than sliding into
a container's `getContents()`, because it lives in the host's
`getFixtures()` tier and moving it would corrupt two-tier bookkeeping.
That is not a policy about who may act — it is the data model refusing to
be made inconsistent. **Structural invariants throw at the chokepoint;
"a person can't lift that" is a capability test on the verb.**

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
`Adornable` surface, composing `FloorMixin` over `Postured` (see
[posture.md](./posture.md)). ⭐⭐ **Since the ground build every Location
has one by construction**: `Location.ensureFloor()` mints
`TemplatePaths.defaultFloor` at `onCreate` unless the room authors a
floor of its own or opts out. The v1 note that used to sit here — *"no
class-level default; floor presence is authored per-Location"* — is
retired: 27 of 180 Locations had a floor, and per-Location authoring was
a choice nobody was making.

The room's one read for its floor is **`Adornable.getFloor()`**. Three
resolvers used to answer that question three times over by scanning
fixtures-then-contents for any Bulkable with a surface slot; they all call
`getFloor()` now and keep their own `hasSurfaceBulk()` check, because a dry
posture floor is a floor and a puddle needs the slot.

```yaml
# Nothing at all — the room gets the default floor, which resolves its own
# material down a five-rung ladder (see posture.md).

# A material and a construction, without authoring a floor row:
data:
  floor: { material: /stuff/idea/material/rock/granite, worked: true }

# Not standable at all — the void today; mid-air and mid-water when they
# exist. ⚠ A different question from `onGrade`, which asks whether the
# ground continues BENEATH a floor that exists.
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

---

## `airExposure` on `PlacingMixin` (extraction W1)

A persistent, authorable fraction (default `1`) with `getAirExposure()` /
`setAirExposure()` — **how much of a thing placed on this host the air
actually reaches.** Read by the two-way arm of the per-instance water state
([spoilage.md § The water state](./spoilage.md)), never by anything spatial.

Three rungs decide it: **enclosed is `0`** (a thing whose immediate container is
not a Location exchanges nothing — a sack, a chest, a pack, a pot), else the
host's own number (read off `getPlacement()`), else
`cure.groundExposure` (`0.35`) for bare ground.

⭐ Drying is **surface-limited**, and that one dial is the whole reason cheese
sits on slatted shelves and turf is built into a lattice: a ham on a stone floor
dries on top and goes off underneath — and a ham HUNG from a hook
(`airExposure: 1`, the shipped meat-hook row) is in the air on every
side, which is the case the number existed for before anything could
express it. ⚠ The enclosed rung is also what keeps the
store sparse — *a read that would change nothing writes nothing*, so the common
case (a ration in a pack) never starts a clock.

---

## History — `Surfaced` → `Placing` (the placement build, MR !302)

`f9107486f..f28eff91d`, 2026-09-28. Recorded because the rename is a
model change wearing a rename's clothes, and because two of the things
it turned up were older than the build.

**What moved.** `SurfacedMixin` → `PlacingMixin`; `_restingOn` → the
pair `(_placementHost, _placementName)`; `canRest(item): boolean` →
`canPlace(item, name): VetoResult`; `getResting()` → `getPlaced(name?)`;
`ContainmentApi.placeOn(item, host)` → `place(item, name, host)`;
`platform/thing/Surface` → `Fitting`; the persisted struct `Placement`
→ `ContentPlacement` (the name freed for the vocabulary Idea, and
distinct from combat's own `type Placement`).

⭐ **`ContainmentApi.looseContents` became `Container.getLooseContents()`.**
A free Api function reading one container's own list, twelve lines
below that file's own comment saying the read-wrappers were removed
*"because those reads live on the objects themselves."* It is a method
now, and [architecture.md § MQL is a VIEW over the model](../architecture.md)
is the rule that came out of arguing about where it belongs.

**Two shipped defects the build's drive surfaced**, neither reachable
by any test:

- ⚠⚠ **Nothing in the world melted from being warm.**
  `Thermal.reconcilePhase()` had three callers — a lit `Furnace`, two
  spell endpoints, and tests — so a block of ice on a hot floor sat at
  its melting point forever with the latent accumulator untouched. The
  phase engine was complete and had no ambient driver.
  `reconcileThermal` drives it now, narrowed to `Meltable` hosts.
- ⚠⚠ **The `coldStorage` archetype satisfier was wrong on both rungs,
  in opposite directions.** The space rung asked whether a room was
  `Thermal` — no `Location` is, so a 279 K cold store authored for
  exactly that satisfied nothing, ever. The holder rung asked only
  *insulated and closable* with no temperature test, so Dave's Bar
  reported cold storage MET on an empty ice bin. Both rungs are
  repaired and `Archetype.ts`'s own docstring now states the bar.

**And a gap that is not spatial's**, filed rather than fixed: a body
could not author its own temperature (a *space* always could), so a
row shipping a block of ice minted one at room temperature.
`ThermalMixin.stampedTemperatureK` is authorable now — see
[thermal.md](./thermal.md).

`Chamber` — a compartment with its own air, the first honest composer
of the `in` member — was deferred by the project owner rather than
built. Its full specification, including the `AtmosphericMixin`
widening it needs (the mixin's base constraint is `Stuff & Container`
and a compartment is not a Container), was graduated into
fridge-design-pack at this
build's sweep — written against the shipped tree and executable cold.
