# Parcel (real-property title)

The parcel subsystem is the **real-property title** layer: it turns the
informal `/world/<team>/` FolderZone-ownership convention into a
first-class, path-resolved **title stored separately from the content it
gates**. A *parcel* is an ownable, titled extent (a Zone + a title record);
parcels form a sparse hierarchy over the template tree; ownership is
un-fused from authorship; and titles are minted and transferred at runtime.

This began as property build **phase 0a** — title only — and has since grown
the **use-grant (lease)** half, which is live (see
[§ `grants[]`](#grants--the-use-grant-lease-and-it-is-live)). Still later
phases: payment-coupled `sell` and rent economics, and the compute-allowance
economy (see property-slate.md).
Chattel/possession shipped separately —
[chattel.md](./chattel.md).

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
  grants: UseGrant[];           // LIVE — the use-grant (lease) relationship
  allowance: unknown | null;    // INERT 0a seam (Phase 1 compute economy)
  landUse: LandUse | null;      // what may be done here; null = inherit
  area: number;                 // declared ground area, m²; 0 = undeclared
  storeys: number;              // how many floors stand on it (default 1)
  reach: string;                // the reach this land fronts; '' = not on water
}
```

⭐ **`reach` is a citation, not a derivation** (water build). Riparian
right belongs to land that *fronts* water, and the obvious way to find
that — walk the address tree and see what the parcel sits under — does
not work here, because **the address tree does not mirror the content
tree**: the market square lives at `/world/terminus/market/square` and is
addressed `terminus/city/counting-houses/market-square`. So a parcel
names its reach directly — a `<watercourse>:<reach>` citation like
`kestrel:confluence` — and the riparian holders of a reach are exactly
the owners of the parcels citing it, their shares equal by construction,
so a drought shrinks every glass rather than emptying the junior ones.
`''` is the ordinary case: most land fronts nothing. Prior appropriation is
the *recorded* doctrine (a `water-right` document); riparian **derives**
over these citations, so which doctrine a polity runs is a choice rather
than a rebuild. See [watershed.md](./watershed.md).

`ParcelOwner` is a **typed, discriminated principal** — the access layer
dispatches on its kind:

```ts
type ParcelOwner =
  | { kind: 'group'; name?: string; ref?: GroupRef }  // a managed group
  | { kind: 'player'; templatePath: string }          // an individual
  | { kind: 'organization'; templatePath: string };   // an Organization host
```

- **`group`** — `ref` (an explicit `managed:<id>`) wins when present;
  otherwise `name` is resolved mint-or-find by the registry (so a
  manifest's `'lounge'`/`'terminus'` holder maps to a runtime ref without
  the pack knowing the group id).
- **`player`** — keyed on the durable `templatePath` (a self-home owner
  `/home/<key>`, or a title transferred directly to an Avatar).
- **`organization`** (content-packs wave 3) — keyed on the organization's
  `templatePath`. The title admits everyone holding a non-exited position
  in it (`EmploymentApi.holdsPosition`) and its appointing authority
  (`EmploymentApi.holdsAuthority` — an office's holder, founder default
  included). The organization **must be resident** for the title to admit
  anyone; a non-resident target fails closed with one `access.organization-
  owner` diagnostic per path. This is the kind that lets the executive
  (`/compact/executive`) hold the platform.

## `grants[]` — the use-grant (lease), and it is LIVE

⚠ **Corrected 2026-08-06.** This section previously described `grants[]` as
a *"present-but-inert 0a seam."* That is no longer true and has not been for
some time — the typed use-grant landed with the dorm tenancy, and
[§ What's NOT in 0a](#whats-not-in-0a) already recorded it while the schema
block above still called it inert. **`allowance` remains genuinely inert**;
only `grants[]` changed.

A **use-grant** is a tenant's time-bounded right to occupy and use an extent,
**distinct from its title** — the lease half of the property model:

```ts
interface UseGrant {
  kind: 'lease';
  holder: string;          // the tenant's durable player templatePath
  grantedAt: number;
  expiresAt: number | null; // epoch-ms; null = indefinite
}
```

The surface is built, gated (`ParcelApiCallers`), sandbox-guarded and
persisted — `ParcelApi` → `ParcelLogic` → `ParcelRegistry`:

| Call | Behavior |
|---|---|
| `grantUse(extent, holder, expiresAt)` | grants a lease, **replacing** any prior grant for that holder (holder-keyed, at most one grant each); mutates the live trie handle in place so `hasUseGrant` reads the same object |
| `revokeUse(extent, holder)` | removes the grant; `false` when none existed |
| `hasUseGrant(extent, holder)` | **expiry-honoring** — delegates to `ParcelRecord.hasActiveGrant(record, holder, now)` |
| `heldUnitOf(holder, underExtent?)` | the reverse index — the unit a holder currently leases, optionally scoped to one institution's wing (⚠ a linear scan; see below) |
| `heldUnitsOf(holder)` | the PLURAL — every unit the holder holds an active grant on (the residence ladder's ascent gate reads this: a dorm lease, an apartment lease and a title are three rungs of one ladder — residences build) |

`ParcelRecord.activeGrantFor` / `hasActiveGrant` are the static readers every
expiry check goes through, so an expired grant is inert without needing a
sweep to remove it.

⭐ **`revokeUse` has a real consequence path**: a holder revoked while inside
the extent's sandbox circle is **reaped** — their wire body exits and
re-attaches at their parked avatar, the non-event eviction the wire-body model
makes free (see [sandbox.md](./sandbox.md)). Eviction is not merely a row
edit.

⚠ **The one-lease-per-holder assumption is load-bearing and undocumented
elsewhere.** `heldUnitOf` returns the *first* matching row, which is correct
only while a player holds at most one lease (true for the v1 dorm). The
residences build **worked around rather than closed** this: `heldUnitsOf`
is the plural read the ascent gate needs, and the dorm's already-housed
check scopes to its own wing with `underExtent`. `heldUnitOf`'s singular
answer is still first-match, and [civics.md](./civics.md) still defers
**primary-home designation** — the two are the same gap seen from
different ends.

**Consumers today:** the Duncan Hall `provision` / `unprovision` pair, the
Seznick House `lease` / `unlease` pair, and the ascent gate at `title buy`
(see [holding.md](./holding.md)). Payment-coupled rent economics remain
deferred; a grant-event log (the chain-of-title equivalent for leases,
beside `parcel_events`) is still a seam.

### Keys mint at the parcel, at three chokepoints

A parcel carries the **keyway** its holding's door is gated on, and it is
written at exactly three places — the sale (`title buy`), the grant
(`lease`) and the dorm's `provision` — each doing the same three calls in
the same order:

```
Lock.mintKeyway() → ParcelApi.setKeyway(extent) → Lock.issueKey(holder, keyway, 'pin-tumbler')
```

Re-keying on release is what makes an ex-tenant's key dead metal rather
than a standing hole in the building. The door reads the keyway through
its institution's cache, synchronously — an empty keyway admits nobody.
See [credential.md](./credential.md) and
[holding.md](./holding.md#the-way-in--keys-doors-and-the-ascent-gate).

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
> `/stuff/idea/lounge` and the `/platform/… + /stuff/…` roots are path-branch titles over the
> template tree, and they all answer `wild`. Were `wild` to admit cultivation, it would be
> legal on every branch nobody thought to zone. Leave those rows unzoned;
> the fail-closed answer is the correct one.

### The `/compact` title

The platform pack's manifest claims the Compact's **publications**
namespace for the executive (`packages/content/platform/pack.yaml`):

```yaml
requires:
  title:
    - { extent: /compact }   # held by the maintainers — the executive
```

⚠ **No `landUse`, no `areaM2`** — a path-branch title, not ground. The
`/studio` row is the shape precedent; the *concept* is not (that is a
workspace namespace, the multiseat `/home`). See the four-namespace
taxonomy in [document-store.md](./document-store.md).

It earns its keep three ways: it gives the press feed paths an **owner**,
gives `CompactApi.committeeOf('/compact')` a claimed title to resolve
(there is no state default to fall through to) — which is what makes
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
singleton at `/platform/idea/ParcelRegistry`, sibling to `AccessRegistry` /
`AddressRegistry` / `OfficeRegistry`) holds the durable state: a
`PathTrie<ParcelRecord>` **coverage index** keyed on `extent` (the
`AddressRegistry` precedent — extents are path-shaped and longest-prefix is
exactly the nearest-parcel-bearing-ancestor query the hierarchy needs), plus
the mint-or-find group-ref resolution that **moved here out of
`AccessRegistry`**. `postRegister` rebuilds the index idempotently from the
`parcels` collection.

Every public method carries
`@CallSecurity(AnyOf(FromModule('/api/parcel#ParcelApi'),
FromTemplate('/platform/idea/api/parcel')))` — `ParcelApi` (and its `ParcelLogic`
singleton) are the only legitimate callers; external code that grabs the
Registry gets a reference but `SecurityError` on any method call
(narrow-entry).

## `ownerOf(path)` — the total resolution chain

`ParcelApi.ownerOf(path)` returns `ParcelOwner | null` via a two-rung
chain — there is *no author rung* and, since content-packs wave 3, **no
state rung**:

1. **Explicit parcel title** — `coveringParcelOf(path)` longest-prefix; if a
   parcel covers the path, return its `owner`.
2. **Self-home identity** — a path strictly under `/home/<key>/` → that
   player, keyed on the durable `/home/<key>` branch, with **no `parcels`
   row** (`ParcelRecord.selfHomeOwnerOf`, the shared rule — see below).
3. Otherwise **`null`** — untitled. Every access predicate (`can`,
   `canMutateZone`, `canAtPath`) **fails closed** on `null`: nobody holds
   what nobody claimed. The former third rung — "everything else is the
   `core` group" — was the state placeholder of property 0a; wave 3
   deleted it along with the group, because the platform's own roots are
   now explicit claims held by the executive (see *Who claims what*).

**There is no author rung.** Authoring confers **credit** (the immutable
`authoring_events` provenance ledger — see [provenance.md](./provenance.md)),
**never title.** You own only what you hold title to; untitled content is
nobody's, and a verb over it is refused until somebody claims it
(`pnpm lint:untitled` proves every shipped path sits under a claim).

## `grant` — how title comes into being

Title is **declared by content packs** (`requires.title` in a manifest —
[content-packs.md](./content-packs.md)) and written by
`ParcelApi.grant(claim: TitleClaim)` → `{ outcome, holder }`. A
`TitleClaim` is `{ extent, holder: ParcelOwner, parentParcel?, landUse?,
areaM2? }` (the shape checks `ParcelSeeder` used to run now live in
`ParcelRegistry.validateClaim`). The outcome (`TitleGrantOutcome`):

| Outcome | When | Effect |
|---|---|---|
| `granted` | no row at `extent` | row written + a `grant` event (`ParcelEvent.event` is `subdivide \| transfer \| grant`) |
| `kept` | the row exists under the **same** holder | untouched — two packs may claim one extent for one holder (`world-seed` and `saxonberg-lounge` both name `/world/lounge` for `lounge`) |
| `conflict` | the row exists under a **different** holder — whoever that is | untouched; the installer records it and the pack reconciles bounded (its rows under that extent are `skip-sold`) |

There is **no migration outcome** (wave 4a deleted wave 3's `migrated`
branch — the `core`-held and retired-corpo-board hand-overs — and the
sold predicate's `core` exemption with it): "no new title over an
existing one" is read literally. A database whose title predates a
rename is dropped, never migrated (deployment.md).

Grants run **before** the pack's rows are planned (a first-boot claim of
`/corpo/<key>` must exist before the chart row under it is judged
sold-or-not), and the nightly reset's `PackApi.reprovision()` re-runs
every applied pack's grants after the wipe takes `parcels`
([record-layer.md](./record-layer.md)).

## The sparse hierarchy

A zone with no parcel row inherits its governing parcel from the nearest
parcel-bearing ancestor: `coveringParcelOf(path)` is
`trie.longestPrefix(path)`. `parentParcel` is stored on the row for O(1)
transfer/subdivide bookkeeping (derivable from the trie, but cheap to keep).
So `/world/lounge/location/bar/stool` resolves to the `/world/lounge` parcel, and a
carve-out at `/world/lounge/east-wing` shadows it for paths beneath.

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
facade (non-HMR) → logic singleton (`/platform/idea/api/parcel`, hot-reloadable) →
state registry:

| File | Role |
|---|---|
| `api/parcel.ts` — `ParcelApi` | Thin forwarding facade; `SecurityApi.decorateApiClass`. |
| `platform/idea/api/ParcelLogic.ts` — `ParcelLogic` | `@internal` logic singleton at `/platform/idea/api/parcel`; gated `FromModule('/api/parcel#ParcelApi')`; resolves the Registry and **degrades gracefully** (no registry → the pure `self-home ?? state` rungs, so `AccessApi.can` stays byte-identical). |
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
  self-home form); an **organization** owner is staff-or-authority (above).
  A `null` owner denies.
- **`canMutateZone`** — same substitution; a group owner keeps the `'owner'`-
  role requirement, a player owner is an identity match, an organization
  owner the same staff-or-authority test.
- **`heldExtents(subject)`** — the inverse read: every `parcels` row whose
  holder admits the subject through the same dispatch, plus the self-home
  root — what `broadcast`, `teleport`, `find` and the CMS tree scope to.
- The former **`ensureAuthorGroups` / `isAuthor`** scope is gone with the
  author tier (wave 3): capability is title over a resource, never a tier.
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
by the `grants[]` seam — no seed used it, so removal is byte-identical today.
⚠ **Still outstanding.** `grants[]` has since shipped, but as a *lease*
(`kind: 'lease'`, a single player `holder`) — **not** as a multi-group ACL, so
this promise is not yet redeemed. Re-providing `accessGroups` needs either a
second `UseGrant.kind` or a group-shaped holder.

## Self-home generalization

The shipped `/home/<key>/` self-home ownership rule (the `DocumentLogic`
document-store base case) is **generalized, not forked**:
`ParcelRecord.selfHomeOwnerOf(path)` is the single pure implementation (a
path under `/home/<key>/` → `{kind:'player', templatePath:'/home/<key>'}`,
else null), exposed as `ParcelApi.selfHomeOwnerOf`. Both `ownerOf` rung 2 and
`DocumentLogic.isOwnHomePath` consume it — byte-identical to the former
`path.startsWith('/home/${key}/')` check. See
[document-store.md](./document-store.md).

## Who claims what (the manifests)

Ownership is declared in **pack manifests** — never on the zone template,
and no longer in a seed file (`config/parcels.yaml` and the backend
`ParcelSeeder` were deleted in content-packs wave 3; every claim is an
explicit `requires.title` entry, there is no implicit root claim). Today:

- **`platform`** — `/platform`, `/obj`, `/cmd`, `/blueprints`, `/compact`,
  `/studio`, `/home`, `/world`, all held by its maintainers, the
  executive organization `/compact/executive`; and `/wiki` for the
  `wiki-editors` group.
- **`saxonberg-lounge`** — `/stuff/idea/lounge` + `/world/lounge` → the `lounge`
  group.
- **`world-seed`** (transitional) — `/world/lounge` again for `lounge`
  (`kept`), the Terminus municipality's ground (`/world/terminus/terminal`,
  `counting-houses`, `general-store`, `registry` → `terminus`), Hinkley
  Hills and its `lot-1` → `hinkley-hills`, Duncan Hall and its `dorms` →
  `duncan-hall`.
- **`expression`** — `/expression` → the `soul` group;
  **`newbie-wilds`** — `/world/newbie-wilds` → `newbie-wilds`; the
  object packs (`generic-objects`, `species-and-names`, …) claim their
  `/stuff/<branch>/<cluster>` branches; each **corpo pack** claims `/corpo/<key>` for
  its organization.

A claim's group is `ensureGroup`d by name (found if present, minted
empty otherwise) before the grant. Idempotent — a second boot is all
`kept`.

## The verbs (`system` category)

- **`subdivide <name>`** — carves a titled child out of the parcel governing
  the giver's current location: resolve the governing parcel → gate to its
  owner via `AccessApi.canMutateZone` → mint the child zone via
  `TemplateApi.saveTemplate(childPath, '/platform/idea/FolderZone', …)` (the
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

The platform pack's `boot:` list carries `{ template: /platform/idea/ParcelRegistry,
role: sync-read, dependsOn: [/platform/idea/GroupRegistry] }` — "the title coverage
trie every `ownerOf` read walks"; `BootstrapManager.run()` stands it up
from the packs' boot union (the code manifest `bootstrap.ts` is gone).
`TemplatePaths.parcelRegistry` names the path. Grants happen inside
`PackApi.install()`, before `BootstrapManager.run`, so the registry
resolves its rows from the store the moment it is resident. The `parcels`
/ `parcel_events` collections +
their indexes are declared in `PersistenceManager` (declaring
`collectionName` is sufficient; the indexes are `extent`/`parentParcel`).

## What's NOT in 0a

- ✅ **Chattel / possession — SHIPPED** as property Phase 0b, the parcel's
  twin for movables: [chattel.md](./chattel.md).
- ✅ **Lease mechanics — SHIPPED, no longer deferred.** The typed use-grant
  (`UseGrant`, `grantUse`/`revokeUse`/`hasUseGrant`/`heldUnitOf`, plus
  `childParcelsOf`/`retire` for unit provisioning) is live and consumed by the
  leased dorm — see [§ `grants[]`](#grants--the-use-grant-lease-and-it-is-live)
  above and [residence.md](./residence.md). Still deferred: **payment-coupled
  rent economics** (→ 0b/Phase 3), a **grant-event log**, and **multi-residence**
  (`heldUnitOf` assumes one lease per holder).
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
introduced. Second, the R2a governance decision — **the author rung
removed** — kept authored-content access unchanged; an earlier draft had
"owner defaults to author." The state rung that decision *kept* (untitled
→ `core`) lasted until content-packs wave 3, when explicit platform claims
made it unnecessary and it was deleted (`ownerOf` → `null`).

## Cross-references

- [access.md](./access.md) — the consumer side (`can` / `canMutateZone` /
  `canAtPath` / `heldExtents` over `ParcelApi`; the Zone field removal).
- [content-packs.md](./content-packs.md) — `requires.title`, the requires
  phase, the covered-extent rule, `reprovision`.
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

## The document store's gate

`ownerOf` is also what the **document store** gates on (content-packs
wave 2, D11): `DocumentApi.save` / `delete` ask
`AccessApi.canAtPath(actor, 'write-document', path)` — the covering title
through this chain, then that owner's `can()` dispatch — after the
self-home short-circuit. A pack's document branch is **claimed** by its
manifest (`/expression` by the `soul` group, `/platform` by the
executive); a player's `/home/<self>/` is rung 2; an unclaimed path is
`null` and the write is refused. See
[document-store.md](./document-store.md) and [access.md](./access.md).
