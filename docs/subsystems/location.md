# Location subsystem

Source of truth for `packages/server/src/mud/lib/location/` — the
world's **geometry** (rooms, coordinates, concrete spatial zones) plus
the **Warren** elastic-graph substrate (the "multilocation" pattern) —
and the lounge content that rides it
(`packages/server/src/mud/domain/lounge/`). Read this before editing in
the area.

`lib/location/` carves the room/coordinate/zone geometry out of what was
`lib/spatial/`; `lib/spatial/` now holds only the containment/movement
substrate (Container, Containable, Mobile, Surfaced, Sealable — see
[spatial.md](./spatial.md)). The base Zone hierarchy (`Zone` /
`SpatialZone` / `FolderZone`) lives in `lib/zone/` (see
[zone.md](./zone.md)); only the concrete coordinate zones
(`CartesianZone`, `SphericalZone`) — whose identity *is* a coordinate
frame — sit here under `lib/location/`.

This doc has two parts: the **geometry** (Locations, Coordinates, Zones)
that everything spatial builds on, and the **MultiLocation / Warren**
substrate + the lounge that uses it.

---

## Locations

`lib/stuff/Location.ts` is the abstract base. Pure structural role:
"any Stuff that can hold other Stuff but doesn't itself live inside
something." Composition layers
`AddressableMixin(AmbientLitMixin(AtmosphericMixin(AdornableMixin(
ContainerMixin(Stuff)))))` — so every concrete Location picks up
biome-aware atmospheric state ([biome.md](./biome.md)), a rooted
address ([address.md](./address.md)), and an ambient light level
([light.md](./light.md)).

> **Corrected 2026-07-31.** This line previously read
> `AtmosphericMixin(TangibleMixin(AdornableMixin(ContainerMixin(Stuff))))`,
> which was wrong twice over: a Location is **space, not matter**, so it is
> deliberately NOT `Tangible` (no material, no mass — that is the seam
> `Thing`/`Vessel`/`Agent` ride, and the same seam that decides what can get
> wet), and `AddressableMixin` was missing. `AmbientLitMixin` is the
> houseplant build's addition — the mixin shipped long before, but **no
> location class composed it**, so `ambientIntensity` was authorable nowhere
> and every room read pitch-black. It is inert by default (the perception
> walk skips a zero-flux ambient), so an unauthored room reads exactly as it
> did before. See [husbandry.md](./husbandry.md) § Content.

Two derived-geometry methods (`getVolume(): Quantity<'m³'> |
null` and `getCeilingHeight(): Quantity<'m'> | null`) live on
`AtmosphericMixin` with null-returning defaults; concrete Location
subclasses override per their topology (and Vessel can too, when a
particular vessel has a meaningful interior volume).

Concrete rooms layer Visible, Exitable, and a coordinate mixin on top:

- **`CartesianLocation`** = `Exitable(CartesianCoords(Visible(Location)))`.
  Overrides `addExit` (async — awaits zone resolution) per the
  **cardinal-only-intra-zone exit invariant**: cardinal directions
  (`north`/`south`/`east`/`west`/diagonals/`up`/`down`) are accepted
  unconditionally; non-cardinal labels (`portal`, `office`) are
  accepted only when the destination's templatePath resolves to a
  *different* zone than the source's. The check is eager and
  path-based — `ZoneApi.resolveZoneForPath` walks template ancestry
  to compute each side's zone without loading the destination room
  as a Stuff. Intra-zone non-cardinal exits throw with a diagnostic
  naming both seed paths and the shared zone. Per the
  cardinal-only-intra-zone exit invariant (see [zone.md](./zone.md)).
- **`SphericalLocation`** = `Exitable(SphericalCoords(Visible(Location)))`.
  No restrictions on `addExit` direction labels — spherical zones
  have no implicit adjacency, so semantic labels are the only way to
  author exits.

Both compose `VisibleMixin` so a `look` on the room returns its
description; `NamedMixin` is opt-in for rooms that take proper names
("Town Square").

Both override `getZone()` with a narrowed return type:
`CartesianLocation.getZone(): CartesianZone | null` and
`SphericalLocation.getZone(): SphericalZone | null`. The cast-by-
invariant lives at the boundary method (each `Zone.addLocation`
rejects mismatched coordinate types), so call sites within the
concrete location see the narrowed type for free —
`CartesianLocation.getSizeScale()` reads `getZone()?.getCellSize()`
without any local cast. See
[antipatterns.md § "instanceof, virtual methods, and cast-by-invariant"](../antipatterns.md)
for the underlying pattern.

## Coordinates

Two coordinate mixins, both pure data carriers. Grid registration and
neighbor lookups live on the `Zone`, not on the coordinate mixin.

**`CartesianCoordinatesMixin`** (`CartesianCoordinates.ts:24-50`):
`coordinates: [x, y, z]` plus `getX/Y/Z` and `setX/Y/Z` accessors.
Persistent (auto via `persistentFields = ['coordinates']`).

**`SphericalCoordinatesMixin`** (`SphericalCoordinates.ts:28-64`):
`coordinates: [rho, theta, phi]` (focus) plus `radius`. Persistent.
Default radius is `1.0`.

The setters round-trip through tuple-replacement (`setX(x)` returns a
new triple) so any reactive system watching `coordinates` sees a
single coherent change rather than three partial states.

### How big a room is — `getLinearExtent()`

**One number, and everything that cares about room size derives from
it.** `Location.getLinearExtent(): number | null` returns the room's
edge length in metres — `CartesianLocation` reads its own persistent
`extent` field and falls back to the zone's cell size, so an authored
room can be larger or smaller than its zone's default;
`SphericalLocation` returns `2 × radius`; the base returns null (size
unknown).

`getSizeScale` (light falloff), `getVolume` (atmosphere) and
`getCeilingHeight` all derive **through** it. That is deliberate: a
combat-only size field would let one room be small for weapon range and
large for illumination. A bigger room is instead dimmer, airier **and**
longer-ranged, because that is what a bigger room is.

Its load-bearing consumer is the range-band ladder — the reachable
bands in a fight are capped by the room's real extent, so a 3 m cell
affords only melee and an authored 20 m one reaches `far`. Nobody
authors a band; the geometry decides. See
[ranged.md](./ranged.md).

## Zones

The base `Zone` hierarchy (`Zone` / `SpatialZone` / `FolderZone`) and
`ZoneApi` are documented in [zone.md](./zone.md); the concrete
coordinate zones below live in `lib/location/`. Every zone owns:

- `name: string` — human-readable ("Narnia Castle").
- `rooms: Set<Location>` — membership. Maintained by `addRoom` /
  `removeRoom`, with a back-reference stamped onto `room.zone`.

Exits are **not** derived by the zone — every room authors its own exits
explicitly (both sides). The spatial zone is the coordinate grid + its
invariants, not an exit source.

### `CartesianZone`

`CartesianZone.ts`. Same-size grid cells; **explicit** exits (no
grid-derived exits — a room connects to exactly what its template
declares).

- `addRoom(room, x, y, z)` — stamps `room.coordinates` from
  the supplied indices, indexes the room by `"x,y,z"` key. The room must
  compose `CartesianCoordinatesMixin` or `addRoom` throws.
- `getNeighbor(from, direction)` — looks up the room at
  `from + offset(direction)` (`NavigationApi.directionOffset`). A
  geometry query for tooling, **not** an exit source.

`cellSize` is **load-bearing** as of the biome substrate
([biome.md](./biome.md)). Default `3.0` linear meters (a typical
room). It drives:

- `CartesianLocation.getVolume()` → `cellSize³` (27 m³ at default).
- `CartesianLocation.getCeilingHeight()` → `cellSize` (3 m default).
- `CartesianLocation.getSizeScale()` → `cellSize²` (9 m² —
  receiving-surface area for the light substrate, derived by
  squaring the linear cellSize rather than authored separately).

A 5 m × 5 m room is `cellSize: 5`. Tests pinning specific LightBand
values may need to set `cellSize: 1` for the prior 1 m² calibration.

### `HomeZone`

`HomeZone.ts` (`lib/home/HomeZone.ts`). A bare `Zone` subclass with
no body (v1) — exists to anchor the per-player namespace at
`/home/`.

The `/home/` Zone template is created at boot from `seeds/home.yaml`.
The folder/leaf invariant requires a Zone-shaped template at every
internal node of the content tree, so anything stored under
`/home/<playerId>/...` needs `/home/` itself to be a real Zone
template — `HomeZone` is the class behind it.

Per-player sub-folders (`/home/<playerId>/`) are NOT seeded upfront.
They're created lazily the first time a feature needs them — e.g.
`EvalController` stamps its singleton at `/home/<playerId>/_eval`
without first materialising a `/home/<playerId>` Zone (paths in
`templatePath`-stamped Stuff don't go through the persistence
chokepoint).

The empty body is deliberate: future home-tier behaviour
(per-player permissions, quotas, customisation hooks) layers onto
this class without churning callers.

### `SphericalZone`

`SphericalZone.ts`. Spheres positioned by focus, **no implicit
adjacency**.

- `addRoom(room)` inherits the base placement and indexes
  by rounded focus tuple in a debug `focusIndex`. The index is for
  authoring tooling only — exit lookup never consults it.
- Exits are authored by hand as semantic labels (`'office'`, `'plaza'`,
  `'portal'`) — the same explicit-exit rule as everywhere; spherical
  space simply has no cardinal grid to have tempted derivation.

### Zone resolution: `ZoneApi`

`api/zone.ts`. Single entry point for `templatePath → Zone`. Caching
is delegated to `StuffApi.singleton()` (Zones compose
`SingletonMixin`); ZoneApi just owns the ancestor walk.

**`ZoneApi.isFolderClass(classPath)`** answers "does this class count
as a zone folder?" — structural: `prototype instanceof Zone`. Any
Zone subclass — spatial (`CartesianZone`, `SphericalZone`) or
non-spatial (`Clade`, future taxonomic / permission scopes) — passes.
`TemplateApi` consults this for the folder/leaf invariant — see
[templates.md § TemplateApi & the Folder/Leaf Invariant](./templates.md#templateapi--the-folderleaf-invariant).

**`ZoneApi.isSpatialZoneClass(classPath)`** is the strict subset:
`prototype instanceof SpatialZone`. Only spatial Zones stamp
`Stuff.zone`; non-spatial Zones (Clade) are folders but never become
a `Stuff.zone` reference. Adding a new Zone subclass means
`extends Zone` (or `extends SpatialZone` if it's a topographical
flavor) — no central allow-list to edit. Both checks dynamic-import
the class once and cache the result.

`ZoneApi.resolveZoneForPath(templatePath)` walks ancestor paths
nearest-first, consulting `isSpatialZoneClass` to skip non-spatial
Zone ancestors (Clades) during the walk. Returns the singleton
SpatialZone at the first spatial-zone ancestor via
`StuffApi.singleton(ancestor)`. The second resolution for the same
zone path is an O(1) cache hit; first resolution clones. Returns
`null` when:

- The template at `templatePath` is itself a spatial Zone (a zone
  isn't inside itself).
- No ancestor resolves to a spatial Zone template.

The clone pipeline calls `resolveZoneForPath` once at clone time and
stamps the result onto `Stuff.zone` before hydrate, so anything
reading `this.zone` during `postRegister` sees the right value
(see [templates.md](./templates.md#clone-pipeline)).

#### Field inheritance via `Zone.lookupField`

For zone-carried defaults that should inherit through the template
tree, `zone.lookupField<T>(fieldName)` walks ancestry nearest-first
and returns the first non-null value defined on any ancestor Zone.
**Unlike `resolveZoneForPath`, the inheritance walk treats every Zone
subclass as an inheritance node** — FolderZone, HomeZone, Clade, and
spatial zones alike. A `celestialProfile` field set on a universe-root
FolderZone is inherited by every spatial zone beneath it. Returns
`null` when nothing in the ancestry defines the field; callers compose
a settings-style default on top:

```ts
const profile =
  (await zone.lookupField<CelestialProfile>('celestialProfile'))
  ?? resolveSetting(host, 'world.zone.celestialProfile.default');
```

The walk lives **on the Zone class** (not on `ZoneApi`) so subclasses
can override its behavior. `lookupField` orchestrates; the
override seam is `lookupAncestorField` — default delegates to the
enclosing zone's `lookupField`, but a barrier subclass overrides it
to return `null` and root inheritance at itself. Full surface and
the barrier-subclass pattern documented in
[zone.md § Field inheritance](./zone.md#field-inheritance-zonelookupfield).

### The setter-with-side-effects pattern

`CartesianLocation.setCoords({x, y, z})`, `SphericalLocation.setFocus({rho, theta, phi, radius})`,
and `Window.setAttachedHosts([pathA, pathB])` share a single
declarative-content shape: a public setter that **stores the value
AND performs cross-object side effects** atomically. The shape is:

1. Validate the input (TypeError on shape mismatch).
2. Resolve the dependency context (`getZone()` for coords/focus; both
   hosts via `StuffApi.singleton` for attachedHosts).
3. Check idempotency (zone already has this room at these coords?
   anchors already wired to these hosts?). If so, accept and no-op
   the side effect — only the storage assignment runs.
4. Check conflict (already at different coords / attached to
   different hosts?). If so, throw with a diagnostic naming both
   states.
5. Apply the side effect (`zone.addLocation(this, x, y, z)`,
   `BoundaryApi.attachExistingBoundary(...)`).
6. Store the value.

Idempotency makes re-hydrate / HMR-after-destruct / cycle-loop-self-call
safe. Conflict-throw catches drift between the YAML and the runtime
state. The setters are `async` because steps 2 and 5 may call
`await StuffApi.singleton(...)` to lazy-clone dependencies.

`coords` and `coordinates` are distinct fields by design: `coords`
is the YAML-shape declarative input `{x, y, z}` and the setter's input
shape; `coordinates` (on `CartesianCoordinatesMixin`) is the runtime
tuple `[x, y, z]` consumed by the zone grid. The setter bridges them:
`setCoords` calls `zone.addLocation(this, x, y, z)`, which internally
calls `setCoordinates([x, y, z])`. Unifying them is out of scope here.

---

## MultiLocation — the Warren elastic-graph substrate + the lounge

A **MultiLocation** is one room *template* with many *live instances*
that coordinate as a group: the graph **buds** new instances when it
fills and **merges** them back when it empties. The owner of each graph
is a **`Warren`** — an incorporeal `Idea` (identity + state, no physical
presence). v1 ships the substrate plus a rudimentary **lounge** (the
universal login landing) over it.

Promoted from `docs/slates/builds/multilocation-slate.md` +
`docs/slates/builds/lounge-slate.md` via
`docs/requirements/multilocation-lounge-requirements.md` and
`docs/plans/multilocation-lounge-plan.md`.

## The pieces

**Warren substrate — `lib/location/` (generic, reusable; alongside the
geometry classes documented above):**

| File | Role |
|---|---|
| `Warren.ts` | Abstract base — the generic mechanism. |
| `WarrenMember.ts` | `WarrenMemberMixin` — optional member-side back-ref. |

**Content — `domain/lounge/` (the lounge area; class paths
`/domain/lounge/*`):**

| File | Role |
|---|---|
| `Lounge.ts` | The one room template every lounge instance clones from. |
| `Bar.ts` | Singleton external-neighbor shell (Dave's Bar); self-stocks crafting content via `populates:` (the back-bar fixture + bottles/tools `onto` it, plus Dave + the menu). |
| `LoungeWarren.ts` | Concrete singleton Warren — the lounge *policy*. (The **second** Warren subclass, `DormWarren`, is an elastic *two-tier* consumer — keyed persistent members + runtime floor scaffold; see [residence.md](./residence.md).) |
| `LoungeMixin.ts` | `LoungeMixin` — lounge-room behavior + the home for future room functionality. |
| `Menu.ts` / `CraftedDrink.ts` / `GradedReceptacle.ts` | Bar crafting content (the offer object, the crafted-drink output glass, the stock bottle) — composed over the general `lib/craft/` substrate. See [crafting.md](./crafting.md). |

Content classes live under `/domain/lounge/` — a managed area's own
class namespace, mirroring its template namespace (the class-path
validator admits `/domain/` alongside `/obj/` and `/lib/`). The generic
Warren substrate stays in `/lib/`.

Content seeds: `seeds/domain/lounge/{warren,lounge,bar}.yaml` (templates
at `/domain/lounge/{warren,lounge,bar}`, leaves under the
`/domain/lounge` FolderZone), plus the bar's crafting fixtures
(`back-bar`, the four bottles, `shaker`/`mixing-glass`,
`cocktail-glass`, `bar-menu`, `dave`) the `Bar` self-stocks via
`populates:`. See [crafting.md](./crafting.md).

## Core model

- **Coordinator, not a containment tier.** Member `Location`s stay
  ordinary roots (`getContainer()` → null); the Warren tracks membership
  in its own Pattern-B set and consults it for routing/landing. It is
  NOT a container and adds NO tier above `Location`. ("Warren as a
  containment tier" is an explicitly deferred, game-wide evolution.)
- **Host is a runtime role.** Every live lounge room — the current
  commons *and* every satellite — is a clone of the one `Lounge`
  template. The Warren designates exactly one live instance as **host**
  (`getHost()` / `isCurrentHost`), migrating the role on forced host
  destruction. No `Commons` class, no host flag.
- **Lazy + runtime-only.** `LoungeWarren` composes `SingletonMixin`;
  `StuffApi.singleton('/domain/lounge/warren')` creates the one instance
  on first landing. Every room instance is a runtime clone, gone on
  restart, recreated on the next first-landing. Only the templates +
  the Warren *definition* persist.

## Base mechanism vs lounge policy

`Warren` (base) owns: the member set (R2.3 prune, R2.4 cleanup), host
designation + **migration**, `spawnMember`/`reapMember`, hub exit wiring
(incl. the `Exitable.onDestruct` asymmetry teardown — remove **and**
destruct the host-side exit, don't leave it blocked), `getHost()` (the
reusable placement kernel), `teardown`, ref discipline, and the
concurrency guards.

`LoungeWarren` (policy) owns: least-full `admitArrival`, star
`attachmentFor` (the satellite direction pool excludes `north` — that's
Dave's — and excludes vertical; the lounge expands horizontally), the
tunable bud/merge band (`getBudThreshold` / `getMergeWatermark` /
`getReapGraceMs`, `setThresholds`), the population `reconcile` loop,
`createMember` (clone the room), and `wireHostFixtures` /
`unwireHostFixtures` (the one Dave's Bar fixture, north of the host).
There is **no** campus/placeholder exit — the way out of the lounge is
fast-travel (TPA), built later.

The strategy is the **simplest tunable** one (least-full seat, reap-empty,
bud-when-all-full; never rebalance a live crowd) — an unproven UX bet,
built to fail safe (flatten-to-one-room is `N` high + `M` 0) and tune
cheap (defaults are code constants headed for app settings (`AppApi`)). The
base never imports the lounge.

## Landing: the `startLocation` spawn instruction

`container` only ever names an actual container. A Warren isn't one, so
landing uses a distinct **`startLocation` instruction field** on
`Avatar` (only avatars have a spawn/recall location — no mixin). The
Hydrator's Phase 2 auto-dispatches it to `applyStartLocation(ref)`, which
owns the *domain* semantics:

- The ref's class is resolved (`StuffApi.classForRef`). If it
  **`instanceof Warren`** — a real, unspoofable check against the
  canonical base class imported directly (no marker static) — land in
  `singleton(ref).getHost()` (the Warren is never the avatar's
  `container`; `container` stays honest).
- Otherwise it's an ordinary room: `StuffApi.singletonOrClone(ref)`, the
  generic primitive that decides **`singleton()` vs `clone()`** purely by
  whether the class composes `SingletonMixin`. No Warren/container
  special-casing lives in `StuffApi`.

`applyContainer` is unchanged (plain `singleton(path)`);
`validateSingletonContainerTarget` keeps its hard deny. The startLocation
path is the only place the Warren semantics live.

The avatar seed (`seeds/obj/Avatar/seed.yaml`) declares
`startLocation: /domain/lounge/warren`. **`DEFAULT_STARTING_LOCATION_PATH`
is unchanged** — it is the container-typed *evacuation* fallback in
`Container.cleanupOnDestruct` (a Warren is not a Container); spawn and
evac are separate concerns.

## Self-registration + ownership (Q2b)

`Lounge` declares its Warren via the `warren` instruction field;
`LoungeMixin.applyWarren` self-registers on hydrate (creating the Warren
singleton if absent — the stray-clone heal; the Warren check there is the
same unspoofable `instanceof Warren`). **The Warren owns the
relationship**: `addMember`/`removeMember` are the sole writers of the
Pattern-B pair. The declared path can *initiate* a membership but never
*re-home* an owned one — the single-warren guard rejects a member already
owned by another Warren (owner wins + warn). Reads always use the runtime
back-ref, never the declared path.

## Recall (placement capture)

The self-persistence spine's `PersistableLogic.capturePlacement` consults an
avatar's live container: when it is a `WarrenMember` with a Warren, it
captures `place.startLocation: <Warren>` on the avatar's record (rather than
the transient room clone), else a plain `place.container` — so logout in the
lounge resumes via `getHost()` into a *live* instance, never a dead clone.
On materialize the placement is restored through
`ContainmentApi.resolveLanding` / `move`. The hook rides the existing
`WarrenMember.getWarren()` back-ref; no extra capability mixin. (This is the
home of the recall logic the retired `TemplateApi.snapshotToTemplate` once
wrote to the avatar template — see
[persistence.md](./persistence.md#the-self-persistence-spine-persistable).)

## Instance identity — live-ref hub exits

Many lounge instances are clones of one template, so they **share** a
template path. That is ambiguous for any ref that resolves by
templatePath — most importantly an `Exit` destination
(`findByTemplatePath` throws on multi-instance). The fix is honest
Instance refs: hub and fixture exits between Warren rooms hold a **live ref**
(`keepLiveDestination` on `Exit` / `addBidirectionalExit`,
`Warren.wireHubExit`) rather than a path. No synthetic per-instance paths
— the instances keep their shared template path, and the exits point at
the specific live room directly.

## Concurrency

- `StuffApi.singleton` coalesces concurrent first-resolution of one path
  onto a shared clone promise (`#pendingSingletons`) — without it the
  second concurrent caller trips `clone()`'s cycle guard.
- `Warren.getHost` coalesces concurrent landings (`_hostInFlight`) → one
  host.
- `Warren.createMemberSerialized` serializes member clones per Warren so
  two concurrent buds never clone the same template path at once.

## The admit seam (single sense)

The arrival to the host completes normally; the host's
`onContainableAdded` (LoungeMixin) hands an over-capacity newcomer to
`admitArrival`, whose synchronous prefix re-seats to an eligible
satellite **within the triggering move** (one perception, in the final
room). Only when a bud is required does it `await`. No `#moveCore`
change, no veto, no redirect flag.

## Deferred

Lounge-room *function* (the `LoungeMixin` is the home for it — toppings,
matchmaking, the order console, …), cues beyond the bud/merge doorway
lines, Dave's contents (NPC, drinks, `sit`, menu), fast-travel/TPA (the
way out of the lounge — there is no placeholder campus exit),
item-spawn-into-a-Warren (reuses `getHost()` as a separate instruction),
the procedural-spatial Warren family (dungeon/desert), and the
Warren-as-containment-tier evolution.
