# Zone subsystem

The `lib/zone/` subsystem holds the **Zone-hierarchy roots** — the
abstract scope abstraction (`Zone`), the abstract spatial-coordinate
base (`SpatialZone`), and the generic organizational `FolderZone`.
Concrete spatial-coordinate zones (`CartesianZone`, `SphericalZone`)
live in `lib/spatial/`; non-spatial taxonomy zones (`Clade`,
`HomeZone`) live in their own domain folders. All of them extend
`Zone`.

`Biome` deliberately does NOT extend `Zone` ([biome.md](./biome.md))
— biomes are leaf Ideas with explicit `_extendsBiomePath` parent
refs. The folder structure under `/obj/biome/` uses `FolderZone`
templates for the biome team's admin/ownership scoping; the actual
biome inheritance lives independently in the leaf data.

## Why a separate subsystem

`Zone` is not a spatial concept — it's the common parent of every
folder-shaped scope in the template tree, spatial and non-spatial.
`SpatialZone` is the abstract base for coordinate-bearing zones, but
its identity isn't a coordinate frame itself. Carving them out of
`lib/spatial/` makes the semantic boundary honest: zone-hierarchy
concerns live in `lib/zone/`; only the spatial-coordinate-bearing
subclasses stay under `lib/spatial/`.

## The class hierarchy

```
                    Zone        (lib/zone/Zone.ts)
                  /  |  \
                 /   |   \
                /    |    \
               /     |     \
   SpatialZone    FolderZone   non-spatial Zone subclasses
   (lib/zone/    (lib/zone/    (in their own folders):
   SpatialZone)  FolderZone)     - HomeZone   (lib/home/)
   /     \                       - Clade      (lib/species/)
  /       \
CartesianZone  SphericalZone
(lib/spatial/) (lib/spatial/)
```

- **`Zone`** — bare scope abstraction. Carries a name and the
  folder-of-templates contract (`ZoneApi.isFolderClass` checks
  `prototype instanceof Zone`). Subclasses participate
  automatically without editing a central allow-list.
- **`SpatialZone`** — abstract intermediate carrying the
  location-aware surface (`addLocation`, `getLocations`,
  `removeLocation`, `contains`, `canDestruct`). Stamps `Stuff.zone`.
  `CartesianZone` and `SphericalZone` extend this — not `Zone`
  directly. Exits are authored explicitly on rooms; the zone is the
  coordinate grid + its invariants, never an exit source.
- **`FolderZone`** — generic organizational scope, no spatial
  topology. Use for templatePath folders like `/domain/narnia/`
  or `/obj/biome/outdoor/` that organize a content team's tree
  without anchoring a coordinate grid. Sub-folders that DO need
  a coordinate frame extend `CartesianZone` / `SphericalZone`
  instead.
- **`HomeZone`** / **`Clade`** — domain-specific non-spatial
  Zone subclasses. They extend `Zone` directly, satisfy the
  folder/leaf invariant, but never stamp `Stuff.zone`.

## ZoneApi

`api/zone.ts` owns two concerns:

- **`resolveZoneForPath(templatePath)`** — walks template ancestry to
  find the nearest *spatial* zone; clones lazily via `StuffApi.singleton`.
  Non-spatial Zone ancestors (Clades, FolderZones) are **skipped** —
  they're folders for the template-tree invariant, but never the
  spatial zone for a descendant. Returns `null` when nothing in the
  ancestry is a SpatialZone, or when the input path is itself a
  SpatialZone (a zone isn't inside itself).
- **`isFolderClass(classPath)`** / **`isSpatialZoneClass(classPath)`** —
  structural class-shape predicates. `isFolderClass` is the broad
  check (extends `Zone`); `isSpatialZoneClass` is the narrow one
  (extends `SpatialZone`). The former gates the folder/leaf
  template invariant; the latter gates `Stuff.zone` stamping.

## Field inheritance: `Zone.lookupField`

Field inheritance is an **instance method on Zone**, not a ZoneApi
static — because subclasses can override the walk to participate
differently. Two instance methods make up the polymorphic surface,
backed by one Api orchestration helper:

- **`Zone.lookupField<T>(fieldName): Promise<T | null>`** —
  top-level entry. Reads `fieldName` on this zone first; if absent,
  delegates to `lookupAncestorField`. Returns the nearest
  non-null/non-undefined value, or `null` at universe-root.
  Field-read uses `get<PascalCase>()` first (the inter-Stuff
  contract surface), then direct property access.
- **`Zone.lookupAncestorField<T>(fieldName): Promise<T | null>`** —
  the override seam. Default: ask `ZoneApi.getEnclosingZone(this)`
  for the nearest enclosing Zone, then delegate to *its*
  `lookupField`. The recursion carries the walk upward. Subclasses
  override here to root or redirect inheritance.
- **`ZoneApi.getEnclosingZone(zone): Promise<Zone | null>`** —
  orchestration helper. Walks `Template.ancestorPaths`, skips
  non-folder segments, lazy-clones the hit via
  `StuffApi.singleton`. Lives on the Api layer (not on Zone)
  because the walk is pure plumbing — see
  [architecture.md § Orchestration lives one layer up from raw
  steps](../architecture.md#orchestration-lives-one-layer-up-from-raw-steps).

**Ownership does NOT live on `Zone`.** The `ownerGroup` / `accessGroups` /
`ownerGroupName` fields were **removed** in property phase 0a —
`Zone.persistentFields = []` and the zone carries no access controls of its
own. Title is a **parcel** in the gated `parcels` registry, resolved by
longest-prefix; `AccessApi.can` consults `ParcelApi.ownerOf`. This is the
governing security invariant (access-check data must be unspoofable by
content edits, so it can't live in the editable `domain` zone template). See
[parcel.md](./parcel.md) and [access.md](./access.md). The inheritance walk
below is untouched — it carries celestial/biome defaults, not ownership.

**Every Zone subclass participates as an inheritance node** —
FolderZones, HomeZones, Clades, and spatial zones alike — because
zone-carried defaults flow through folder ancestry too (e.g., a
`celestialProfile` set on the universe-root FolderZone is inherited
by every spatial zone beneath). Callers layer a settings-style
default on top:

```ts
const profile =
  (await zone.lookupField<CelestialProfile>('celestialProfile'))
  ?? resolveSetting(host, 'world.zone.celestialProfile.default');
```

### Barrier subclasses

A subclass that overrides `lookupAncestorField` to return `null`
becomes an **inheritance barrier**: its own defaults are authoritative
for descendants, and ancestor values do not flow through:

```ts
class RootedZone extends Zone {
  // The zone roots its own defaults — nothing from above leaks in.
  override async lookupAncestorField<T>(_field: string): Promise<T | null> {
    return null;
  }
}
```

No barrier subclass ships in this build — the seam is there for the
moment a real consumer needs it. A subclass could also override
`lookupAncestorField` to consult a non-template-parent source
(e.g., a sibling-template inheritance, or a per-field fallback
chain).

## Zone derivation rule

When the clone pipeline stamps `Stuff.zone` for a freshly-cloned
template, it walks ancestor paths nearest-first and picks the first
ancestor whose `class` resolves to a `SpatialZone` subclass.
**FolderZones (and other non-spatial Zone subclasses) are skipped
during this walk** — they're inheritance nodes for `Zone.lookupField`
but never become `Stuff.zone`. This decouples "this Stuff lives in
this scope" (the spatial nearest-ancestor) from "this Stuff inherits
fields through this scope" (every Zone ancestor).

## Cardinal-only-intra-zone exit invariant

A CartesianLocation accepts cardinal exits unconditionally and
semantic-label exits only when the destination's templatePath
resolves to a *different* zone than the source's. The check is eager
and path-based — `ZoneApi.resolveZoneForPath` walks template ancestry
without loading the destination room. See
[`spatial.md`](./spatial.md) for the full Location-side detail.

> **⚠ Content that needs a semantic exit needs a zone, and two things
> about that zone are easy to get wrong.** Both were hit live by the
> Hinkley lot gate (`lot-1` off a cartesian street — see
> [smallholding.md](./smallholding.md)):
>
> - **It must be SPATIAL.** A `FolderZone` is skipped by the resolver, so
>   the destination lands back in the enclosing spatial zone and the exit
>   is refused anyway — with an error naming the *parent's* zone, which
>   reads as if the branch you authored does not exist.
> - **It must be an AUTHORED template row.** The walk reads template
>   ancestry in Mongo, so a zone *minted* at runtime through
>   `asTemplatePath` is invisible to it. One authored branch zone above N
>   minted children is the shape; a zone per minted child is not
>   available at all.

## Folder/leaf invariant

The persistence chokepoint validates: a node with descendants must
be a folder (Zone subclass); a leaf node must be a non-Zone class.
Adding a new Zone subclass to satisfy a folder need is the right move
when no existing class fits. `FolderZone` is the generic answer when
the folder doesn't carry domain-specific behavior.

## Authoring guidelines

The cardinal-only-intra-zone invariant and the zone-derivation rule
together do most of the structural work for content authors. The
cardinal rule forces a zone break at every meaningful boundary
(building entries, portals); the derivation rule means folder
organization carries no zone implications as long as folders stay
non-spatial. What's left to author judgment is **how to organize
within and across zones**.

### Default pattern for `/domain/<team>/`

A team owns its subtree under `/domain/<team>/` and organizes freely
inside it. The recommended shape is geographic at the top, with
`FolderZone` for organization and `CartesianZone` (or `SphericalZone`)
where a coordinate grid is needed:

```
/domain/narnia/                  FolderZone (organizational)
  forest/                        CartesianZone — the woods, one grid
    western-clearing               Location (leaf, at coords)
    wolf-pack                       NPC (leaf, in the woods zone)
    silver-knife                    Item (leaf, in the woods zone)
  castle/                        FolderZone (castle has substructure)
    exterior                       CartesianZone (small grid)
    great-hall                     CartesianZone (interior grid)
    dungeon                        CartesianZone (interior grid)
  shared/                        FolderZone — reused across regions
    items/excalibur                Item leaf
    npcs/tumnus                     NPC who wanders (leaf)
```

The spatial zones connect via cross-zone exits at their boundaries
(e.g. `castle/exterior` → `castle/great-hall` through an `enter` exit
on the front door).

### Principles

1. **Top-level under a team's domain is geographic.** Reflects how
   authors think ("I'm working on the forest") and inherits
   geographic-flavor defaults cleanly (a forest biome set at
   `/domain/narnia/forest/`).
2. **Within a region, categorical or flat is the author's call.**
   Either `.../forest/western-clearing` flat, or
   `.../forest/clearings/western-clearing` sub-categorized. Depth is a
   taste choice.
3. **Shared content lives in a sibling `shared/` (or `items/`,
   `npcs/`) folder.** Things reused across regions go there; don't
   force them into one region's tree just because they appear there
   first.
4. **Internal nodes are Zones; leaves are content.** This is the
   folder/leaf invariant. `SpatialZone` subclass if the node anchors a
   coordinate grid; `FolderZone` if purely organizational.
5. **The cardinal rule decides "is this its own zone."** If you'd need
   a non-cardinal exit to reach a place from its neighbor, that place
   is its own zone. Don't fight this.
6. **Granularity heuristic (not a rule):** roughly 50–200 rooms per
   `CartesianZone` (CircleMUD area precedent). Tune at content build;
   don't ship a 5000-room zone.

### Anti-patterns to avoid

- **Everything categorical at the top** (`/domain/narnia/rooms/`,
  `.../items/`, `.../npcs/`). Breaks inheritance — geographic defaults
  can't reach rooms scattered across a categorical tree.
- **One zone for an entire team's domain.** Too coarse; the cardinal
  rule forces splits the moment there's any indoor/outdoor
  distinction.
- **Deeply nested per-sub-sub-region trees.** Authoring overhead
  doesn't pay off below ~2–3 levels of organizational depth.
- **Mixing geographic and categorical dirs at the same level**
  (`forest/`, `items/`, `dungeon/` all siblings). A sibling `shared/`
  is fine — it's clearly a meta-category — but mixing semantically
  same-level dirs is confusing.
- **Putting a `SpatialZone` where you wanted a `FolderZone`.** It
  promotes the whole subtree into one grid, potentially merging things
  you didn't mean to merge. When unsure, start with `FolderZone` and
  promote to a spatial zone only when you actually want a coordinate
  frame.

### `/idea/...` vs `/domain/...` are organized differently

These guidelines target `/domain/<team>/` content trees. Core
taxonomies under `/idea/` (`/idea/material/`, `/idea/species/`,
`/idea/biome/`) are organized **taxonomically** — root → kingdom →
species, or biome-category hierarchy — not geographically. Their
structure is determined by their own domain, not by these geographic
recommendations.

> **Backlog:** resource-boundary semantics — runtime caps (mob counts,
> item limits, reset cadence) scoped at the zone level — are
> aspirational today and deserve their own slate. Not covered here.

## History

The Zone subsystem was carved out of `lib/spatial/` as part of the
spatial+boundary substrate build that shipped declarative-content
field shapes (`coords`, `focus`, `exits`, `attachedHosts`). The
field-inheritance walk was drafted as `ZoneApi.resolveZoneField` in
the requirements/plan docs but moved to instance methods on `Zone`
during implementation review — the override-on-subclass extension
point (for barrier zones that root inheritance at themselves)
needed instance dispatch. The build's source slates (the
zone-architecture and declarative-content slates, since retired — their
authoring guidance folded into the "Authoring guidelines" section below
and into [templates.md](./templates.md)); the spawn-shape side
(`PopulatesMixin` + `container:` instruction field, with live-ref
consultation in `Avatar.enter` for the across-restart spawn case)
shipped in a follow-up build — see
[templates.md § Hydrator contract](./templates.md#the-hydrator-contract)
for the appliers and [connection.md § Phase 4](./connection.md#phase-4-avatar-materialization-and-handoff)
for the entry flow.

The Wave 1 + Wave 2 build landed on the `spacial` branch between
commits `b9afbaa` and `869c47a`. The spawn-shape follow-up landed
on `spawn-save`.

Property phase 0a (`feature/property-0a-title`, commit `81b250be`) **removed**
the three `Zone` ownership fields (`ownerGroup` / `accessGroups` /
`ownerGroupName`), their accessors, and `validateGroupRef`, setting
`persistentFields = []`. Ownership moved to the gated `parcels` registry (the
governing security invariant); the folder/leaf invariant and the field-
inheritance walk are untouched. See [parcel.md](./parcel.md).

The sandbox build (`feature/sandbox`) made two changes here. `Zone`
gained a `wire` classification field and, with it, a non-empty
`persistentFields = ['name', 'wire']` — the empty list from property
phase 0a meant a declared field with an accessor pair was never
hydrated, so a seeded `wire: true` sat in the row and never reached the
instance. `ZoneApi.resolveEnclosingZoneForPath` joined
`resolveZoneForPath` as its any-Zone sister: the spatial resolver skips
non-spatial zones, so asking it about `/home` or `/studio` answers
`null`, which reads as "not wire". See [sandbox.md](./sandbox.md).
