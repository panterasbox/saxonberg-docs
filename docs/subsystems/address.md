# Address substrate

A rooted, named, path-shaped **addressing namespace** — divorced from
both `templatePath` and the zone tree — plus the **Locality** tier that
claims a subtree, and the upward **longest-prefix resolve-walk** that
answers "which Locality covers this place?". This is the first build of
the delivery/addressing vision
(delivery-slate), carved to its
single load-bearing unit, and the substrate the
[weather build](./weather.md) named as its dependency (weather Wave 1 has
since shipped on it): weather is a field over the locality tree, and a
room finds its weather by walking up the address tree to the nearest
weather-bearing Locality.

Source: `lib/address/` (`Locality.ts`, `Addressable.ts`),
`obj/AddressRegistry.ts`, `obj/api/AddressLogic.ts`, `api/address.ts`.
The resolve chain mirrors `BiomeApi.resolve*For` beat-for-beat — read
[biome.md](./biome.md) alongside this.

## Three independent namespaces, separated by job

The address namespace meets `templatePath` and the zone tree at no
required correspondence — the divergence is day-one and deliberate (a
mailing address is not a filesystem path).

| Tree | Role | Mechanism |
|---|---|---|
| **Address namespace** | logical locality (delivery/coverage identity) | `Locality._address` / `Location._address` — Pattern-A path strings; the rooted `narnia/...` tree the Localities claim prefixes in |
| **templatePath** | engine identity (where the template lives) | the CMS template tree |
| **zone tree** | spatial integrity + admin taxonomy | `Zone` / `SpatialZone` |

A Locality at templatePath `/obj/Locality/cair-paravel` claiming address
`narnia/castle` is normal and expected; the demonstrative roster pins
this divergence.

## The Locality node

`Locality extends PostRegistrationMixin(Idea)` — reference data, a leaf
like `Biome` / `Material` / `Species`, hanging under `/obj/Locality/`.

- **`_address` is the claimed coverage prefix.** A Locality claims
  everything at or under its node; there is no separate coverage field
  (v1). One Locality per prefix — a duplicate claim is an authoring
  error (the first wins).
- **One concept, variable depth.** Whether a Locality plays the coarse
  "Region" role (a realm / climate source — the shortest prefixes) or a
  finer "settlement" role is authoring flavor — its `_address` depth and
  which tier-level fields it carries — **not** a subclass. There are no
  `Region` / `Block` / `Spot` classes (resolves delivery-slate Q6 for
  this unit: the **Locality** tier is reified; finer/coarser segments
  stay implied-by-string).
- **Lifecycle.** Like `Biome`, a Locality does **not** call
  `Stuff._registerTopLevelBranch` and does **not** compose
  `SingletonMixin`. Unlike `Biome`, it composes `PostRegistrationMixin`:
  `postRegister` self-registers its prefix into the `AddressRegistry`
  coverage index and `onDestruct` deregisters, so HMR re-clone never
  leaves a stale node. (`Biome` needs no such hook because `BiomeApi`
  re-resolves on every read with no index.)

## Declaring an address — `AddressableMixin`

Composed onto `Location` (`AddressableMixin(AtmosphericMixin(…))`). A
sparse, optional `_address: string | null` declaration (an identity ref,
modeled on `AtmosphericMixin._biomePath`); `null` is the common case
and costs one field. **Resolution is not the mixin's job** —
`getAddress()` returns the raw declared string; turning it into a
covering Locality is `AddressApi`'s walk, which reads `_address` off
ancestors directly during the containment walk (the same chain-walker
carve-out `_biomePath` documents). The mixin gives the resolve-walk a
`MixinApi.isAddressable` predicate, and leaves the seam open for a
future Vessel-carries-an-address case.

## The resolve chain (the four steps)

`AddressApi.resolveLocalityFor(scope)` mirrors the biome chain
step-for-step (simplified — one field, no detail keys):

1. **Containment-outward** — walk container ancestors (depth-capped at
   32, `stepOutward` copied from `BiomeLogic`) to the nearest
   `Addressable` ancestor declaring an `_address`. A vessel inside an
   addressed room inherits the room's address (`source:
   'inherited-containment'`).
2. **Spatial-zone fallthrough** — if no ancestor declares one, the
   outermost Location's `getZone()?.lookupField<string>('address')` (the
   one `await`). A whole castle zone can carry one address for all its
   rooms (`source: 'zone'`).
3. **Longest-prefix match** — match the resolved address against the
   reified Localities; the longest claimed prefix wins (more-specific
   covers).
4. **No declared/inherited address, or no covering Locality → `null`** —
   the global / off-grid result. **Never throws**; `null` is a normal
   value (contrast biome's mandatory universe terminal — addressing has
   no mandatory root).

### Worked traces

```
# Roster: narnia (narnia) · cair-paravel (narnia/castle) · lantern-waste (narnia/wild)

room.setAddress('narnia/castle')        → Cair Paravel   (source: declared)
vessel inside that room                  → Cair Paravel   (source: inherited-containment)
room, no address, zone['address']='narnia' → Narnia        (source: zone)
room.setAddress('narnia/castle/closet')  → Cair Paravel   (longest prefix; chain [castle, narnia])
room.setAddress('narnia/other/room')     → Narnia         (falls back to the root Region)
room.setAddress('elsewhere/void')        → null           (source: declared, no cover)
room, no address, no zone                → null           (source: none)
```

## The coverage index — `AddressRegistry` + `PathTrie.longestPrefix`

Because address ≠ templatePath, Localities cannot be found by walking
template paths; the longest-prefix match needs an index keyed by
claimed prefix. `AddressRegistry` (`/obj/AddressRegistry`, a
`PostRegistrationMixin` singleton parallel to `AccessRegistry`) holds a
`PathTrie<Locality>` of claimed-prefix → Locality. The index lives on
the Registry (not the stateless logic singleton) so it survives a
reload of `api/address.ts`; a reload of the Registry re-clones and
`postRegister` rebuilds idempotently.

- **`PathTrie.longestPrefix(path)` / `longestPrefixPath(path)`** — the
  additive nearest-ancestor reads (the deepest value-carrying prefix);
  segment-wise, no glob semantics.
- **Eager roster clone.** Leaf Ideas clone lazily, so a Locality's
  self-registration only fires once something clones it. The Registry's
  `postRegister` therefore eagerly clones every Locality template under
  `/obj/Locality/` so the index is complete even for never-accessed
  Localities. `PathTrie.insert` is idempotent, so the eager insert and a
  Locality's self-registration converge.
- **v1 simplification (flagged for the delivery build):** the eager
  clone-at-boot is trivial for the demonstrative roster; a future
  delivery build with hundreds of Localities may want a lazy/incremental
  scheme.

## `AddressApi` surface

```ts
// resolution — the weather-consumed seam
resolveLocalityFor(scope): Promise<Locality | null>   // full chain (async)
resolveFor(scope): Promise<AddressResolution>          // + matched address + source
traceResolveFor(scope): Promise<AddressTrace>          // + provenance + coverage chain
coveringLocalityOf(address): Locality | null           // SYNC longest-prefix fast-path
findByAddress(address): Locality | null                // SYNC exact-match
findByPath(path): Locality | null                      // SYNC templatePath lookup
// index lifecycle (Locality self-registration; gated)
registerLocality / deregisterLocality / rebuildCoverageIndex
```

The full walk is **async** (the zone-fallthrough step awaits
`Zone.lookupField`, matching `BiomeApi.resolve*For`). The three-tier
narrow-entry pattern is the access precedent: `AddressApi` (thin) →
`AddressLogic` (`/obj/api/address`, stateless resolve orchestration,
gated `FromModule('/api/address#AddressApi')`) → `AddressRegistry`
(the durable index, gated `FromTemplate('/obj/api/address')`).

## The seam weather consumes

- **Contract.** `AddressApi.resolveLocalityFor(scope)` returns the
  nearest covering Locality or `null` — and **`null` === "global field"**
  for weather (a place outside any Locality has one global weather
  state). The deferred weather build is forward-compatible: with no
  addressing authored, every place resolves `null` → global; as
  Localities land, weather upgrades to per-Locality by reading the same
  walk, with no weather-specific structure added.
- **Sync fast-path.** The full walk is async, but
  `coveringLocalityOf(address)` is a sync longest-prefix lookup: a
  consumer resolves the scope's address string once (async, cached on
  its own read path) then does covering lookups synchronously — keeping
  a hot read path (e.g. weather feeding thermal) off the async chain.
- **Locality is the home for future tier-level fields.** This unit
  ships the node and the walk; it ships **no** tier field. Weather hangs
  its per-locality field (seed / overrides) on the Locality later; the
  delivery build hangs provider-coverage refs there. **Realized since:**
  the weather pin + climate lean (weather Wave 2), and the civics
  build's `_governmentKey` (the sparse diegetic-jurisdiction
  declaration — [civics.md](./civics.md)), which also exposed the
  registry's `coverageChainOf` as `AddressApi.coverageChainOf` (the
  sync chain sibling of `coveringLocalityOf`).

## Roster

The slim demonstrative inventory (parallel to Biome's 8-template
roster) — each leaf earns its place, templatePath diverging from the
claimed address:

| templatePath | claims | proves |
|---|---|---|
| `/obj/Locality` (FolderZone) | — | admin/ownership root |
| `/obj/Locality/narnia` | `narnia` | root Region (coverage fallback) |
| `/obj/Locality/cair-paravel` | `narnia/castle` | nested longest-prefix winner |
| `/obj/Locality/lantern-waste` | `narnia/wild` | sibling discrimination + null-outside-tree |
| `/obj/Locality/hinkley-hills` | `terminus/hinkley-hills` | a **sibling of the city, not a child** — a suburb beside Terminus rather than inside it, with a `_governmentKey` of its own |

### The suburb tier (living-world phase 2)

`hinkley-hills` claims `terminus/hinkley-hills`, deliberately **not**
`terminus/city/hinkley-hills`. The address IS the geographic argument: the
city is dense and has no room for a garden, and the Hills are what happens
at its edge. Everything addressed `terminus/hinkley-hills/...` resolves
there rather than to the municipal corporation.

It carries its own `_governmentKey`, so the two are separate jurisdictions
a short walk apart — which is what makes *leaving* a real option long
before there is a law worth leaving over. See
[civics.md](./civics.md) and [smallholding.md](./smallholding.md).

## Verb

`analyze address [<location>]` — no instrument (parallel to `analyze
sky` / `analyze atmosphere`). Dumps the resolved address, its
provenance step, the covering Locality, and the longest-prefix chain
via `traceResolveFor`. Cheap; doubles as the developer debug surface.

## What's NOT in this build

Deferred to later units of the delivery vision (or other builds):
providers / coverage-as-service, anchors, provider-grade off-grid
(absence-of-provider), the post carrier (`delivers` brain), the aether
line / utilities / metering, all Layer-2 comms (mail / forums /
broadcast), the concrete Saxonberg address scheme (content),
trunk/regional sorting, vector wind / weather itself, per-region
latitude/longitude. The eager-clone roster walk is the one flagged v1
simplification.

## Cross-references

- delivery-slate — the seeding
  slate; this is its "Addressing foundation" separable unit. Q4/Q6
  resolved here for this unit; providers/anchors/carrier/aether/Layer-2
  remain deferred.
- [weather.md](./weather.md) — the first consumer (Wave 1 shipped); the
  `resolveLocalityFor` seam above is its load-bearing dependency. Its
  deferred surface lives in
  weather-slate.
- [biome.md](./biome.md) — the resolve-chain shape this mirrors, the
  Pattern-A ref, the slim-roster precedent, and `SkyExposedMixin`
  (weather's indoor gate).
- [zone.md](./zone.md) — `Zone.lookupField` for the zone-fallthrough
  step.
- [access.md](./access.md) — the facade → logic-singleton → registry
  three-tier this mirrors.
- [ref-shapes.md](../ref-shapes.md) — the identity ref for the `_address`
  declaration.
- `lib/collections/PathTrie` — the coverage-index machinery.
