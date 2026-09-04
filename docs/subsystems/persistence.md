# Persistence

Saxonberg persistence runs on **two tracks**, split by *what the thing
is*:

1. **`Document`** — plain MongoDB-backed records: auth/meta data
   (`User`, `GoogleProfile`) and CMS assets (`Template`). A standalone
   base (**NOT** in the Stuff hierarchy) that adds an explicit
   `save`/`delete`/`findById`/`find` CRUD surface over MongoDB.
   Construction is a plain `new T()` — no proxy wrap, no `StuffApi`
   registry membership, no security gate, no create/destroy lifecycle.
   A loaded `Document` is a plain object you read and drop. Documents
   are **value-like**: two `findById` calls for the same id return two
   distinct instances.
2. **Templates → Stuff** — every game-world object (rooms, doors, props,
   avatars, NPCs). Stored as `Template` records (themselves `Document`s)
   in the `domain` collection, **cloned** into runtime `Stuff` instances
   by the clone/hydrate/save-template pipeline. Documented in
   [templates.md](./templates.md); not repeated here. Stuff is
   **identity-like**: the registry guarantees one canonical live
   instance.

The split is the `Document`-vs-`Stuff` distinction: a `Document` *is*
persisted state (the row is the thing), while a `Stuff` is a live
world entity *hydrated from* a Document (data in, entity out) — a Stuff
is never itself a row. (This supersedes the former `Persistable extends
Idea` design, where every persisted record was a full Stuff; see the
persistence-architecture slate
for the rationale. The deferred tail — un-Stuffing marshallers/hooks —
is *not* part of that change; they remain Idea-rooted Stuff for HMR.)

This doc covers the `Document` track and the cross-cutting machinery
(`PersistenceManager`, around-hooks, `Collections` enum) used by both.

## `Document`

`Document` lives at `lib/persistence/Document.ts` and is a **plain
class** — it does not extend `Idea`/`Stuff`. There is no proxy, no
registry membership, no security gate, and no lifecycle. It carries
only the CRUD + serialization surface (`save` / `delete` / `findById` /
`find` / `toDocument` / `fromDocument`), the `createdAt` / `updatedAt`
timestamps, and the static `collectionName` / `fieldMeta`'s persistent entries
contract.

```typescript
class User extends Document {
  static collectionName = 'users';
  static fieldMeta: FieldMeta = {
    googleProfileId: { persistent: true },
    playerIds: { persistent: true },
  };

  googleProfileId: string = '';
  playerIds: string[] = [];
}

const user = new User();
user.googleProfileId = '...';
await user.save();

const found = await User.findById(id);
const matches = await User.find({ googleProfileId: 'xyz' });
await user.delete(); // deletes the row — no registry/destruct cascade
```

Subclass contract:

- **`static collectionName: string`** — required. Throws if missing.
- **`static fieldMeta`'s `{ persistent: true }` entries** — fields to copy
  to/from the document. Mixin-contributed fields are picked up
  automatically (see "Field aggregation" below).

Provided by the base class:

- `save()` — sets `updatedAt`, builds the doc via `toDocument()`,
  delegates to `PersistenceManager.save(collection, doc)`, sets `_id`
  on first save.
- `delete()` — throws if `_id` is missing; deletes via
  `PersistenceManager`. **No `StuffApi.destruct` cascade** — a Document
  is not registered, so there is no runtime instance to unregister.
- `findById(id)` / `find(query)` — static. Construct fresh instances
  via a plain `new this()` (loaded instances are plain objects, **not**
  registered or proxy-wrapped), populate via `fromDocument`. Return
  `null` / `[]` when nothing matches.
- `toDocument()` / `fromDocument()` — copy persistent fields plus
  `createdAt` / `updatedAt`. `_id` is included on `toDocument` only
  when present. For the rare marshalled field, resolution goes through
  an **injected resolver seam** (`setDocumentMarshallerResolver`, wired
  once in `AppBootstrap.run`) so the persistence core never imports
  `StuffApi`; marshallers themselves remain Idea-rooted Stuff.
- `createdAt` / `updatedAt` — auto-managed. Set in constructor;
  `updatedAt` refreshed on every `save()`.

Current inhabitants of `Document`: `User`, `GoogleProfile`, and
`Template`. The first two are auth/meta records; `Template` is a CMS
asset (the doc you clone game-world objects from — see
[templates.md](./templates.md)).

## Field Aggregation

`Document.getAllFields()` returns the union of:

1. The class's own a `persistent` entry in `fieldMeta`.
2. Every a `persistent` entry in `fieldMeta` declared by mixins in the prototype
   chain.

The walk is centralised in `MixinApi.getAllPersistentFields(constructor)`
— `Document` calls it automatically. It is constructor-static (no
instance/Stuff coupling), so the same walk serves the `Document` CRUD
path **and** the Stuff `PersistentHydrator` used by templates. A
subclass MAY override `static getAllPersistentFields()` for an escape
hatch, but this is rare.

This also means: a mixin author who adds a new persistent field declares
it once on the mixin, and every consumer (a `Document` record, a Stuff
template) gets it for free. No subclass changes required.

## `PersistenceManager`

`PersistenceManager.get()` returns the singleton. It owns:

- The MongoDB connection (`MongoClient`, `Db`).
- The `Collections` vocabulary — re-exported, not defined here (below).
- `save` / `findById` / `find` / `delete` operations.
- A hook registry (around-save / around-delete chains).

The save and delete entry points dispatch through registered hooks for
that collection before the terminal MongoDB write. Hooks may transform
the doc, short-circuit, or wrap the operation. The terminal `next`
performs the upsert / delete.

```typescript
// Direct call (works regardless of hooks):
const id = await PersistenceManager.get().save(Collections.Users, doc);
await PersistenceManager.get().delete(Collections.Domain, id);
```

The dispatch keeps a `Set<string>` of currently-active
`(collection, operation)` slots. Re-entry into the same slot from inside
its own dispatch throws `HookReentryError`. Loud failure beats a silent
loop.

### The resident `content` cache

`PersistenceManager` holds the whole `content` collection in memory —
path → row — and answers a bare `find(content, { path })` from it.

**Why it is allowed to hold the whole thing.** `content` is the authored
world: on the shipped packs, 991 rows totalling under 600 KB. It is
read-mostly and read *by path*, thousands of times a boot — the clone
pipeline's template load, and the zone walk's one read per ancestor path
— and each of those was a serialized ~30 ms round trip.

**The MISS is the point.** Because the map holds the entire collection,
an absent path is an *answer*, not a reason to ask. `resolveZoneForPath`
walks every ancestor of a template path, and most of those rows do not
exist; each was a round trip to learn nothing.

**Invalidation is by construction, not by enumeration.** Every write to
every collection lands in `persistSave` / `persistDelete` / `deleteMany`
on this same object, so the cache is updated at the chokepoint that
writes. There is no list of CMS-save / pack-install / go-live /
`restoreFromTemplate` / hot-reload callers to keep in sync, and a new
writer cannot forget to invalidate. A save that re-paths a row (the `mv`
shape: same `_id`, new `path`) evicts the path that id used to hold; a
bulk `deleteMany` drops the cache whole.

**Two conditions gate it, and both are load-bearing.**

- **A live connection.** The cache is a property of one process owning
  one database. A `PersistenceManager` with a stubbed collection and no
  `db` — every unit test — reads through, so no test inherits another
  test's cached rows. It follows that a *second* process writing
  `content` against the same database would not be seen; one process
  per database is the deployment (deployment.md) and
  the four-database rule keeps the dev worktrees apart.
- **`content` is still a sandbox `pass` collection.** A cached row is
  one row for every reader, which is only true while
  `composeScopeReadFilter` leaves its queries untouched. Should the
  policy ever become `stamp` or `shadow`, the cache disengages itself
  rather than serving one circle's row to another.

⚠ **A write that lands while the preload query is in flight** is not in
the snapshot that query returns, and has no map to fold itself into —
so it is buffered and replayed the moment the map exists. Without that,
a row written in the window between issuing the read and assigning the
map would be invisible for the life of the cache.

Rows are handed out as `structuredClone` copies, so a caller mutating
what it got cannot corrupt the cache. A row that will not clone is left
out of the map and read through — a slower right answer rather than a
fast wrong one.

⚠ **A cache is not a substitute for a query.** Only a bare `{ path }`
read is served; anything with more terms, or with sort/limit/skip, goes
to Mongo unchanged.

### The query trace

`SAXONBERG_QUERY_TRACE=1` counts every op at the chokepoint, attributes
it to a chain of caller frames outside the persistence and
call-security plumbing, and dumps a table ranked by time-in-Mongo every
30 s and on `SIGUSR2`.

It exists because **a CPU profile is the wrong instrument for a process
that is waiting.** A boot that is 96% blocked on round trips profiles as
76% idle, and the profile names no query. This prints the count, the
collection and who asked. Off by default: one boolean test per op.

## `PersistApi` — the gated chokepoint + `lint:pm` lockdown

`PersistenceManager` is an **ungated process singleton** — reaching it via
`PersistenceManager.get()` from domain / logic code bypasses the security
layer entirely. `PersistApi` (`api/persist.ts`) is the sanctioned,
call-security-decorated facade over it (`isConnected` + the document
data-ops `find` / `findById` / `save` / `delete`); non-framework code
flows through here and picks up the Api security treatment.

The lockdown is enforced by **`lint:pm`** (`scripts/check-pm-access.ts`,
CI-gating, mirrors `check-gate-strings`): `PersistenceManager.get()` is
forbidden everywhere except the persistence framework itself
(`lib/persistence/**` = `Document`, `lib/stuff/Template.ts`), the backend
(`backend/**` — it owns PM's lifecycle: connect / seed / hooks),
`api/hot-reload.ts` (the HMR hook-manifest reload — PM lifecycle, not
data), the facade, and tests. The logic-layer callers
(`Renown` / `Group` / `Chronicle` / `Belief` `active()` guards) migrated to
`PersistApi.isConnected()`. Caller policy starts permissive — the value is
the single decorated chokepoint plus the lint, both tightenable later;
`Document` / `Template` stay the sanctioned low-level boundary
(allowlisted), so the facade's data-ops have no callers yet in v1.

## Around-Save / Around-Delete Hooks

Hooks are middleware. Two mixins compose the capability onto an `Idea`
subclass:

```typescript
// AroundSaveHookMixin
async aroundSave(
  collection: string,
  doc: Record<string, unknown>,
  next: (doc: Record<string, unknown>) => Promise<string>
): Promise<string>;

// AroundDeleteHookMixin
async aroundDelete(
  collection: string,
  id: string,
  next: (id: string) => Promise<void>
): Promise<void>;
```

Default implementations are pass-through (`return next(doc)`). Subclasses
override.

A hook is a regular `Idea`-rooted Stuff. It's instantiated via the CMS
template pattern (`StuffApi.clone()`) at boot, and registered against PM
slots via a YAML manifest:

```yaml
# obj/hooks/hooks.yaml
hooks:
  - collection: domain
    operation: save
    template: /platform/idea/hooks/DomainHook
  - collection: domain
    operation: delete
    template: /platform/idea/hooks/DomainHook
```

`PersistenceManager.loadHooks(yamlPath?)` reads the manifest, clones
each named template, narrows with `MixinApi.isAroundSaveHook` /
`isAroundDeleteHook`, and registers a forwarding closure with
`PM.registerHook(collection, operation, fn)`.

Multiple hooks may register against the same slot — they execute in
registration order, each receiving `next` to invoke the rest of the
chain (terminating in the actual MongoDB write).

The canonical hook today is `DomainHook` (`obj/hooks/DomainHook.ts`),
which composes both around-save and around-delete and enforces the
folder/leaf invariant on the `domain` collection. See
[templates.md § TemplateApi & the Folder/Leaf Invariant](./templates.md#templateapi--the-folderleaf-invariant)
for the rule it enforces. `DomainHook.aroundSave` also calls
`TemplateApi.validateSingletonContainerTarget` — the singleton-target
check for the `data.container` declarative-content field shipped with
the spawn substrate.

### Avatar persist-back rides the self-persistence spine

`Avatar.save()` → `PersistableApi.capture(this)` and `Avatar.restore()` →
`materialize` — Avatar persists through the universal spine (below), NOT the
retired `snapshotToTemplate`. Its record (`holder_snapshots`, not the avatar
template) carries fields + carried inventory + worn gear + spawn location.
See [The self-persistence spine](#the-self-persistence-spine-persistable) §
Avatar on the spine.

## Setter-Based Field Invariants

For both tracks, **per-field shape invariants belong on setters, not in
post-hydrate `normalize()` hooks**. `PersistentHydrator`'s two-phase
dispatch — Phase 1 prefers `set<Field>` methods (or bracket-assigns
through an accessor pair when no setter exists); Phase 2 calls
`apply<Field>` for instruction fields — fires the rule for free
during template hydration. Cross-field invariants — "if `isLocked`
is true, `lockKey` must reference a real key" — go in a custom
`Hydrator` subclass.

Full rule: [antipatterns.md § Per-Field Invariants](../antipatterns.md#per-field-invariants-belong-on-setters-not-in-normalize-hooks).

## Scalar-Default Rule

**Persistent fields default to scalars and arrays of scalars.** That
covers nearly everything Saxonberg persists: booleans, numbers,
strings, primitive tuples like `[x, y, z]`, keyword lists,
templatePath strings for Stuff cross-references. Mixins that carry
richer runtime types (value objects, structured composites)
**decompose** them into named scalar fields and reconstruct on
read; the runtime API stays strict on the value-object type.

Why: the hydrator's bracket-assign is dumb — it copies whatever
shape comes out of MongoDB straight into the field via a setter.
For a Light value object that means the setter would have to accept
both a runtime `Light` AND the raw `{intensity, color}` plain
object, smushing two jobs (validation + coercion) into one
signature with a union type. Splitting storage into scalar fields
keeps each setter validating one primitive shape independently
(intensity is a number ≥ 0, color is a string-or-null), and the
runtime API stays strict on the value class.

Canonical example — `AmbientLitMixin`:

```ts
class AmbientLitMixin {
  static fieldMeta: FieldMeta = {
    ambientIntensity: { persistent: true },
    ambientColorTemperature: { persistent: true },
  };

  // Stored scalars — accessor pairs validate primitive shape:
  protected set ambientIntensity(v: number) {
    if (typeof v !== 'number' || !Number.isFinite(v) || v < 0) {
      throw new TypeError(/* … */);
    }
    this._ambientIntensity = v;
  }
  // ambientColorTemperature accepts numeric Kelvin or a tag string
  // (resolved through KELVIN_TAGS via Quantity.parse(s, 'K')).
  protected set ambientColorTemperature(v: number | string | null) {
    /* normalize to numeric Kelvin or null; throw on bad shape */
  }

  // Runtime API — strict on Quantity value objects, reconstructed
  // from the stored scalars on read:
  getAmbientFlux(): Quantity<'lumen'> {
    return Quantity.of(this._ambientIntensity, 'lumen');
  }
  setAmbientFlux(value: Quantity<'lumen'> | number | string): void { ... }

  getAmbientColorTemperature(): Quantity<'K'> | null { ... }
  setAmbientColorTemperature(value: Quantity<'K'> | string | null): void { ... }
}
```

Same shape for `LightSourceMixin` (`emittedIntensity`,
`emittedColorTemperature`) and for `Window`'s directional override
pair (`aToBOverride`, `bToAOverride` — the structured
`{aToB?, bToA?}` shape lives only at the runtime API layer).

For value-class fields like `Quantity<U>`, the typed setter shape
is achievable two ways: setter coercion (the AmbientLit pattern
above) OR a per-field Marshaller. The marshaller route keeps the
runtime setter strict on `Quantity<U>` and pushes shape coercion
to the persistence boundary — see § Marshaller Framework below
and [quantities.md § Persistence](./quantities.md#persistence).

When NOT to flatten: a small number of fields genuinely don't
decompose into named scalars — variable-key maps
(`Record<currency, amount>`), structured composites whose internal
structure IS the data, future composite types we can't anticipate.
That's what the Marshaller framework below is for.

Full rule + counter-examples:
[antipatterns.md § "Persistent Fields Default to Scalars; Marshallers Are the Escape Hatch"](../antipatterns.md#persistent-fields-default-to-scalars-marshallers-are-the-escape-hatch).

## Marshaller Framework

For the rare field whose storage shape genuinely doesn't decompose
into scalars, authors write a `Marshaller` (`lib/persistence/Marshaller.ts`):

```ts
abstract class Marshaller<TRuntime, TStored> extends Idea {
  abstract fromStored(stored: TStored): TRuntime;
  abstract toStored(runtime: TRuntime): TStored;
}
```

A Marshaller is an Idea-shaped Stuff (singleton-resolved by
templatePath, mirroring `PersistentHydrator`'s shape) so
content-author marshallers participate in `HotReloadApi` and can
hot-swap without restarting the server. Stateless by contract;
`fromStored` and `toStored` are pure functions of their input.

### Wire-up

The mixin declares a `marshaller` entry in `fieldMeta` mapping the persistent
field name to the marshaller's templatePath:

```ts
class WalletMixin {
  static fieldMeta: FieldMeta = {
    wallet: {
      persistent: true,
      marshaller: '/lib/persistence/MoneyBagMarshaller',
    },
  };

  // Setter is STRICT on the runtime type — by the time the
  // hydrator's bracket-assign fires, the marshaller's
  // `fromStored` has already produced a runtime MoneyBag.
  setWallet(value: MoneyBag): void {
    if (!(value instanceof MoneyBag)) throw new TypeError(/* … */);
    this._wallet = value;
  }
}
```

`MixinApi.getAllFieldMarshallers(constructor)` walks the prototype
chain (same shape as `getAllPersistentFields`) collecting the maps,
with subclass declarations winning over base for the same field
key.

### Resolution

`PersistentHydrator.hydrate` resolves marshallers via
`StuffApi.singleton(path)` — lazy. The async path mirrors how
`StuffApi.clone` resolves `hydratorClass`: returns the cached
instance if registered, or clones from the seeded template on first
need. No bootstrap manifest entry is required for marshallers; they
self-organize when first used.

`Document.save` / `findById` / `find` pre-resolve any registered
field marshallers (`preloadFieldMarshallers` instance method /
exported `preloadFieldMarshallersFor` static counterpart used by
`findById`/`find` and by `Template._materialize`) before the sync
`toDocument` / `fromDocument` walk. Pre-warming the singleton
cache lets the sync `findByTemplatePath` lookup inside those
methods always hit a populated cache. The async barrier sits in
the already-async `save`/`findById`/`find` boundary; `toDocument`
/ `fromDocument` keep their sync contract.

The self-persistence spine's `PersistableLogic.capture` uses the same
lazy-create pattern: it pre-warms every marshaller referenced by the host
and its deep contents (`Document.preloadFieldMarshallersFor`) BEFORE the
synchronous capture snapshot, so the sync `findByTemplatePath` lookup during
marshalling always hits a populated cache. This avoids requiring a bootstrap
manifest entry per marshaller while keeping `toStored` calls themselves sync.

In production, the marshaller's template row is installed into the
`content` collection from the platform pack at boot; the first save /
hydrate that needs it triggers `singleton(path)` to clone the
template. In tests there's no Mongo to clone from, so tests
register marshallers in-memory before use — see
`lib/persistence/__tests__/quantity-marshaller-test-helpers.ts`
for the v1 quantity-marshaller install helper, or
`registerMarshallerForTest` in
`lib/security/__tests__/test-setup.ts` for the lower-level
primitive.

### `QuantityMarshaller` — the production user

The first real production marshaller, shipping alongside the
[Quantity substrate](./quantities.md). Round-trips
`Quantity<U>` value objects through the storage shape
`{ value, unit }`, with `fromStored` liberally accepting numeric
/ string / JSON shapes for authoring ergonomics.

One class, parameterized by target unit at the instance level
(`unit` is a persistent field on the marshaller itself). Each unit
gets its own templatePath — call
`QuantityMarshaller.pathFor(unit)` rather than hardcoding the
encoded form (composite units encode `'/'` → `'-per-'`).

```ts
class Material extends ... {
  static fieldMeta: FieldMeta = {
    density:   { persistent: true, marshaller: QuantityMarshaller.pathFor('kg/m³') },
    molarMass: { persistent: true, marshaller: QuantityMarshaller.pathFor('g/mol') },
  };
  // Strict accessor pair on Quantity<U>; the marshaller
  // absorbed coercion at the persistence boundary.
}
```

Today's adopters: `Material.density` (kg/m³),
`Material.molarMass` (g/mol), and `TangibleMixin.mass` (kg).

### Marshalled props — `PropertiedMixin`

PropertiedMixin's `savedProps` is a heterogeneous record with
runtime-only keys, so the per-field-marshaller pattern doesn't fit
directly. Instead, PropertiedMixin grows a sibling persistent
field, `savedPropMarshallers: Record<string, string>`, that maps
prop name → marshaller templatePath. The binding lands via
`initProp`:

```ts
const mass = Property.of<Quantity<'kg'>>('mass');
avatar.initProp(mass, {
  transient: false,
  marshaller: QuantityMarshaller.pathFor('kg'),
});
avatar.setProp(mass, Quantity.of(5, 'kg'));
avatar.getProp(mass);  // → Quantity.of(5, 'kg')
```

`setProp` applies `marshaller.toStored` before writing to
`savedProps`; `getProp` applies `marshaller.fromStored` on read.
Storage carries the canonical `{value, unit}` shape; runtime callers
see the strict `Quantity<U>` instance. The binding persists alongside
the value, so reload-after-restart re-applies it without
redeclaration.

`removeProp` clears both the value and the marshaller binding.
`configureProp`'s transient↔persistent flip routes through
`fromStored` / `toStored` as appropriate so values move across the
boundary correctly.

### Don't reach for it as a first move

For fields whose runtime shape is a primitive or a primitive tuple,
flatten — the scalar-default rule above. The marshaller framework
is for the rare field whose storage shape genuinely doesn't
decompose: variable-key maps (`Record<currency, amount>`) or
specialized value objects with a canonical wire shape
(`Quantity<U>`'s `{value, unit}`). Most fields decompose; flatten
them.

## The self-persistence spine (`Persistable`)

The **persistence spine** is the universal substrate by which a `Stuff`
serializes **its own** runtime state — its declared fields, its held
contents, its worn gear — so that property, inventory, and room contents
survive residency eviction, logout, and reload, and reassemble faithfully on
materialize. It generalizes the Avatar-only `snapshotToTemplate` mechanism
(above) into one per-mixin-composed capture/restore model shared by any
persistence *host*.

The governing constraint is **security**: hydration bypasses the `setFoo()`
call-security gates, so the spine routes capture/restore **through** the
gated setter surface and reconstitutes items **through** the gated
`StuffApi.clone` path — never raw field injection — executed **as the owning
principal**. Persistence adds no new attack surface: it is exactly as safe as
the setter gates it routes through. See
[call-security.md](./call-security.md) and [access.md](./access.md).

### Hosts, content, and the record

Persistence is a property of **hosts** — singletons keyed by `templatePath`
(an avatar, an authored home/room, a unique container). A host composes
`PersistableMixin` (`lib/persistence/Persistable.ts`, `_mixinName =
'PersistableMixin'`), **outermost**, and carries three behaviors: a
`postRegister` **materialize driver** (with a record → restore; without →
capture the first record, the seed-then-persist gate), an `applyProps`
override that skips the seed once a record exists (no duplication), and a
`cleanupOnDestruct` **capture-on-destruct backstop**.

The engine-of-record is `PersistedRecord`
(`lib/persistence/PersistedRecord.ts`) over the **`holder_snapshots`**
collection — a `{ scope, owner, state }` envelope:

- `scope` — the host's singleton `templatePath` (identity + re-clone base;
  materialize loads every record scoped to it).
- `owner` — whose content (a principal's durable `templatePath`, a
  `group:<name>` sentinel, an `organization:<path>` sentinel, or — for
  untitled ground — the scope itself) → the **account-deletion cascade
  key**. Derived from `ParcelApi.ownerOf(scope)` — never a parameter.
- `state` — the **per-mixin-composed** capture: a map keyed by mixin/layer
  name to that layer's `MixinSlice` (`lib/persistence/PersistenceSlice.ts`).
  Stored as opaque JSON (no marshaller — `Document.toDocument` copies it
  as-is, the `StoredDocument.data` precedent); only `PersistableLogic` builds
  or interprets it.

There is **no player-facing write path**: `holder_snapshots` is written only
by the gated `PersistableLogic`, reachable only through `PersistableApi`'s
gated methods (`capture` / `materialize` / `restoreOrSeed` / `hasRecord` /
`deleteAllFor`).

### Per-mixin composition

Capture is **composed per-mixin**, mirroring `getAllPersistentFields`:
`MixinApi.getPersistenceContributors(ctor)` walks the prototype chain (own
`_mixinName` only — so a concrete subclass doesn't borrow its base mixin's
slice key) and yields one descriptor per contributing layer. Each layer
either runs its `captureSlice` hook or contributes the **default slice** (its
own declared `fieldMeta`'s persistent entries, marshalled to stored form). Three slice
shapes:

- **default** (`{ fields }`) — every ordinary mixin (`Graded`, `Propertied`,
  `Named`, …).
- **container** (`{ contents }`, `ContainerMixin.captureSlice`) — one
  `ContentEntry` per Containable, in `getContents()` order. A **non-host
  item** nests `{ templatePath, state, placement }` (recursing through
  sub-containers); a **nested host** is a reference
  `{ ref, key?, placement }` — not absorbed, because it persists itself
  (see *Keyed nested hosts* below). Surface-resting items record the index
  of the Surfaced sibling they rest on. Three occupants are **skipped**
  (one filter, one ordering shared with the Slotted slice): a live avatar
  (`HasInteractive` — persists itself), a player-stamped good
  (`ChattelApi.isOwnerPersisted` — persists in its owner's estate), and a
  `Behaved` NPC (**cast, never content** — see *The third skip* below).
- **slotted** (`{ worn }`, `SlottedMixin.captureSlice`) — worn/equipped
  occupancy by **position** (indices into the container slice) + the slot
  names each item claims; non-content occupants (a rider, a sitter) resolve
  to −1 and are skipped.

The recursion seam (`CaptureContext`) lets a mixin's `captureSlice` descend
into item state without importing `PersistableLogic` (breaking the lib →
obj/api cycle). Container/Slotted **restore** is centralized in
`PersistableLogic` (it cross-references the two slices by index).

### The security path

`PersistableApi` → `PersistableLogic` (`/platform/idea/api/persistable`, gated
`FromModule('/api/persistable#PersistableApi')`) owns the walk. Restore
composes three defenses:

1. **Principal frame** — the whole restore runs inside a pushed frame whose
   acting author is the record's owner (a live player when online, else the
   host itself — resolved decision #3), so `getActingAuthor` and any
   principal-based gate resolve to it, and restore is isolated from the
   ambient frame. This is the spine's single reviewed **frame-mutator
   allowlist** entry in `execution-context.ts` (the
   `SchedulerRegistry`/`EventSubscriptions` precedent).
2. **Drift guard** — a default slice's fields are filtered to the cloned
   class's declared `fieldMeta`'s persistent entries before hydration, so a forged record
   cannot inject `class`/`hydratorClass`/`brain` (Template-level, never
   persistent fields) nor any undeclared key. Fields hydrate through the
   standard two-phase `PersistentHydrator` (prefers the invariant-enforcing
   `set<Field>` setter, un-marshals rich values, bracket-assigns only
   setterless pure-storage fields).
3. **Gated reconstitution** — each `ContentEntry` is re-created via
   `StuffApi.clone(templatePath)` (the record names only a path + declared
   slices, never a class), then has its own `state` applied recursively;
   `{ref}` entries follow the reference — cloning the nested host, which
   self-materializes its own records via `postRegister`, reconstructing the
   tree by walking references.

Capture snapshots synchronously (after a marshaller pre-warm) so concurrent
triggers each write a valid full snapshot (last-write-wins). **The snapshot
is a copy, not a view:** an object-valued persistent field (a `reserves`
record, a `details` map, a `keywords` array) is detached with a JSON
round-trip on the way out, because storing it by reference would let a
mutation between the capture and the save silently rewrite the snapshot that
the sync-block invariant promises is frozen. A persistent field is about to
be BSON-serialized, so it holds plain data by construction.

### Keyed nested hosts

A nested host is captured as a `{ ref }` and restored by following the
reference. Originally that walk assumed **the nested host is a singleton** —
it resolved the ref through `findByTemplatePath`, returning the single live
instance if one existed — and `PersistableLogic` said so in its own comment.
That held while the only nested hosts were singleton fixtures.

It stops holding the moment a host is **multi-instance**: two peace lilies
cloned from one plant template, nested in one room, would collapse into one
on restore. So a ref entry now carries the nested host's per-instance key
when it has one:

| entry | restore |
|---|---|
| `{ ref }` | the singleton walk, unchanged — live instance at the path, else clone once and materialize keylessly |
| `{ ref, key }` | clone a **fresh** shell, stamp the key, materialize its `(scope, key)` record |

**Keys carry provenance, and that is what picks the branch.**
`Persistable.setPersistenceKey(key, explicit = true)` records whether an
*establishing context* supplied a real per-instance key (a leased dorm
room's unit parcel, a cultivated plant's uuid) or whether
`PersistableLogic` merely stashed the **scope-derived owner** it resolves on
a keyless capture. Only an explicitly-keyed host emits `{ ref, key }` —
otherwise a singleton's derived owner would send its own restore down the
keyed branch and clone a duplicate of a host that is already live.
`isPersistenceKeyExplicit()` is sticky once true.

**Backward compatibility is a hard requirement** and the shape delivers it:
`key` is optional and simply **absent** (not null) on an unkeyed ref, so
every record written before the change is byte-identical and restores
unchanged, and keyless nested hosts (`DormRoom`, `ConsignmentShelf`) show no
behaviour change.

Two related rules fall out:

- **A nested host captures `place: null`.** It is placed by its *referrer* —
  the ancestor whose container slice holds its ref — so its own `place`
  would fight that, and could phantom-clone a stale container template on
  restore. A `HasInteractive` host (an avatar) is exempt, because the
  container slice filters it out, so nothing refs it and its own `place`
  stays load-bearing.
- **The record is authoritative over born-with content.** A non-host content
  item is restored by re-cloning its template, and a clone re-runs the
  template's `props:` — so a container declaring born-with contents (a
  stocked chest, a pre-planted pot) arrived holding a *fresh* set that then
  collided with the recorded set: doubled contents, and a `Slotted`
  re-occupy that threw because the born-with occupant had already claimed
  the slot. `restoreItem` now clears what the clone seeded before applying
  what the record says, scoped by the record (contents only when the state
  carries a container slice, slots only when it carries a slotted one). A
  *host* never had this problem —
  `PersistableMixin.applyProps` only retains the specs and
  `seedBornWith` runs on the no-record branch alone — so this is the same
  rule for the non-host case.

The first consumer is the houseplant (each cultivated plant is its own keyed
host, carrying its own location), and **pets and livestock are the same
shape** — see [husbandry.md](./husbandry.md).

### `captureHostOf` — the mutating-act capture

Capture is **event-driven, not periodic**: autosave is Avatar-only.
Shutdown is the one exception and the backstop — a stop between two
residency sweeps would lose everything consigned or placed since (the
libations live drive watched a dev restart empty the cash-and-carry
counter), so `AppBootstrap.shutdown()` captures each persistable host
best-effort, alongside the world clock.

⭐ **It ASKS rather than remembering.** The hosts come from
`world:[mixin.PersistableMixin]` in system mode — the sanctioned world
search ([mql.md](./mql.md); it is what `lint:world-scan` points a
bespoke `getAllObjects()` loop at) — and each answers
`capturesAtShutdown()` for itself. That predicate holds the exclusions,
because who wants a shutdown capture is this mixin's knowledge, not the
bootstrapper's: a host with no persistence key never established one, and
an **Avatar** says no because it captures at logout on its own seam and
the sweep would only race it.

⚠ A `PersistableRegistry` briefly sat here, with hosts enrolling on
`setPersistenceKey` and withdrawing on destruct. It was deleted: a third
index of Stuff beside `byId` and `byTemplatePath`, caching a fact every
member already held — and its one consumer re-derived all of it on read
(`isPersistable`, a null key, `isDestroyed`). *A cache whose reader
revalidates everything it caches is buying nothing*, and this one was
maintained on every key-set and every destruct, forever, to save a single
sweep at process exit.

Reconcile-on-read state survives a rolled-back checkpoint by
re-deriving elapsed time from its clock stamp — but it cannot re-derive an
**intervention**, so an act that changes a host's state has to capture it.

`PersistableApi.captureHostOf(stuff)` is that call: capture `stuff` itself
when it is a host (a watered plant), else walk outward to the nearest
persistable containment ancestor (the dorm room a chest sits in) and capture
that under its own stashed key. A clean no-op when no host is found — the
thing lives in transient space and owns no record — and hop-capped against a
containment cycle.

### The keyed-holder ground pattern — `restoreOrSeed`

`PersistableApi.restoreOrSeed(host, key)` is the decision every
**multi-instance holder** makes when it stands one of its instances up:

```ts
host.setPersistenceKey(key);
return (await hasRecord(scope, key))
  ? (await materialize(host, key), true)   // re-entry
  : (await host.seedBornWith(), await capture(host, key), false); // first time
```

The scope is the host's own `templatePath`, derived exactly as `capture`
and `materialize` derive it, so a caller supplies only the key. It returns
`true` on a restore and `false` on a fresh seed, which lets a holder tell a
first provisioning from a re-entry and wire exits, announce or bill
accordingly.

Two consumers: **`DormWarren.admit`** (per leased unit) and
**`PlatWarren`** (per titled lot — see
[smallholding.md](./smallholding.md)). It lives here rather than in either
because hand-rolled, the same six lines invite three specific mistakes:
capturing on the restore path, re-seeding a room that already has
contents, and skipping the key stash so the next *keyless* re-capture
writes a second record. A non-persistable host throws — a call-site
programming error, not a user path.

### The eviction seam

A plain `ContainerMixin` vetoes eviction while it holds contents; a
**persistable host does not** — `ContainerMixin.canEvict` falls through for a
persistable host so its *other* vetoes still apply (a live Avatar's
`HasInteractive`, a `WarrenMember`) but the contents-count veto is dropped.
Durability is guaranteed at the sweep: `ResidencyLogic.runEvictionSweep`
(now async) **awaits `PersistableApi.capture` before `StuffApi.destruct`** for
a persistable host — rooms/chests have no autosave backstop, so a dropped
eviction write would be silent data loss. The sync `cleanupOnDestruct`
capture is the non-sweep backstop. Only a hard crash mid-session (before any
capture) loses since-last-capture changes.

### Avatar on the spine + host self-placement

Avatar persists through the universal spine — the per-player-template
`snapshotToTemplate` path is **retired**. Avatar composes `PersistableMixin`
outermost; its record carries its declared fields, its carried inventory
(`Container` slice), its worn gear (`Slotted` slice — new; gear was lost on
logout before), and its **own spawn/recall location** (`place`). `Avatar.save`
→ `PersistableApi.capture`, `Avatar.restore` → `materialize`. `postRegister`
orders the two paths differently: a **fresh signup** runs the born-with
loadout first, then `capture`; a **returning login** runs `materialize`
first, then re-runs the loadout **on top** — the snapshot carries the worn
cranial implant (Container + Slotted slices), so loadout-first would
double-occupy the slot, while the aether-hosted apps (comms / forums / the
credential wallet) are deliberately session-scoped and *not* in the
snapshot, so they must re-provision every login (the loadout's own
occupancy + hosted-update guards make the re-run idempotent). Two restore
subtleties the ordering implies: `materialize` preloads the organism's
anatomy between field hydrate and Slotted occupancy (the slots are defined
by the just-restored species' body plan, and no loadout has run yet), and
`restorePlacement` materializes (`singletonOrClone`) a captured container
that isn't live yet — rooms are lazy after a restart.

Three generic substrate capabilities support it:

- **`PersistedRecord.place`** — a Containable *top-level* host's own durable
  location (`{startLocation}` or `{container}`), captured with the
  `WarrenMember` reconciliation the retired `snapshotToTemplate` once did
  (the Warren, not the transient room clone), and restored via
  `ContainmentApi.resolveLanding` / `move`, overriding the clone-time template
  spawn. Null for a room (not Containable) or a host placed by its referrer.
- **Self-owner** — a `HasInteractive` host owns its own record
  (`owner = scope`), so the account-deletion cascade `deleteAllFor(<avatar
  path>)` is a keyed match; and `ContainerMixin.captureSlice` **skips**
  `HasInteractive` occupants (a live avatar is never a room's content).
- **`shouldPersist()`** — a per-instance opt-out (an Avatar returns
  `!isGuest`), consulted by capture / materialize / `postRegister` /
  `cleanupOnDestruct`, so a guest writes and restores nothing.

Only `snapshotToTemplate` (the Avatar-only snapshot direction) was retired;
**`restoreFromTemplate` stays** — `CmsLogic` and `PackLogic` use it to
re-hydrate live clones from an edited template (content go-live, a distinct
concern).

### Deferred

Possession (per-owner *loose* items in a shared room) and the
compute-allowance persistence cap — property Phase 0b/1.

**Multi-instance persistable hosts** (many live instances of one template,
each keyed by an explicit per-instance key) — **shipped (D1)**: the
`capture/materialize/hasRecord` `key` param and the host key stash, with the
**uniform `(scope, key)` identity** — a host is `(scope = templatePath, key)`,
the key resolved explicit → stashed → scope-derived, so a singleton is the
degenerate case (key from scope) and a keyed room is the general one, with
**no `multiInstance` marker/mode**. The single invariant `assertUniqueKey`
(no two live instances share a `(scope, key)`) replaced the eager
singleton-scope guard; `applyProps` is a uniform no-op (holders seed
imperatively via their context); `postRegister` no longer auto-drives. First
consumer: the leased dorm room (keyed on its unit parcel). See
[residence.md](./residence.md).

## Design Decisions

### Why static `collectionName`?

Each class declares its own collection name to:

- Avoid passing collection names to methods (error-prone).
- Make collection names discoverable from the class definition.
- Enable type-safe CRUD without per-call configuration.

### Why explicit `fieldMeta`'s persistent entries?

While we could scan all properties, explicit declaration:

- Excludes computed/runtime properties from persistence.
- Documents the persistence contract at the class boundary.
- Prevents accidental data leaks (a field added later isn't silently
  persisted).
- Is the lookup mechanism the standard `PersistentHydrator` uses to
  decide which keys to copy from `data`.

### Why not decorators?

A decorator-based approach was considered. The base-class approach was
chosen for:

- No build configuration overhead.
- Simpler debugging (standard inheritance chain).
- Works cleanly under TypeScript strict mode without
  `experimentalDecorators` quirks for this surface.

(The codebase DOES use decorators elsewhere — `@Final`, `@CallSecurity`,
etc. for the security framework. Persistence didn't need that machinery.)

### Why is `Document` NOT a Stuff?

An earlier design (`Persistable extends Idea`) folded every persisted
record into the Stuff hierarchy, to avoid "two parallel hierarchies."
But that made the *rare* case (a live world entity) the default and
forced the *common* case (plain document data — auth records, and soon
dialogue trees, loot tables, lesson content) to pay proxy + registry +
security-gate + lifecycle overhead it never uses. Most of what a
platform persists is plain data with no game-entity behavior.

The fix isn't to make everything Stuff, nor to duplicate the
persistence machinery — it's to **extract** the shared field-mapping
machinery (`MixinApi.getAllPersistentFields` and friends are already
constructor-static and neutral) and let *both* a plain `Document` base
and the Stuff `Hydrator` consume it. One persistence story, two object
models. A `Document` is value-like persisted state; a `Stuff` is an
identity-like live entity hydrated from a Document. `Document` losing
the per-object security gate is correct — document access control binds
at the Api/collection/lease layer, not per-object. (An audit confirmed
nothing relied on `User`/`GoogleProfile`/`Template` being Stuff.)

The one Stuff-coupling that remains is marshallers + hooks +
`PersistentHydrator`, which stay Idea-rooted *for HMR only* (they're
stateless strategy objects hot-swapped via the clone pipeline).
Un-Stuffing them — re-homing them as path-resolved code modules — is a
deferred, separate change; `Document` reaches them through the injected
resolver seam in the meantime.

## `PersistApi` — the mudlib's only route

Every mudlib read and write goes through `PersistApi` (`mud/api/persist.ts`).
`lib/persistence/Document` and `lib/stuff/Template` used to hold the last
framework carve-out — sanctioned on the grounds that they *are* the data
layer — and now route through the facade like everything else. The
[import boundary](../architecture.md) is what closed it: a mudlib module
may not import `backend/` under any justification, and PersistApi's
surface already covered every call the two made. `check-pm-access.ts`
dropped both allowlist entries; only the facade itself and
`api/hot-reload` (PM *lifecycle*, not data) may still reach
`PersistenceManager.get()`.

The same face owns **encrypt-at-rest**: `PersistApi.sealString` /
`unsealString` (AES-256-GCM under `TOKEN_ENC_KEY`, fresh IV per value,
`EncryptedEnvelope` stored shape). `crypto` is outside `src/mud/`, so the
cipher lives in the Api tier and `EncryptedStringMarshaller` is the
field-level adapter over it — it keeps the envelope validation and the
`Marshaller` shape, and holds no key. The key is memoised process-wide
and lazily loaded; `_resetEncryptionKeyForTest` invalidates it.

## Collections, and the schema docs that describe them

**Every collection has exactly one authored YAML doc** at
`packages/server/src/schema/<collection>.yaml`. That doc is where the
collection is described — what it is for, what is true of every row, why
each index exists — and it is the source the three machine-readable
tables are generated from.

Before this existed the four facts about a collection lived in four
TypeScript files (the name, the sandbox policy, the reset disposition,
the indexes), what the collection was *for* lived almost nowhere (4 of 48
had any prose), and 11 record classes named their collection with a bare
string literal so the vocabulary could not see them.

⭐ **The motivation is pedagogy, not tidiness.** Moving four tables into
YAML is the means; `help bank_ledger` is the end.

### The shape

```yaml
collection: bank_ledger
owner: LedgerEntry                    # the Document class, or `none`
ownerModule: /lib/banking/LedgerEntry # where that class lives
subsystem: banking.md                 # the doc that owns the concept
summary: >-
  The money system of record. Every movement of money, append-only.
purpose: |
  Double-entry, append-only, and the only thing in the system that is
  allowed to say how much money exists. …
invariants:
  - >-
    Only `BankingApi.postTransaction` writes here — the sealed
    chokepoint, and the reason conservation holds.
sandbox: stamp                        # or a mapping: {verb: pass, mark: true}
reset: wipe                           # or {verb: keep, because: …}
indexes:
  - keys: { fromAccount: 1 }
    why: |
      The per-account replay and every outbound read — one side of the
      double entry.
```

`mud/lib/persistence/SchemaDoc.ts` is the parser, and **all three readers
go through it**, so a malformed doc fails the same way everywhere. It is
a pure value object — no `fs`, no YAML parser; the caller hands it an
already-parsed object, which is what keeps it import-clean under
[the import boundary](../architecture.md).

⚠ It deliberately does **not** check that `collection` is a member of
`Collections`: the enum is generated from these docs, so a membership
check there would be the tail wagging the dog. Set equivalence is
asserted by the loader (at boot) and by `pnpm lint:schema` (at build).

### The three readers

| reader | what it does with them |
|---|---|
| `backend/PersistenceManager` | `loadSchemaDocs()` at `connect()`, then builds every index from them |
| `scripts/gen-schema.ts` | emits `Collections.ts`, `CollectionPolicy.ts`, `ResetPolicy.ts` |
| `platform/idea/HelpCatalogue` | `projectCollections()` — one help topic per doc |

### ⚠ The generated files are never hand-edited

`Collections.ts`, `CollectionPolicy.ts` and `ResetPolicy.ts` each open
with a do-not-edit banner. Edit the YAML doc for the collection you mean
and re-run `pnpm gen:schema`; `pnpm lint:schema` fails if the emitted
files and the docs disagree.

The module-level TSDoc of each emitted file is a template constant in
`scripts/gen-schema.ts` — **that** is its editable copy. Everything below
the module comment (every enum member, every table row, every `because`)
comes from the docs.

**Why generate rather than parse at runtime.** `Collections` is used in
TYPE position — `Record<Collections, …>`, `PersistApi.find(Collections.X,
…)` across ~50 files. Parsing the vocabulary at boot would trade a whole
class of compile-time error for a boot-time one. Generating keeps both:
one authored source, and a typo is still a build failure. It is the
lint-family pattern applied to data.

### Indexes are data; two loops stay derived

`createIndexes()` was 570 lines of hand-written `createIndex` calls. It
is now a driver over three sources:

| kind | count | where it comes from |
|---|---|---|
| **Authored** | 84 | the `indexes[]` of each schema doc |
| **Text** | 3 of those | the same, with `text: true` — routed through `ensureTextIndex`, whose drop-and-recreate-on-conflict recovery is BEHAVIOUR and stays in PM |
| **Derived** | two loops | the `circleScope` partial on every STAMP collection (5 today), and the `{ kind, data.<naturalKey> }` partial-unique per declared document kind (5 today) |

Both derived loops are **consequences of another declaration**, not
authored facts, and writing them out per collection would be the
duplication the docs refuse. Their COUNTS move on their own when the
vocabulary they walk changes, which is exactly why they are not a list.

⭐ Declaring `sandbox: stamp` in a doc is now what gives a collection its
`circleScope` index: `STAMP_COLLECTIONS` derives from
`COLLECTION_POLICIES`, which is generated from the docs. There is no list.

⭐⭐ **Every index is attempted independently.** The predecessor wrapped
the whole list in one `try`, so the first failure skipped every index
below it — found by driving, when a stale text index on `player_frames`
silently took the wiki index, the forum index and every sandbox partial
with it. Per-index isolation was called "a large mechanical rewrite"
while the calls were written by hand; over a loop it is free.

`plannedIndexes()` is the pure half — what this deployment declares,
computable without a connection, and what a test asserts against.

### ⚠⚠ A missing doc is an error, not a default

`loadSchemaDocs()` throws at boot, naming every undescribed collection
(not just the first). A doc naming an unknown collection throws too. A
collection nobody described is precisely the state these docs exist to
end, so the runtime refuses to paper over it — `lint:schema` catches it
long before a boot does, and the runtime check is what makes the gate
load-bearing rather than advisory.

### `pnpm lint:schema` — the six assertions

CI-gating (wired in `.gitlab-ci.yml`'s lint job), **no exemption list**:

1. Every doc names a collection; every collection has exactly one doc;
   neither set has an extra.
2. Regenerating produces byte-identical generated files.
3. Every `static collectionName` outside `__tests__` is `Collections.X`,
   never a string literal.
4. Every `owner` names a real `Document` subclass whose `collectionName`
   is that collection, and `ownerModule` names exactly the file that
   class is declared in. `owner: none` is legal and means *nothing but
   `PersistApi` writes here*, which must then be true.
5. Every `subsystem` resolves to a real file under `docs/subsystems/`.
6. Every doc has a non-empty, non-placeholder `summary` and `purpose`.

Test-fixture classes (`boxes`, `widgets`, `test_wallets`) are exempt from
(3) by living under `__tests__`; they name collections that are not in
the vocabulary and must not be. That is the only carve-out.

⚠ Resolution for (3) and (4) is AST-based and **file-scoped** — the
`lint:topics` lesson: a tree-wide name table silently resolves a name
against an unrelated file, and the gate then passes while the thing it
watches is broken.

### `ownerModule` is not a second copy of `owner`

The field list in a collection's help topic is **harvested** from
`Owner.fieldMeta`, never restated in the YAML — a doc carrying a field
list would be two copies of one sentence, and the copy that drifts is the
one nobody executes. Harvesting needs the class OBJECT rather than its
name, and `StuffApi.resolveClassFile` is the one place a class path
becomes a file. So the doc names the path, the projector resolves it, and
assertion 4 proves the path names exactly the file the class is declared
in — a class that moves fails the build.

### What a schema doc does NOT carry

- **A field list** — harvested from `fieldMeta`, above.
- **Per-field prose** — it has no home until `fieldMeta` grows a
  `description`. That is the obvious next move and is explicitly not
  attempted here; the projector's field harvest would render it with no
  schema-doc change.
- **A history block** — git is the record.
- **Schema *validation*** — these docs describe; they do not enforce
  document shape at write time. Mongo-side JSON Schema validators are a
  separate build with a real migration question attached.

They are also **not a content pack** (a pack cannot create a collection,
and the set is closed and repo-owned) and **not seed files** (they seed
no data). They are repo files loaded at boot, exactly as `hooks.yaml` is.

### Proving a change is safe

`scripts/dump-indexes.ts --reindex --out <file>` drops every non-`_id`
index, reconnects so the driver rebuilds from scratch, and dumps
`listIndexes()` for all 48 collections. Run it before and after any
change to index declaration and diff. When indexes moved into the docs
the diff was empty: **139 `listIndexes()` rows across 48 collections** (45
of them the automatic `_id_`), byte-identical before and after.

⚠ Point it at this worktree's own `saxonberg_buildN`. Rebuilding every
index on a live database is not free.

## Cross-References

- [templates.md](./templates.md) — the other persistence track:
  clone/hydrate/save-template pipeline for game-world objects, including
  `TemplateApi.saveTemplate` and the folder/leaf-invariant `DomainHook`
  that rides on the around-hook mechanism above.
- [lifecycle.md](./lifecycle.md) — Stuff create/destroy lifecycle
  (applies to the cloned Stuff a Template produces, not to the
  `Document` itself).
- [antipatterns.md § Per-Field Invariants](../antipatterns.md#per-field-invariants-belong-on-setters-not-in-normalize-hooks)
  — setter contract that hydration rides on.
- [antipatterns.md § Persistent Fields Default to Scalars](../antipatterns.md#persistent-fields-default-to-scalars-marshallers-are-the-escape-hatch)
  — full statement of the scalar-default rule and the marshaller
  escape hatch, with BAD/GOOD examples.
- [state-model.md](./state-model.md) — the `Document` track, Avatar
  self-contained model, why Player class is gone.
- [light.md](./light.md) — first major user of the scalar-default
  rule; `AmbientLitMixin` / `LightSourceMixin` / `Window`
  decompose Light value objects into scalar fields.
- [quantities.md](./quantities.md) — first production user of the
  Marshaller framework. `QuantityMarshaller` round-trips
  `Quantity<U>` through `{value, unit}` JSON; PropertiedMixin's
  per-prop marshaller binding (`savedPropMarshallers`) lets host
  Stuff store rich Quantity props without a per-class declaration.

## History

- **The self-persistence spine + Avatar migration** (persistence-spine
  build). The universal `PersistableMixin`/`PersistableApi`/`PersistableLogic`
  substrate landed, and Avatar migrated off the per-player-template
  `snapshotToTemplate` persist-back onto it — so carried inventory and worn
  gear now survive logout (they were lost before; only `fieldMeta`'s persistent entries
  persisted). The load-bearing shifts: the durable-location capture (the
  `WarrenMember` recall reconciliation) moved from `TemplateLogic.snapshotToTemplate`
  onto `PersistableLogic.capturePlacement` (`PersistedRecord.place`);
  `snapshotToTemplate` was retired while **`restoreFromTemplate` stayed**
  (`CmsLogic`/`PackLogic` content go-live depend on it — the plan's original
  "retire the pair" was corrected). A late review pass typed the
  `shouldPersist` hook (off duck-typing) and added the singleton-host
  runtime guard.

- **Keyed nested hosts + three snapshot-fidelity fixes** (houseplant build,
  living-world phase 1 — `1dfbe2df..0936da6e`). The spine's stated invariant
  was that *"a nested host reached by the `{ref}` walk is a singleton (unique
  templatePath)"*; the first multi-instance nested host (a cultivated plant,
  one of many from one template) broke it, so ref entries gained an optional
  per-instance `key` and keys gained **provenance** (`setPersistenceKey(key,
  explicit)` / `isPersistenceKeyExplicit()`) so a singleton's scope-derived
  owner cannot send its own restore down the keyed branch. Keyless records are
  byte-identical to the pre-key shape — backward compatibility was a hard
  requirement, and there is live data.

  Building the durability proofs surfaced **three pre-existing bugs**, each
  fixed here rather than worked around: `captureFields` stored object-valued
  fields **by reference** (so a mutation between capture and save rewrote the
  snapshot the sync-block invariant promises is frozen — now detached, and
  promoted to an [antipatterns.md](../antipatterns.md) entry because the trap
  generalizes past persistence); a restore-cloned **non-host container
  re-ran its `props:`**, doubling contents and throwing on the `Slotted`
  re-occupy (the record is authoritative — `restoreItem` now clears what the
  clone seeded, the same rule `seedBornWith` already gives hosts); and a
  nested host captured its own `place`, fighting the referrer that actually
  positions it. `PersistableApi.captureHostOf` landed as the mutating-act
  capture the whole husbandry family reuses. See
  [husbandry.md](./husbandry.md).


## History — the furnishing build (2026-07-31)

The **second persistence scope** landed: owned chattel persists with its
*owner*, not with the room it stands in. See
[furnishing.md](./furnishing.md) for the whole model; what changed here:

- **A fourth slice type.** `EstateSlice` joins `FieldsSlice` /
  `ContainerSlice` / `SlottedSlice` in the `MixinSlice` union.
- **`restoreSlice` is now invoked.** It was declared on
  `PersistenceContributor` and collected by `getPersistenceContributors`
  but never called — `Container` and `Slotted` have explicit passes. It is
  now dispatched as step (4) of `restoreState`, after the slotted pass, so
  any layer can carry an async restore hook as originally intended.
- **`ContainerMixin.captureSlice` skips a second kind of content** — a good
  stamped to a *player*, whose own estate record carries it
  (`ChattelApi.isOwnerPersisted`). A good stamped to an *organization* has
  no estate to ride and stays in its host's record (the libations build:
  a business's consigned counter, its bought rail). Both skips share ONE
  filter, because the Container and Slotted slices read one content
  ordering.
- **Capture reports what it skipped.** `CaptureContext.noteOwnedGood`, and
  `captureImpl` flushes each skipped good into its owner's estate after the
  synchronous state build. Without it, a good in a room going dormant while
  its owner is offline would be captured by nobody and destroyed with the
  room.
- **New Api surface:** `captureDetached` / `restoreDetached` (one non-host
  good's composed state) and `placeIdOf` (a host's room identity — scope
  plus its per-instance key **only when explicit**).

## History — the residences build (2026-08-31)

The spine's keyed-host model was already here (the dorm's D1
multi-instance change); the residential ladder made two things follow
from it, and added one marker.

### A placement can name a KEYED container

`HostPlacement` gained `containerKey?`. A holding's rooms all share one
template row and are separated only by their persistence key, so
`{ container }` alone would collapse every tenant's yard into one; and
the room is not standing when the placement restores, because the whole
holding is dormant. So `capturePlacement` records the pair, and
`restorePlacement` re-enters through `OuterWarren.admitFor(key)` —
which finds the institution whose parent extent prefixes the key, stands
the holding back up, and returns the exact room.

That is the **"log out in your own yard, log back into the same yard"**
acceptance, and it is the general form of the dorm's Warren
reconciliation rather than a second mechanism beside it.

### An owned good can be MOUNTED

`EstateEntry` gained `mounted?: { slot }`. A good hung on a wall lives in
its room's `Adornable` fixture map rather than its contents; the marker
is **derived** from the good's own live attachment (`getMountSlot()`) at
capture time rather than passed in, so it can never disagree with the
map, and the room overlay re-attaches instead of floor-placing.

⚠ **`AdornableMixin` needed a capture pass of its own.** The container
slice never sees a fixture, so a stamped good on a wall was reported by
nobody: a room going dormant while its owner was offline took the lamp
with it. The mixin now walks `getFixtures()` and reports stamped ones to
`noteOwnedGood`, exactly as the container slice reports stamped contents.
Its slice is deliberately empty — fixtures are runtime-only and an
authored one is rebuilt from the row on every hydrate; the pass exists
for the REPORT.

### `_identityStampOf` peels, it does not unwrap once

The D17 identity split reads a hard-private slot to key the registry
index. One `ProxyApi.unwrap` of a proxy-of-a-proxy yields the inner
PROXY, which carries no private slot and throws — which turned a
deliberate `postRegister` failure into the wrong error on the unregister
path. It now peels until the slot is present, which is correct at any
wrapping depth.

## Props and cast — the two born-with designations (2026-09-01)

**Cast is CAST, never a room's content.** A `Behaved` NPC commutes between
persistable rooms — a hand's floor and the counter it consigns at — so each
room's capture caught it wherever it happened to stand, BOTH records could
carry it, and the next boot restored it twice: `expected singleton, found
2`, boot dead. Verified live in the farming build (2026-08-31); the
designation became **declared** the day after. A template authors its
born-with content in the theatre vocabulary:

- **`props:`** — the set dressing (fixtures, stock, furniture). Ordinary
  content: captured into the host's record and written back; the record is
  authoritative after the first seed.
- **`cast:`** — the troupe (`Behaved` NPCs). Never captured, conserved
  live, re-seeded on restore.

The class is the check, not the designation: each `PopulatesMixin` applier
resolves an entry's template class and gates on `Mixins.Behaved` **before**
minting — a Behaved entry under `props:` (or a non-Behaved one under
`cast:`) is an authoring error at hydrate, not a latent lifecycle bug.
Two touches carry the lifecycle:

- **Capture skips `Behaved` occupants** — the third skip in
  `ContainerMixin.captureSlice`'s single filter (beside `HasInteractive`
  and owner-persisted goods; the three must share one filter because the
  Container and Slotted slices read one content ordering). This keys on the
  MIXIN, not the authored list — a wandering NPC standing here at capture
  time is some other room's cast, and equally not our content. Capture is
  the only record writer, so no record can carry a cast entry (the old
  restore-side sniff was migration compat and is deleted).
- **`Persistable.reseedCast()`** — the establishing half: after a
  successful restore, `materializeImpl` walks the host's retained
  `_bornWithCast` (the `cast:` data the hydration hook keeps) and re-mints
  any entry with **zero live instances**, moving it in. Live cast is
  conserved — an instance standing anywhere (mid-commute at the counter)
  suppresses the re-mint. No class resolution: the declaration already
  said what is cast.

The troupe's durable home is the authored `cast:` data, exactly like the
born-with seed; only its *liveness* is transient. Proven in
`lib/persistence/__tests__/CastReseed.test.ts` and the designation gate in
`lib/stuff/__tests__/Populates.test.ts`.
