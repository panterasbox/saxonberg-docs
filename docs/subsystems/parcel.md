# Parcel (real-property title)

The parcel subsystem is the **real-property title** layer: it turns the
informal `/domain/<team>/` FolderZone-ownership convention into a
first-class, path-resolved **title stored separately from the content it
gates**. A *parcel* is an ownable, titled extent (a Zone + a title record);
parcels form a sparse hierarchy over the template tree; ownership is
un-fused from authorship; and titles are minted and transferred at runtime.

This is property build **phase 0a** — title only. Chattel/possession, the
lease *mechanics*, payment-coupled `sell`, and the compute-allowance economy
are later phases (see property-slate.md).

## The governing security invariant

**Access-check data lives only in a collection that content-authoring
cannot reach.** Ownership is written **only** by the gated `ParcelApi`
(subdivide/transfer) + gated platform `parcels` seeds — **never** declared
*in* content (not on a zone template, not in a content pack). This is the
*reason* 0a exists: it moves ownership out of the editable `domain`
collection so a content edit can't forge a title, and it is the shape
forward-compatible with the untrusted content-pack/mod future (ownership
declared inside content would be a supply-chain spoof).

## The `parcels` row + `ParcelOwner`

`ParcelRecord` (`lib/parcel/ParcelRecord.ts`) is a plain `Document` in the
`parcels` collection (the `StoredDocument` precedent — a path-keyed row, not
a Stuff):

```ts
class ParcelRecord extends Document {
  static collectionName = 'parcels';
  extent: string;               // the path this parcel claims (the coverage key)
  zonePath: string;             // the backing Zone's templatePath (== extent in 0a)
  owner: ParcelOwner | null;    // the typed title holder
  parentParcel: string | null;  // parent parcel's extent (the sparse-hierarchy edge)
  grants: unknown[];            // INERT 0a seam (0b lease/grant mechanics)
  allowance: unknown | null;    // INERT 0a seam (Phase 1 compute economy)
  landUse: LandUse | null;      // what may be done here; null = inherit
  area: number;                 // declared ground area, m²; 0 = undeclared
  storeys: number;              // how many floors stand on it (default 1)
}
```

`ParcelOwner` is a **typed, discriminated principal** — the access layer
dispatches on its kind:

```ts
type ParcelOwner =
  | { kind: 'group'; name?: string; ref?: GroupRef }  // a managed group
  | { kind: 'player'; templatePath: string };         // an individual
```

- **`group`** — `ref` (an explicit `managed:<id>`) wins when present;
  otherwise `name` is resolved mint-or-find by the registry (so a seeded
  `'lounge'`/`'terminus'`/`'core'` owner maps to a runtime ref without the
  seed knowing the group id).
- **`player`** — keyed on the durable `templatePath` (a self-home owner
  `/home/<key>`, or a title transferred directly to an Avatar).

`grants[]` and `allowance` are **present-but-inert** in 0a — the shape is
the seam so 0b/Phase 1 land additively, no migration.

## Land use and area (living-world phase 2)

`landUse` and `area` are **live**, not seams. Together they are one zoning
act: *"this ground is residential, at this size."*

- **`landUse`** — one of the closed six (`residential · agricultural ·
  commercial · industrial · civic · wild`), defined in
  `lib/parcel/LandUse.ts`. `null` means **inherit**;
  `ParcelApi.landUseOf(path)` resolves by longest prefix through the
  coverage trie and then up the `parentParcel` chain, answering `wild`
  when nothing claims the ground.
- **`area`** — plain m² as a `number`, **declared at provision**, `0` =
  undeclared (which is every row that predates the field, and is never
  policed). `ParcelRegistry.subdivide` refuses a child outside its
  effective use's permissible band, which is the whole of "constrained by
  zoning": one check at mint time, not an ongoing simulation. The band is
  measured against **`area`, not `area × storeys`** — it is a ground-area
  band, and a four-storey building stands on the same dirt as a
  one-storey one. Display banding (`"a quarter-acre lot"`) wraps the
  number at read time via `Quantity.of(area, 'm²').tag('lot')`; the
  stored field stays a scalar so `spaceOf`'s arithmetic needs no
  unwrapping.

> **⚠ `wild` admits nothing, and that default is load-bearing.** Most rows
> in this collection are not ground at all — `/studio`, `/compact`,
> `/obj/lounge` and the `/obj/…` roots are path-branch titles over the
> template tree, and they all answer `wild`. Were `wild` to admit cultivation, it would be
> legal on every branch nobody thought to zone. Leave those rows unzoned;
> the fail-closed answer is the correct one.

### The `/compact` title

`config/parcels.yaml` carves one row for the Compact's **publications**
namespace:

```yaml
  - extent: /compact
    owner: { kind: group, name: core }
```

⚠ **No `landUse`, no `areaM2`** — a path-branch title, not ground. The
`/studio` row is the shape precedent; the *concept* is not (that is a
workspace namespace, the multiseat `/home`). See the four-namespace
taxonomy in [document-store.md](./document-store.md).

It earns its keep three ways: it gives the press feed paths an **owner**,
gives `CompactApi.committeeOf('/compact')` a claimed title to resolve
rather than falling through to the state default — which is what makes
`{kind: 'committee', parcel: '/compact'}` a usable appointing authority
([press.md](./press.md)) — and gives the branch a chain of title that can
be subdivided or transferred later.

⚠ **Quotas over it are parked, deliberately.** The Compact has its own
resource allowance, separate from any locality's, and the authority to set
it is genuinely unresolved because *the executive is the same branch that
provisions parcels in the first place.* `ParcelRecord.allowance` is inert
and the allowance cascade is unbuilt, so nothing here decides it — see
balance-slate.

> **⚠ Area is NEVER derived from room geometry.** `Location.getSizeScale()`
> is m² as well, but it is a **photometric denominator** (flux ÷ area →
> lux) with exactly one consumer. Deriving parcel area from it would make
> placeholder rooms load-bearing *and* promote a lighting constant into a
> land-tenure fact, so every future lighting tweak became a title
> migration. A structure's draw on its parcel is its authored blueprint
> footprint, not a sum over its rooms.

The area band is a **LOT** band, checked only on a subdivided child. A
24-hectare `residential` district (Hinkley Hills) does not contradict a
2-hectare residential *lot* ceiling: a district is what lots are
subdivided out of.

Both fields are declared here in the gated `parcels` collection for the
same reason `owner` is — land use **gates behaviour**, which makes it
access-check data, and access-check data on an editable zone template is
forgeable. See [smallholding.md](./smallholding.md) for the consumer.

## `ParcelRegistry` + the coverage index

`ParcelRegistry` (`obj/ParcelRegistry.ts`, an `Idea + PostRegistrationMixin`
singleton at `/obj/ParcelRegistry`, sibling to `AccessRegistry` /
`AddressRegistry` / `OfficeRegistry`) holds the durable state: a
`PathTrie<ParcelRecord>` **coverage index** keyed on `extent` (the
`AddressRegistry` precedent — extents are path-shaped and longest-prefix is
exactly the nearest-parcel-bearing-ancestor query the hierarchy needs), plus
the mint-or-find group-ref resolution that **moved here out of
`AccessRegistry`**. `postRegister` rebuilds the index idempotently from the
`parcels` collection.

Every public method carries
`@CallSecurity(AnyOf(FromModule('/api/parcel#ParcelApi'),
FromTemplate('/obj/api/parcel')))` — `ParcelApi` (and its `ParcelLogic`
singleton) are the only legitimate callers; external code that grabs the
Registry gets a reference but `SecurityError` on any method call
(narrow-entry).

## `ownerOf(path)` — the total resolution chain

`ParcelApi.ownerOf(path)` is **total** (resolves for any path) via a
three-rung chain — the **governance decision** that default title is
publicly held and there is *no author rung*:

1. **Explicit parcel title** — `coveringParcelOf(path)` longest-prefix; if a
   parcel covers the path, return its `owner`.
2. **Self-home identity** — a path strictly under `/home/<key>/` → that
   player, keyed on the durable `/home/<key>` branch, with **no `parcels`
   row** (`ParcelRecord.selfHomeOwnerOf`, the shared rule — see below).
3. **The state (public default)** — everything else is `{kind:'group',
   name:'core'}`. `'core'` is the state placeholder in 0a; a dedicated
   state/treasury principal is a governance-build refinement.

**There is no author rung.** Authoring confers **credit** (the immutable
`authoring_events` provenance ledger — see [provenance.md](./provenance.md)),
**never title.** You own only what you hold title to; untitled content is the
state's. This makes `ownerOf` **byte-identical** to today's core walk for
untitled content (untitled → `core`).

## The sparse hierarchy

A zone with no parcel row inherits its governing parcel from the nearest
parcel-bearing ancestor: `coveringParcelOf(path)` is
`trie.longestPrefix(path)`. `parentParcel` is stored on the row for O(1)
transfer/subdivide bookkeeping (derivable from the trie, but cheap to keep).
So `/domain/lounge/bar/stool` resolves to the `/domain/lounge` parcel, and a
carve-out at `/domain/lounge/east-wing` shadows it for paths beneath.

## Chain of title (`parcel_events`)

The registry is **log-backed** (the `bank_ledger→bank_accounts` /
`renown_events→renown` sibling shape). `ParcelEvent` (`parcel_events`) is an
**append-only** log: every title event (`subdivide` genesis, `transfer`
handoff) records `{extent, event, from, to, actor, at}` — the `parcels` rows
are the rebuildable current-state cache. A **`transfer` is therefore never a
destructive overwrite**: the prior owner stays recoverable from the log,
preserving ownership lineage (the real-estate-metagame provenance seam, slate
§L). The `actor` is derived from `ExecutionContextApi.getActingAuthor`
(never a parameter). 0a writes the trail; rebuild-from-log and the
chain-of-title *readout* are deferred consumers.

## The Api three-tier

Mirrors the `AccessRegistry` / `AccessApi` / `AccessLogic` shape —
facade (non-HMR) → logic singleton (`/obj/api/parcel`, hot-reloadable) →
state registry:

| File | Role |
|---|---|
| `api/parcel.ts` — `ParcelApi` | Thin forwarding facade; `SecurityApi.decorateApiClass`. |
| `obj/api/ParcelLogic.ts` — `ParcelLogic` | `@internal` logic singleton at `/obj/api/parcel`; gated `FromModule('/api/parcel#ParcelApi')`; resolves the Registry and **degrades gracefully** (no registry → the pure `self-home ?? state` rungs, so `AccessApi.can` stays byte-identical). |
| `obj/ParcelRegistry.ts` — `ParcelRegistry` | The state home + real logic. |

Surface: `ownerOf` / `coveringParcelOf` / `resolveOwnerRef` (group owner →
ref) / `groupOwnerRefs` (the author-scope input) / `subdivide` / `transfer` /
`rebuildCoverageIndex` / `selfHomeOwnerOf` (pure). Internal self-calls route
through ungated private impls (`coveringImpl` / `resolveRefImpl`) — an
intra-singleton call through a gated public method resolves the caller to the
Registry itself, outside the allowlist (the `OfficeRegistry` lesson).

## The `AccessRegistry` repoint

Access-*decision* logic stays in the access layer; title *storage* moved to
parcels. See [access.md](./access.md) for the consumer side. In brief:

- **`can`** — resolves `owner = ParcelApi.ownerOf(zoneOf(resource).path)`,
  then dispatches: a **group** owner resolves to a ref
  (`ParcelApi.resolveOwnerRef`) → `GroupApi.isMember`; a **player** owner is
  an identity match (the Avatar's own `templatePath`, or its `/home/<key>`
  self-home form). Byte-identical for the migrated areas — no seed used
  `accessGroups`, so the old flat-union collapses to the single nearest owner
  (or `core`), which is what `ownerOf` returns.
- **`canMutateZone`** — same substitution; a group owner keeps the `'owner'`-
  role requirement, a player owner is an identity match.
- **`ensureAuthorGroups`** (the `isAuthor` scope) — repointed to
  `ParcelApi.groupOwnerRefs()` + `'core'`; the former template-`data` scan is
  retired.
- **Retired** from `AccessRegistry`: `effectiveOwnerRef`,
  `resolveOwnerGroupName`, `cachedOwnerNameRefs` (the mint-or-find moved to
  `ParcelRegistry`).

## Zone field removal

The three `Zone` ownership fields (`ownerGroup`, `accessGroups`,
`ownerGroupName`), their accessors, and `validateGroupRef` are **deleted**;
`Zone.persistentFields = []`. The zone carries no access controls of its own —
title lives in `parcels`. The folder/leaf invariant (structural,
`prototype instanceof Zone`) and the `lookupField` / `lookupAncestorField`
inheritance walk (celestial/biome defaults) are untouched. `accessGroups`
(the flat-union multi-group ACL) is a **conscious deletion to be re-provided**
by 0b's `grants[]` seam — no seed used it, so removal is byte-identical today.

## Self-home generalization

The shipped `/home/<key>/` self-home ownership rule (the `DocumentLogic`
document-store base case) is **generalized, not forked**:
`ParcelRecord.selfHomeOwnerOf(path)` is the single pure implementation (a
path under `/home/<key>/` → `{kind:'player', templatePath:'/home/<key>'}`,
else null), exposed as `ParcelApi.selfHomeOwnerOf`. Both `ownerOf` rung 2 and
`DocumentLogic.isOwnHomePath` consume it — byte-identical to the former
`path.startsWith('/home/${key}/')` check. See
[document-store.md](./document-store.md).

## Migration (seeds)

Ownership is declared as gated platform `parcels` seed rows — never on the
zone seed. `mud/config/parcels.yaml` (installed by the backend `ParcelSeeder`,
insert-iff-absent on `extent` — the `RecipeSeeder` precedent) carries the two
migrated areas: **lounge** (`/obj/lounge` + `/domain/lounge` → the managed
`lounge` group) and the **Terminus terminal** (`/domain/terminus/terminal` →
the managed `terminus` group — the first *spatial-zone* ownership stamp,
validating "parcel = FolderZone *or* spatial zone"). `ownerGroupName` was
stripped from those zone seeds; `crossroads` (newbie-wilds) stays intentionally
unowned. Idempotent — no world-wide data sweep.

> **Operator follow-up (Risk R1).** The byte-identical claim rests on no
> deployed zone carrying a stray `data.accessGroups` or a nested differing
> `ownerGroup`. No *seed* uses either; a one-time audit of the deployed
> `domain` collection gates the claim on a live box before merge-to-prod.

## The verbs (`system` category)

- **`subdivide <name>`** — carves a titled child out of the parcel governing
  the giver's current location: resolve the governing parcel → gate to its
  owner via `AccessApi.canMutateZone` → mint the child zone via
  `TemplateApi.saveTemplate(childPath, '/obj/FolderZone', …)` (the
  `MkdirController` precedent) → `ParcelApi.subdivide` writes the child row
  (owner inherited, `parentParcel` set) + a genesis event. **FolderZone-
  first** — spatial (grid sub-region) carve-outs are a deferred non-goal.
- **`transfer <parcel> to <player>`** — moves a title under **bilateral
  consent**: gate to the current owner → resolve the receiver (an online
  Avatar) → the receiver accepts via `PromptApi.confirm` on their Interactive
  (decline/timeout aborts) → `ParcelApi.transfer` appends the `transfer`
  event and updates the current-state row. Title-only — `sell` (payment-
  coupled) is deferred (it pulls in banking + Contract atomicity, which must
  not gate the title primitive).

Both derive the acting principal from execution context, never a parameter;
both funnel through `ParcelApi`, the only legitimate caller of the Registry.

## Boot

`bootstrap.ts` manifest entry `{ templatePath: '/obj/ParcelRegistry',
dependsOn: ['/obj/GroupRegistry'] }`, after `/obj/AccessRegistry` (which
consults it). `TemplatePaths.parcelRegistry` names the path. `ParcelSeeder`
runs in the per-collection seeder block (after `PersistenceManager.connect`,
before `BootstrapManager.run`). The `parcels` / `parcel_events` collections +
their indexes are declared in `PersistenceManager` (declaring
`collectionName` is sufficient; the indexes are `extent`/`parentParcel`).

## What's NOT in 0a

- **Chattel / possession** + `PersistableHolder` → 0b.
- **Lease mechanics** (`useRightOf`, the lease verb, revert-on-leave) — the
  `grants[]` field ships inert in 0a; the **use-grant lease is now typed +
  live** (`UseGrant`, `grantUse`/`revokeUse`/`hasUseGrant`/`heldUnitOf`,
  plus `childParcelsOf`/`retire` for unit provisioning), consumed by the
  leased dorm — see [residence.md](./residence.md). Payment-coupled rent
  economics stay → 0b/Phase 3.
- **`sell` / payment-coupled transfer + Contract atomicity** → 0b/later.
- **Compute allowance metering + degradation** → Phase 1 (`allowance` inert).
- **Coordinate-region ("region") sub-zone spatial ownership** → deferred,
  additive.
- **Rebuild-from-log + the chain-of-title readout** — 0a only writes the
  trail.

## History

Property phase 0a landed on the `feature/property-0a-title` branch (commit
`81b250be`). Two shifts from the plan are worth recording. First, the plan's
blast-radius analysis predated the Terminus merge (`ec5e5353`) and had to be
re-derived: the removal covered **three** Zone fields (not two) across **two**
owner groups (lounge + Terminus), and retired the data-driven
`effectiveOwnerRef`/`resolveOwnerGroupName` machinery the Terminus merge had
introduced. Second, the R2a governance decision — **default title is the
state, the author rung removed** — is what makes the `ownerOf` chain
byte-identical to today's core walk (untitled → `core`); an earlier draft had
"owner defaults to author," which would have changed authored-content access.

## Cross-references

- [access.md](./access.md) — the consumer side (`can` / `canMutateZone` /
  `isAuthor` repointed onto `ParcelApi`; the Zone field removal).
- [zone.md](./zone.md) — `FolderZone`, the `lookupField` inheritance walk
  (the ownership fields it used to document now live here).
- [address.md](./address.md) — the `AddressRegistry` + `PathTrie.longestPrefix`
  coverage-index precedent this reuses.
- [document-store.md](./document-store.md) — the `/home/<self>/` self-home
  base case generalized into `ParcelApi.selfHomeOwnerOf`.
- [provenance.md](./provenance.md) — the `authoring_events` ledger; authorship
  is credit, orthogonal to title (the un-fusing).
- property-slate.md — the design surface
  (§A parcel = a Zone + hierarchy, §B separate collection + author≠owner, §K
  dorm = implicit default parcel; and the later phases).


## History — the furnishing build (2026-07-31)

Acreage landed. See [furnishing.md](./furnishing.md) § Acreage.

- **`ParcelRecord.area`** (m², gross ground) and **`storeys`** (default 1).
  Declared at provision, **never derived**.
- **`subdivide` conserves children against `area × storeys`**, not `area`.
  Multi-storey is why: a 300 m² footprint at four storeys offers ~1200 m²
  of interior, so a rule written against ground area alone refuses
  apartments on the second floor. `storeys` rather than a `grossFloorArea`
  field because the row **already encoded floors** — `slotOfExtent` parses
  `f<floor>-r<pos>`, built for the dorm.
- **`ParcelApi.spaceOf`** — the space account: `capacity` (`area ×
  storeys`), `allocated` (Σ children), `unallocated` and `utilisation`.
  Reads as **coverage + yard** on a lot and **efficiency + common area** in
  a building. All derived on read, none stored — a ceiling alone is useless,
  these are the numbers anybody plans against.
- **`ParcelApi.workableAreaOf`** = the `unallocated` half, derived on
  read and never stored. Any child consumes ground whether it is a building
  or a sub-lot, so a building's **footprint needs no field**: it *is* that
  building parcel's own `area`.
- Unmeasured land is not policed — a parcel with no declared area
  subdivides exactly as it did before these fields existed.
