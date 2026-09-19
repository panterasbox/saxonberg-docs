# Templates

Game-world objects in Saxonberg are not constructed directly. They are
**cloned** from templates stored in the MongoDB `domain` collection. The
clone pipeline owns construction, hydration, registration, zone resolution,
proxy wrapping, and a post-registration hook — the same plumbing every
game-world object goes through.

This is one of two persistence tracks. Auth/meta records (User,
GoogleProfile) extend `Document` and live as plain MongoDB documents;
those are covered in [persistence.md](./persistence.md). Templates cover
everything in the game world.

## The Template Class

Templates are modelled as a `Document` subclass — like `User` and
`GoogleProfile`, a Template is a record (plain persisted data, **not** a
Stuff), not a game-world entity. It is the data a game-world Stuff is
*cloned from*, never a live entity itself. `Template` itself is
**abstract**; concrete subclasses (`ZoneTemplate`, `LeafTemplate`) are
returned by `Template.findByPath` based on
`ZoneApi.isFolderClass(doc.class)` — a structural check
(`prototype instanceof Zone`) rather than a central allow-list. The
base lives at `lib/stuff/Template.ts`:

```typescript
abstract class Template extends Document {
  static collectionName = 'domain';
  static fieldMeta: FieldMeta = {
    path: { persistent: true },
    class: { persistent: true },
    hydratorClass: { persistent: true },
    data: { persistent: true },
  };

  path: string = '';
  class: string = '';
  hydratorClass?: string;
  data: Record<string, unknown> = {};

  static findByPath(path: string): Promise<Template | null>;
  static findByPaths(paths: readonly string[]): Promise<Template[]>;
  static findDescendants(basePath: string): Promise<Template[]>;
  static loadById(id: string): Promise<Template | null>;
  static ancestorPaths(path: string): string[];
}

class ZoneTemplate extends Template {} // type-level marker; folders.
class LeafTemplate extends Template {} // type-level marker; leaves.
```

`findByPath`, `findDescendants`, and `loadById` all dispatch into the
right subclass — callers that hold a `Template` get back the correct
shape without needing to sniff `class`.

⭐ **`findByPath` does not go to the database.** The `content`
collection is held resident by `PersistenceManager` and a by-path read
is answered from memory, hit or miss — which is what makes the clone
pipeline's per-clone template load, and the zone walk's read per
ancestor path, free rather than a ~30 ms round trip each. The cache is
kept current at the same chokepoint every write passes through; see
[persistence.md § The resident `content` cache](./persistence.md). The
other three still query. The split is the primary
expression of the folder/leaf invariant; the `DomainHook` that fires
on save/delete is defense-in-depth at the persistence chokepoint.

`Document.findById<T>` is still available on the concrete subclasses
(`LeafTemplate.findById(id)` works); on the abstract base it's a
compile-time error because `new Template()` isn't legal — that's by
design and is the reason `loadById` is a separate method on
`Template`. `_materialize` constructs the chosen subclass with a plain
`new` (a Template is a `Document`, not a registered Stuff).

CRUD goes through the inherited `Document` surface
(`save`/`delete`/`findById`/`find`) plus the helpers above. See
[persistence.md](./persistence.md) for the `Document` contract.

- `path` is the canonical identifier — clones happen by path. Folder/leaf
  invariants (below) constrain paths.
- `class` names the runtime backing class to instantiate. Resolved by
  dynamic import; validated against an allow-list (below).
- `hydratorClass` is opt-in. **When absent, no hydrator runs and `data` is
  ignored.** Templates that want generic mixin-field copy must explicitly
  set `hydratorClass: '/platform/idea/persistence/PersistentHydrator'` (the standard
  implementation). Custom hydrators are also class paths under `/lib/`.
- `data` is pure hydration payload — never carries class paths itself.

The two class fields are independent. A single hydrator can serve many
backing classes (a `CreatureHydrator` for both `Guard` and `GuardDog`).
A single backing class can be paired with many hydrators (different
domain-specific hydrators per template family).

## Class Path Validation

`StuffApi.#validateClassPath(classPath)` gates every dynamic import:

- Must start with `/`
- Must NOT contain `..` (no directory traversal)
- Must start with one of the allowed prefixes: `/platform/… + /stuff/` or `/lib/`

Class names are the last path segment (e.g., `/platform/agent/Avatar` →
`Avatar`); the import succeeds only if a named export with that name
exists in the resolved module. Anything else throws.

This validation is **format-only** (shape of the path). The orthogonal
**trust** question — *may this author name this code at all?* — is
enforced separately at the `saveTemplate` chokepoint: a non-wizard
(protowizard) author cannot introduce or change the executable
code-naming fields (`class` / `hydratorClass` / `behaviors[].brain`).
See [access.md § The code-trust lockdown](./access.md).

## The Clone Pipeline

`StuffApi.clone<T>(templatePath, context?): Promise<T>` runs:

1. **Load template** by path from `Collections.Domain`. Throws if missing.
2. **Validate `class` path**, dynamic-import the module, fish out the
   constructor by name.
3. **Resolve zone** via `ZoneApi.resolveZoneForPath(templatePath)` — walks
   ancestor paths nearest-first, returns the Zone clone of the first
   ancestor whose template names a Zone class. Returns `null` when the
   template is itself a Zone, or when no ancestor is a Zone. Stamped onto
   the instance before hydrate so anything that reads `this.zone` during
   hydrate sees the right value.
4. **Resolve `hydratorClass`** if present. Hydrators are themselves
   templated `Idea` Stuff. Because they're stateless by contract,
   `clone` resolves them via `StuffApi.singleton` — one cached
   instance per hydrator class, reused across every backing. The
   first clone that needs a particular hydrator triggers a recursive
   `clone` (HMR-aware via the same path as the backing class) to
   warm the cache; subsequent clones reuse the singleton. Recursion
   terminates because hydrator Templates name no `hydratorClass` of
   their own. Cycles (a hydrator transitively naming itself) are
   caught by `clone`'s in-flight-path guard and surfaced as
   `circular template dependency`. See
   [hot-reload.md § Hydrators](./hot-reload.md#hydrators).
5. **Construct** the backing under the construction sentinel:
   ```typescript
   Stuff._beginConstruction();
   try { obj = new ClassConstructor(); }
   finally { Stuff._endConstruction(); }
   ```
   The sentinel is a runtime gate — every Stuff constructor throws unless
   the sentinel is set. This is what enforces "only StuffApi may
   construct Stuff." See [lifecycle.md](./lifecycle.md) for details.
6. **Stamp `templatePath`** onto the instance so identity-keyed security
   policies (`FromTemplate`) can match against it.
7. **Wrap in a Proxy** via `ProxyApi.wrap(raw)`. Every consumer that
   resolves the object by `stuffId` thereafter sees the proxy, not the
   raw instance — the security gate is then in the call path for those
   callers.
8. **Register** the proxy in `StuffApi`'s `objectsById` map.
9. **Hydrate**: if `hydratorClass` was named, run
   `await hydrator.hydrate(proxy, template.data ?? {})` inside a synthetic
   constructor frame (`ExecutionContextApi.run` with
   `FrameKind.Constructor`).
10. **postRegister**: if the backing composes `PostRegistrationMixin`,
    `await proxy.postRegister(context)` — same synthetic frame.

If hydrate or `postRegister` throws, the object is unregistered before the
error propagates. Half-initialised objects never linger in the registry.

Order is load-bearing. **Register fires before hydrate** so that anything
resolving the in-flight object by `stuffId` during hydrate or
`postRegister` (e.g., a self-referencing exit hydrator) finds it.

## ⭐ Instruction appliers run ONCE — `props`/`cast` are initial furnishing

⚠ **Added 2026-08-30** (the libations review). `TemplateApi.restoreFromTemplate`
— the **CMS save go-live** (`CmsLogic`) and the **pack reconcile go-live**
(`PackLogic`) — re-runs the *full* `hydrate`, which re-dispatches every
Phase-2 instruction applier. For a *value* field that is harmless: the
field is re-assigned to the authored value, which is the point. For an
applier that **mints objects** it is a faucet: before this,
publishing an edit to a `props:` row minted a fresh set into every
live instance — every crate in the world gaining six more grapefruits,
every non-singleton fixture in a plain room duplicated.

`PopulatesMixin` therefore records `_propsPopulated`/`_castPopulated`
(one once-flag per list — `props:` the write-back set dressing, `cast:`
the conserved troupe; see persistence.md § Props and cast) and each
applier no-ops once its flag is set.
Singletons were already safe (the applier skips one already placed);
this covers the plain-clone branch. `PersistableMixin` hosts (every
`FurnishableRoom`) were never exposed — that override only *retains*
the specs and lets the establishing context seed them once via
`seedBornWith` — and the guard now makes that idempotent too.

⚠ **Deliberately not count-aware.** "Top up to the declared list" was
rejected: the same mechanism serves a room's fixtures, which are meant
to be permanent, and a crate's contents, which are meant to be
**consumed**. "Declared minus present" would resurrect goods somebody
drank, ate or sold — the faucet the libations build spent its whole
length removing from the bar. An author who edits the list and wants it
applied **re-clones**; there are no migrations.

**The general rule for a new instruction field: if the applier creates
Stuff, it must be idempotent, because go-live will call it again.**

## The Hydrator Contract

`Hydrator` (`lib/stuff/Hydrator.ts`) is a one-method interface:

```typescript
interface Hydrator {
  hydrate(backing: Stuff, data: Record<string, unknown>): Promise<void>;
}
```

Hydrators are stateless. One instance hydrates many backings. They
don't mirror-compose the backing's mixin chain — they introspect the
backing directly. That's why a single hydrator class can serve multiple
backing classes.

The standard `PersistentHydrator` (`lib/persistence/PersistentHydrator.ts`)
runs a **two-phase dispatch**:

- **Phase 1 — property fields.** For each entry in
  `MixinApi.getAllPersistentFields(backing.constructor)`, the hydrator
  prefers `await target.set<PascalCase(field)>(value)` when that
  method exists, and falls back to `target[field] = value`
  bracket-assign otherwise. The async-first dispatch lets setters
  with side effects (e.g.,
  `CartesianLocation.setCoords` registers with the zone via
  `addLocation`) complete their work before the next field is
  processed. The bracket-assign fallback still fires an accessor
  pair when one is defined on the prototype — runtime-shape
  validation declared as a `set` accessor continues to gate the
  write.
- **Phase 2 — instruction fields.** For each entry in
  `MixinApi.getAllInstructionFields(backing.constructor)`, the
  hydrator calls `await target.apply<PascalCase(field)>(value)`. The
  applier is **required** — absence of `applyX` is a configuration
  bug surfaced as a clear runtime error. No bracket-assign fallback
  for instruction fields. Phase 2 runs sequentially after Phase 1
  so all property fields settle before any instruction is applied.

**Bracket-assign (the Phase 1 fallback) IS still part of the contract
surface.** It invokes accessor pairs when present. So if a field has
a shape invariant ("must be boolean", "lowercase / trim / dedupe"),
put the rule on the setter accessor and hydration routes through it
for free even when no `setX` method exists. Don't add
`normalize()`-style post-hydrate fixups for per-field rules — see
[antipatterns.md § Per-Field Invariants](../antipatterns.md#per-field-invariants-belong-on-setters-not-in-normalize-hooks).

Cross-field invariants ("if `isLocked` is true, `lockKey` must reference
a real key") can't live on a single setter — that's the legitimate use
case for a custom `Hydrator` subclass. Override `hydrate()`; call
`super.hydrate()` first if you want the default two-phase dispatch
too.

### Property vs instruction fields

Two distinct field shapes ride on the Hydrator's two-phase dispatch
(per `feedback_property_vs_instruction_fields`):

- **Property fields** — declared `{ persistent: true }` in `fieldMeta`.
  Data IS the field's value: `setX(v); assert(getX() === v)`.
  Symmetric on shape. Side effects on the setter (e.g., `setCoords`
  registers with the zone) don't change the field's identity. Shape:
  optional backing slot `_x`, optional accessor pair
  `get x / set x` for shape invariants, public methods
  `getX()` / `setX()` for the inter-Stuff contract. Hydrator
  dispatch: Phase 1 — `setX` if defined, else bracket-assign.
- **Instruction fields** — declared `{ instruction: true }` in `fieldMeta`
  (new sibling to `fieldMeta`'s persistent entries). The data is a **recipe**
  consumed to produce or modify *separately-named* runtime state —
  no "value" to set/get on the spec, only a verb's argument. The
  canonical example is `exits` on `ExitableMixin`: the YAML data is
  a `Record<string, ExitInstruction>` recipe, `applyExits` consumes it to
  populate the runtime `exits: Map<string, Exit>` collection (which
  has its own established `getExit` / `addExit` / `removeExit`
  surface). No paired getter for the spec. Hydrator dispatch:
  Phase 2 — `applyX` required.

  Other appliers shipped by the substrate:
  - `applyContainer` on `ContainableMixin` — resolves a templatePath
    via `StuffApi.singleton` and moves self into it via
    `ContainmentApi.move`. Compare-and-move idempotency: no-op when
    the current container's templatePath matches the declared path.
    Target must be singleton-shaped (validated at template-save
    time by `TemplateApi.validateSingletonContainerTarget`).
  - `applyProps` on `PopulatesMixin` — iterates a list of
    entries, each either a bare templatePath or a
    `{template, onto}` object. Dispatches per-entry by
    source-template singleton-shape: singletons resolved via
    `StuffApi.singleton` (skip-when-already-elsewhere);
    non-singletons cloned via `StuffApi.clone`. A bare entry is
    moved into self (`ContainmentApi.move`); an `{template, onto}`
    entry is placed on an already-populated sibling surface
    (`ContainmentApi.placeOn`), keyed by the `onto` source path —
    so the surface fixture must be listed before its resting
    items (the back-bar before its bottles). See
    [crafting.md](./crafting.md).

The two shapes split apart cleanly: properties have symmetric shape
and storage IS the value; instructions are commands whose outcome
lives elsewhere. Trying to use `set/get` for instructions produces
shape asymmetry (setter takes specs, getter returns runtime state)
— "marshaller work in the wrong place." Recognizing them as separate
concepts dissolves that.

The constant `PersistentHydrator.templatePath` is the single source of
truth for the standard hydrator's path; use it at call sites instead of
duplicating the literal:

```typescript
TemplateApi.saveTemplate(path, classPath, data, PersistentHydrator.templatePath);
```

## The Context Bag

`StuffApi.clone(path, context?)` accepts an opaque `context` bag that
gets threaded through to `postRegister`. It exists to carry runtime
setup that cannot come from the template — typically references to other
runtime objects.

The canonical example is Avatar:

```typescript
// Avatar.ts
export interface AvatarInitContext {
  user?: User;
  playerId?: string;
}

class Avatar extends AvatarBase {
  override async postRegister(context?: unknown): Promise<void> {
    const ctx = context as AvatarInitContext | undefined;
    this.user = ctx?.user;
    // ...
  }
}

// Application.ts
const avatar = await StuffApi.clone<Avatar>(
  Avatar.getTemplatePath(playerId),
  { user, playerId }
);
```

Subclasses narrow the context to a concrete type locally. The clone path
itself stays generic — no type parameter on `StuffApi.clone`.

## PostRegistrationMixin

`PostRegistrationMixin` (`lib/stuff/PostRegistration.ts`) is the opt-in
post-registration hook. Spring `@PostConstruct` semantics:
`postRegister(context?)` runs **after** registration, so any resolver
that walks the registry sees the in-flight instance.

Composition is the marker. The clone pipeline checks
`MixinApi.isPostRegistration(proxy)` and only awaits the hook when the
backing composes the mixin. The default implementation is a no-op;
subclasses override.

This replaced the older `'initialize' in obj && typeof init === 'function'`
duck-typing check that used to live in `StuffApi`.

## TemplateApi & the Folder/Leaf Invariant

`TemplateApi.saveTemplate(path, classPath, data, hydratorClassPath?)` is
the typed convenience wrapper for writing a template:

```typescript
await TemplateApi.saveTemplate(
  '/narnia/castle/foyer',
  '/lib/location/CartesianLocation',
  { /* persistent fields */ },
  PersistentHydrator.templatePath
);
```

It looks up an existing `_id` for upsert semantics and delegates to
`PersistenceManager.save(Collections.Domain, doc)`.

The **folder/leaf invariant** (Phase 7 Decision 12) constrains the
`domain` collection paths:

- **Folders** = Zone-class templates. Detected structurally by
  `ZoneApi.isFolderClass(classPath)` — a class is a folder iff its
  `prototype instanceof Zone`. Spatial Zones (`CartesianZone`,
  `SphericalZone`) AND non-spatial Zones (`Clade` — taxonomic) all
  qualify. Folders MAY have descendant templates.
- **Leaves** = any non-folder template. Must NOT have descendant
  templates.

The folder check (`ZoneApi.isFolderClass`) is a strict superset of
the spatial-zone check (`ZoneApi.isSpatialZoneClass`) — the latter
is `prototype instanceof SpatialZone`, the set of classes whose
templates stamp `Stuff.zone` via `ZoneApi.resolveZoneForPath`.
Non-spatial folders (Clades) are folders for the invariant but
**never** become a `Stuff.zone`. The structural-check approach
means content devs add new folder or spatial-zone classes by
extending the right base — no central allow-list to edit. See
[spatial.md § Zones](./spatial.md) and
[race.md § Clade](./race.md#clade--taxonomic-scope).

The rule is enforced by `DomainHook` (`obj/hooks/DomainHook.ts`), which
composes `AroundSaveHookMixin` and `AroundDeleteHookMixin` and registers
against `Collections.Domain`. The hook calls
`TemplateApi.validateFolderLeafSave` / `validateFolderLeafDelete`, which
reject:

1. Path doesn't start with `/`.
2. Doc shape isn't a template (missing `path` or `class`).
3. Leaf save with existing children.
4. Save under a non-Zone ancestor — "Ancestor `A` is a leaf template,
   not a zone folder."
5. Delete of a Zone with surviving descendants.

The validation fires at the PM chokepoint, so calling
`PM.save(Collections.Domain, doc)` directly is equivalent to
`TemplateApi.saveTemplate` — both go through the hook.

`hydratorClass` is orthogonal to zonehood. Zone classification uses the
runtime `class` field only.

## Avatar Template Convention

```typescript
class Avatar {
  static readonly TEMPLATE_PATH_PREFIX = '/platform/agent/Avatar/';

  static getTemplatePath(playerId: string): string {
    return `${this.TEMPLATE_PATH_PREFIX}${playerId}`;
  }
}
```

Avatar templates are stored at `/platform/agent/Avatar/<playerId>` and created
automatically when a Player is added to a User. Cloning happens at user
connect (see `Application.handleUserConnect`).

The per-player Avatar template carries the **initial** (char-gen) state at
first clone. It is no longer written back to on save: Avatar persists through
the **self-persistence spine** into its own `holder_snapshots` record (fields
+ inventory + gear + spawn location), NOT the template — see
[persistence.md § The self-persistence spine](./persistence.md#the-self-persistence-spine-persistable).
The template is authoritative only on first-ever login (no record yet), then
the record is (seed-then-persist).

## Restore-from-template (content go-live)

`TemplateApi.restoreFromTemplate(stuff)` re-hydrates a live Stuff from its
backing Template's `data` — looks up the Template by the host's
runtime-stamped `getTemplatePath()` and runs
`PersistentHydrator.hydrate(host, tpl.data)`. Operates on the existing live
instance; preserves identity / stuffId / wired Interactives. Phase 2 appliers
re-fire (e.g., `applyContainer` moves the host via compare-and-move).

Its consumers are the **content go-live** paths: `CmsLogic` and `PackLogic`
re-hydrate live clones after an author edits a template. It is NOT a
persistence-of-runtime-state mechanism — that is the spine.

The **snapshot** direction (`snapshotToTemplate`) was **retired** with the
Avatar migration onto the spine; its capture behavior (per-mixin field
marshalling, the `WarrenMember`-reconciled location capture, the
sync-prefix-before-first-await ordering) now lives in `PersistableLogic` and
is covered by `lib/persistence/__tests__/persistence-spine.test.ts`.

## `create` and `createSync` (Sister APIs)

`StuffApi.clone()` is the production path. Two sister APIs cover cases
where templates aren't right:

- **`StuffApi.create(factory, context?)` (async)** — caller-supplied
  factory, no template lookup, no hydration step. Same register +
  `postRegister` tail. Used for runtime-only objects whose construction
  needs explicit arguments and don't round-trip through the CMS pattern.
  `Interactive` is the canonical example: `socketId`, `sessionId`, `user`
  all flow through the closure, not a template.

- **`StuffApi.createSync(factory)` (sync)** — same sentinel-flip + Proxy
  wrap + register guarantees as `create`, but no hydrate step and no
  `postRegister` await. **Throws if the constructed Stuff composes
  `PostRegistrationMixin`** — silently skipping `postRegister` would
  yield a half-initialised object. Used inside sync helpers where
  awaiting would force the caller (and its callers) to become async too;
  `Exitable.addBidirectionalExit`'s `new Exit(...)` calls are the typical
  trigger.

Reach for `create()` whenever async hydration or post-registration
matters; `createSync()` is the narrow-use sister.

## `singleton(path)` and the Clone Pre-Flight

`StuffApi.singleton<T>(templatePath, context?)` returns the existing
instance for a path when one is already loaded; otherwise it routes
through `clone()`. The lookup hits the `byTemplatePath: Map<string,
Set<Stuff>>` index that `register` / `unregister` keep up to date —
see [lifecycle.md](./lifecycle.md#what-registration-actually-does).
A non-empty bucket with more than one entry throws (the caller mixed
`clone()` and `singleton()` on a class that does NOT compose
`SingletonMixin`).

`SingletonMixin` (`lib/stuff/Singleton.ts`) is a marker mixin (no
public surface; see
[mixins.md § Marker mixins](./mixins.md#marker-mixins-empty-public-surface)).
Composing it opts a class into a clone-time pre-flight: `clone()`
checks `byTemplatePath` first and throws if any instance already
exists for that path. The pre-flight depends on `unregister`'s
empty-bucket cleanup running before the next clone — which it does,
because `Stuff.destroy()` synchronously calls `unregister`.

Use `singleton()` for shared-state Stuff (the starting room, the
EventRegistry, well-known service objects); use `clone()` for
per-instance Stuff (avatars, items, NPCs) that should multiply.

## Failure Modes

- Template not found → `Error("Template not found: ${path}")`
- Class path validation fails → `Error("Class path must…")`
- Dynamic import fails → `Error("Failed to import class…")`
- Class name not exported by module → `Error("Class … not found in module…")`
- Hydrator path invalid / import fails → analogous "Failed to import
  hydrator…" / "Hydrator … not found…"
- Hydrate or `postRegister` throws → object is unregistered, then the
  original error propagates
- `createSync` on a `PostRegistrationMixin` class → throws before
  registration

## Deferred / known limitations

The declarative-content substrate (the `container:` field, `applyProps`,
the structural field shapes) is shipped, but a few constraints are known and
left for future work:

- **Reset / respawn shares substrate with `props:` (own slate).** A
  future "reset this game item — destroy the current one, respawn fresh"
  feature needs three things `applyProps` doesn't yet provide: (a) a
  runtime ref from the Container back to the children it spawned, so it can
  destroy-and-respawn later — the ref must evaporate on a child's `onDestruct`
  so an already-gone child doesn't haunt the list; (b) an ownership
  distinction between "I spawned this and own it for reset purposes" and "a
  player put this here, leave it alone on reset"; (c) a declarative reset
  cadence on the entry (a richer entry form like `{ path, resetCadence }`).
  `applyProps` v1 is fire-and-forget; it extends to track the ref and
  respect the policy when reset lands as its own slate. Flagged so the
  props entry shape doesn't get locked into something that fights reset
  later.

- **`container:` can't target a multi-room facade (Lounge case).** A
  `container:` value must resolve to a singleton-shaped Container (a canonical
  Stuff). That can't express "land in the Lounge" when the Lounge is
  conceptually singular but implemented as a dynamic multi-room assembly that
  grows and shrinks with occupancy — there's no single canonical Stuff to
  point at. A known Phase-2 limitation; the eventual fix (a facade Stuff that
  IS a Container and routes internally, a master/slave room, or Pattern-C
  resolve-at-clone-time) isn't pre-baked. Cross this bridge when the Lounge
  ships.

- **`container:` vs `props:` conflict detection (linter / boot
  validation).** The same Containable-belongs-in-Container relationship can be
  declared from either side — `container:` on the occupant or `props:` on
  the host. Consistent declarations coexist (first to fire moves the
  singleton, the second no-ops via the existing-container check). But
  contradictory declarations (X says "I belong in A", yet A's `props:`
  omits X while B's includes X) aren't caught today — they need linter-level
  or boot-time validation that walks the template collection and flags the
  conflict. Not required for runtime correctness (refs still resolve lazily);
  a dev/CI affordance if conflicting declarations become an operational pain.

## Cross-References

- [lifecycle.md](./lifecycle.md) — full create → register → hydrate →
  postRegister → destroy lifecycle, construction sentinel, onDestruct
  hook
- [persistence.md](./persistence.md) — `Document`, around-save/delete
  hooks (the mechanism `DomainHook` rides on)
- [call-security.md](./call-security.md) — `ProxyApi.wrap`,
  `ExecutionContextApi.run`, `FrameKind.Constructor`,
  `SecurityApi.decorateApiClass`
- [state-model.md](./state-model.md) — why game-world objects use
  templates and Avatar's "self-contained" design
- [antipatterns.md § Per-Field Invariants](../antipatterns.md#per-field-invariants-belong-on-setters-not-in-normalize-hooks)
  — setter contract for hydration
