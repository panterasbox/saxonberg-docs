# Hot Module Reload

Saxonberg expects running code to be edited and reloaded against a
live server, in the LP-MUD tradition. New clones of a reloaded
blueprint pick up the new behavior; existing instances keep the
prototype chain they were constructed against; old blueprints fall
out of the registry and become GC-only when their last instance is
gone.

The mechanism is a small piece of infrastructure (`HotReloadApi`)
plus a single integration point (`StuffApi.clone` consults it). Every
HMR-correctness story in the codebase flows from those two facts.

## Why this works in Saxonberg

The codebase is already free of `instanceof` checks for class
identity. Mixin detection goes through `_mixinName` markers via
`MixinApi.isX(obj)` / `MixinApi.hasMixin(ctor, Mixins.Foo)`; templates
clone by path; command controllers register by verb; hooks register
by name. Everything that has to survive a class swap is already
looked up by name. The HMR subsystem must not regress this property —
no new code path may introduce identity-based class checks.

## Surface

`HotReloadApi` lives at `packages/server/src/mud/api/hot-reload.ts`.

```ts
class HotReloadApi {
  // Mutators
  static reload(path: string): Promise<void>;     // load → swap registry
  static rollback(path: string): void;            // swap current ↔ previous
  static unload(path: string): void;              // clear + freeze

  // Introspection
  static getCurrent(path: string): Function | null;
  static getCurrentExport(path: string, exportName: string): Function | null;
  static getPrevious(path: string): Function | null;
  static getRegisteredPaths(): string[];
  static isFrozen(path: string): boolean;

  // Targeted invalidation
  static reloadHookManifest(): Promise<void>;
}
```

The path argument is always an absolute filesystem path — the same
shape `StuffApi.clone` resolves a `/platform/agent/Avatar`-style class path to.

## State machine

Three states per path. The registry holds at most two blueprints;
anything older falls out on the next successful reload.

| State | `current` | `previous` | `clone(path)` | `rollback(path)` |
|---|---|---|---|---|
| Empty | — | — | lazy-`reload()` (or throw if frozen) | throws "nothing to roll back to" |
| V1 | v1 | null | uses v1 | throws "nothing to roll back to" |
| V2 | latest | one prior | uses latest | swaps current ↔ previous |

Transitions:

- `reload(path)` from Empty → V1 on success; stays Empty on failure.
- `reload(path)` from V1/V2 → V2 on success; the prior `previous` (if
  any) drops out of the registry and becomes GC-only. Failure leaves
  the registry untouched.
- `rollback(path)` from V2 swaps current ↔ previous; still V2.
- `unload(path)` from any state → Empty + frozen.

### Empty has two flavors

`StuffApi.clone` distinguishes "never seen" from "explicitly
unloaded" via `#frozenPaths`:

- **Never seen:** `clone` calls `reload(path)` lazily and self-warms.
- **Frozen (post-`unload`):** `clone` throws "no blueprint at path"
  until the next `reload` repopulates and clears the frozen flag.

## Reload mechanism

`reload(path)` reads the source bytes, computes a `versionId` (sha256
truncated to 16 hex chars), then dynamic-imports the module with a
cache-busting query: `import(\`${url}?hmr=${counter++}\`)`. Each
query string produces a fresh module evaluation; the source transform
re-runs and `ModuleApi.stamp` registers the new class objects under
fresh `ModuleId` entries.

Concurrent `reload(path)` calls share one in-flight promise — the
second caller waits for the first instead of starting a parallel
import.

### TypeScript transpilation

The cache-busted `import()` does no compilation of its own — it
relies on whatever loader hook the host runtime has installed. In
dev that's `tsx` (the server runs under `tsx watch src/preload.js`,
which registers the call-security loader hook before dynamically
importing `src/index.ts`);
in tests it's Vitest's loader. Both transpile `.ts` on import,
including dynamic imports, so paths like `/abs/.../Foo.ts?hmr=N`
work without a separate `tsc` step or built artifact.

Prod is `tsx` too (`deployment.md § Runtime shape`: the server runs from
TypeScript source; `pnpm build` is a typecheck gate, not the runtime),
so there is no built-JS path to plan for: the same `.ts` resolution
holds in dev, test and prod. `reload <target>` accepts a **class path**
(`/platform/thing/Thing`, `/system/arcana/thing/Wand`) and resolves it through
`StuffApi.resolveClassFile` — the kernel tree, or a capability pack's
`src/` (content-packs.md § The capability rung) — before falling back to
the workspace-logical join; `pack sync <id>` reloads a pack's changed
`src/` files the same way before re-hydrating its rows.

## Integration

### `StuffApi.clone` (the only HMR-aware site)

`clone()` consults `HotReloadApi.getCurrentExport(absPath, className)`
before falling back to a bare dynamic import. Frozen paths throw "no
blueprint at path." This means **every Stuff that flows through
`clone()` automatically picks up HMR**, including:

- Hooks (cloned by `PersistenceManager.loadHooks` from `hooks.yaml`).
- Command controllers (cloned by `CommandGiver.executeController`
  from the `/platform/idea/cmd/*` Templates the platform pack installs).
- Hydrators (cloned by `clone()` itself from the `hydratorClass`
  field of the backing's Template — see Hydrators below).
- Anything else templated, including avatars, locations, ideas.

The fallback from `clone` to a bare dynamic import only fires when
the registry is Empty for a path — i.e., the path was statically
imported by something else first. After the first lazy `reload()` it
flows through the HMR registry.

### Api logic singletons (`/platform/idea/api/<feature>`)

`Api` classes are static and direct-imported, so they are **not**
reloadable (see "What's intentionally out of scope"). The
surface-architecture refactor keeps the `Api` as the stable, typed,
secured boundary but relocates its **logic** into a stateless `Stuff`
singleton at `/platform/idea/api/<feature>` (e.g. `MaterialLogic` at
`/platform/idea/api/material`, `LocomotionLogic` at `/platform/idea/api/locomotion`) — and
*that* is hot-reloadable. The Api's public statics forward to it.

The seam is `StuffApi.singletonSync(path, factory)` (a sync,
registry-keyed get-or-create — see
the StuffApi source and
its unit tests in `api/__tests__/singleton-sync.test.ts`). Each Api
file resolves its singleton through a `logic()` helper:

```ts
const LOGIC_PATH = '/platform/idea/api/material';
const LOGIC_CLASS_FILE = fileURLToPath(
  new URL('../platform/idea/api/MaterialLogic', import.meta.url)
);
function logic(): MaterialLogic {
  return StuffApi.singletonSync(
    LOGIC_PATH,
    () =>
      new ((HotReloadApi.getCurrentExport(LOGIC_CLASS_FILE, 'MaterialLogic')
        as typeof MaterialLogic | null) ?? MaterialLogic)()
  );
}
```

The logic class is a stateless `Stuff` (`extends Idea`, **no**
`PostRegistrationMixin`), marked `@internal` **on the `export class`
declaration** (a leading file-top comment would become TypeDoc's module
comment and fail to exclude the class), and each public method carries
its own `@CallSecurity(FromModule('/api/<feature>#<Feature>Api'))`
gate — per-method, not class-level. The gating recipe (why per-method,
the intra-singleton self-call gotcha, the `ApiOnly` widening, the
two-singleton state re-point) lives in
[call-security.md § The api↔logic-singleton recipe](./call-security.md#the-apilogic-singleton-recipe);
this doc covers only the reload mechanics.

Two facts make this HMR-correct:

- **The factory resolves the blueprint, not a bare `new`.** The
  `getCurrentExport(...) ?? StaticClass` line is load-bearing: after a
  reload it picks up the fresh class; a bare `new MaterialLogic()`
  would rebuild the stale one. The class-module path (for
  `getCurrentExport`) and the `/platform/idea/api/<feature>` stamp path (for
  addressing) are **distinct** — both appear in every conversion.
- **Reload is `dest`.** The singletons are stateless by construction
  (no `PostRegistrationMixin`), so destruction is free.
  `StuffApi.destruct` unregisters the singleton, emptying its
  `byTemplatePath` bucket; the next `logic()` call re-creates it
  through the factory against the current blueprint. No automatic
  invalidation routing — `dest` is the invalidator, same as for any
  `Stuff`. The stuffId is ephemeral (a new one per recreate); the
  **path is the stable handle**, MQL-addressable with no `Template`
  doc.
- **Lazy materialization — un-materialized singletons aren't
  enumerable.** A `/platform/idea/api/<feature>` singleton doesn't exist until its
  first `logic()` call; before that there's no live instance and no
  `Template` doc to fall back on, so it won't appear in a
  `findByPathGlob('/platform/idea/api/*')` sweep. Enumerating the *un*-materialized
  set would need a static path list (a mild registry) — **deferred until
  it bites**: glob-over-live covers every real case (you only care about
  singletons that have actually been used), and the no-premature-registry
  rule argues against standing one up speculatively. (Graduated from the
  retired surface-architecture slate.)

#### Demonstration (locomotion)

The dest→recreate-fresh-via-`getCurrentExport` cycle is unit-verified
end-to-end against a real `HotReloadApi.reload` of an on-disk fixture
in `api/__tests__/singleton-sync.test.ts`. The same cycle is observable
in-game on `locomotion` through the movement verbs:

1. `pnpm dev:server` and connect a client; confirm `walk west` (or any
   movement verb) behaves normally — the first `go` lazily materializes
   `LocomotionLogic` at `/platform/idea/api/locomotion`.
2. Edit a `LocomotionLogic` method body — e.g. make
   `defaultModeFor(actor)` return `'fly'`, or have `canTraverseExit`
   reject with a custom `reason`.
3. In-game (as a developer): `reload /platform/idea/api/locomotion` to load the
   new source, then `dest /platform/idea/api/locomotion` to drop the live
   singleton. (Reloading the *source* alone does not swap the live
   instance — the cache holds the old one until `dest`.)
4. Issue a movement verb again. The next `LocomotionApi` call
   re-creates the singleton through the factory, which resolves the
   freshly-reloaded `LocomotionLogic` class, and the movement verb
   exhibits the new behavior.

This is the [hot-reload caveat for hydrators](#hydrators) generalized:
the `byTemplatePath` index *is* the registry; `dest` + lazy re-create
is the reload.

### Hydrators

Hydrators are templated `Idea` Stuff. Unlike controllers, they are
stateless by contract (`Hydrator.ts` documents this), so the clone
pipeline resolves them via `StuffApi.singleton` — one cached
instance per hydrator class, reused across every backing it hydrates:

```ts
const hydrator = template.hydratorClass
  ? await StuffApi.singleton<Hydrator & Stuff>(template.hydratorClass)
  : null;
// ... construct backing ...
if (hydrator) await hydrator.hydrate(backing, data);
```

The first clone needing a particular hydrator lazy-clones it through
the (HMR-aware) `StuffApi.clone` path; subsequent clones find the
instance in the `byTemplatePath` index and reuse it.

**HMR for hydrators.** A reloaded hydrator class produces a new
blueprint in the registry, but the cached singleton instance is still
pinned to the old class. To pick up new behavior, destruct the cached
instance — the next clone that needs it will lazy-re-create through
`singleton(path) → clone(path) → HMR-aware class lookup`:

```ts
await HotReloadApi.reload('/abs/.../PersistentHydrator.ts');
const stale = StuffApi.findByTemplatePath('/platform/idea/persistence/PersistentHydrator');
if (stale) StuffApi.destruct(stale);
```

No dedicated helper — the byTemplatePath index is the registry.

**Cycle detection.** `clone()` keeps an `#inFlightClonePaths` set:
on entry it adds the templatePath, removes in `finally`, and throws
`circular template dependency` if the same path is already in
flight. This catches a hydrator template that (transitively)
references itself before the recursion stack-overflows. Hydrators are
the only realistic cycle vector in v1; the guard is at the clone()
level so any future template-resolution cycle (mod system, custom
loaders) gets the same treatment.

### Command dispatch

`CommandGiver.executeController` clones a fresh controller per
execution, runs it, and destructs in `finally`:

```ts
const controller = await StuffApi.clone(`/platform/idea/cmd/${command.controller}`);
try {
  return await controller.execute(fields, context);
} finally {
  StuffApi.destruct(controller);
}
```

Per-execution clone is the right semantic for stateless dispatchers —
it sidesteps state-leak between commands and gives HMR for free
(every clone goes through the override path). The destruct keeps
`StuffApi`'s indexes from accumulating one stuck controller per
command.

### Hooks

Hook instances are persistent — `PM.loadHooks` clones them once at
boot and registers them in `PM.saveHooks` / `PM.deleteHooks` arrays.
Reloading a hook source file leaves the cached instance pinned to
its old class; the in-place fix is `HotReloadApi.reloadHookManifest()`,
which calls `pm.clearHooks() + pm.loadHooks()` to rebuild the chain
against the current HMR blueprints.

### Activity framework

Activities are plain TS classes (not Stuff-templated), so they
don't pick up HMR through `StuffApi.clone`'s blueprint dance.
`SchedulerApi` (`api/scheduler.ts`) carries its own type-keyed
class registry (`#activityRegistry: Map<string, ActivityClass>`)
as the HMR seam.

The index populates by **capture-at-start** (the
no-module-scope-statements rule — activity files declare only):
`start(engagement)` records `engagement.constructor` under
`engagement.type`, and `SchedulerApi.reloadActivity(type)` re-points
the entry to the freshly-reloaded class after
`HotReloadApi.reload()` re-evaluates the file (the next `start` of
the type would also re-point it). Two consequences:

- **Newly-constructed engagements use the new class.** Callers
  resolve activity classes via `SchedulerApi.getActivityClass(type)`
  rather than static `import`, so a freshly-issued `walk west`
  after a reload uses the new class without reloading the
  controller too.
- **In-flight engagements pick up the new code on their next
  lifecycle fire.** `SchedulerApi`'s internal
  `#dispatchOnComplete` / `#dispatchOnAbort` / `#dispatchGetHost`
  helpers look up the class through `#activityRegistry` and call
  `cls.prototype.onComplete.call(engagement)` — bound to the
  instance, but resolved through the latest class.

`onStart` is the exception. It runs once at start time against
the just-constructed instance (which was constructed from the
class that was registered at that moment), so direct instance
dispatch is fine.

`SchedulerApi.reloadActivity(type)` rounds out the surface as a
wizard convenience: it resolves the activity class's source path
via `ModuleApi.lookup`, strips the `#exportName` suffix, and calls
`HotReloadApi.reload`. Intended for use via the `eval` verb until
a dedicated reload verb ships.

**Caveats** (accepted v1 limitations):

- **Emission closures pin to construction.** An engagement's
  `emissions` array was populated by the old constructor; closures
  inside captured old-class lexical scope. Editing emission
  factories has no effect on in-flight engagements. Code-only
  `onComplete` / `onAbort` body edits work cleanly because those
  go through the registry dispatch.
- **Field additions don't backfill old instances.** If new code
  reads `this.someNewField` that wasn't set by the old
  constructor, it gets `undefined`. Typical edits (changing
  behavior, not shape) are unaffected.
- **`SchedulerRegistry` internal edits still need a reboot.** The
  active engagements and their timers live on the `SchedulerRegistry`
  singleton; in-flight engagement state doesn't survive a reload of
  that Registry. The timers are `WorldClockApi` `ClockHandle`s
  (game-time schedules), not raw Node timers — ghost handles from
  old-class registrations fire into the new code with stale state.
  Same for `EngagedMixin` storage shape.
- **Registry miss recovers gracefully.** If a class is
  unregistered between `start()` and a lifecycle fire (file
  deleted, type renamed),
  `dispatchOnComplete` forces `onAbort('thrown')` so timers and
  slot state get cleared, and the engagement degrades cleanly.

**Future: drain-and-reload wizard verb.** Composes
`SchedulerApi.cancelByType(actor, type)` (per actor) +
`SchedulerApi.reloadActivity(type)` for the case where new code
can't safely resume from old engagement state. The infrastructure
exists in v1; the verb is a small future addition tied to the
wizard subsystem's `'hotreload'` `AbortReason` augmentation.

See [activity.md § The Scheduler — Lifecycle dispatch
(HMR-aware)](./activity.md) for the full producer-side framework.

### `ModuleApi`

Untouched. The existing source transform stamps reloaded class
objects normally via the cache-busted `?hmr=N` query. `ModuleApi`'s
WeakMap is keyed by class object, so each reload's distinct class
objects each get their own `ModuleId` entry; nothing collides.

### Composition validation

`MixinApi.assertComposable` (the runtime hook for mixin composition
rules like `Globbable ⊥ Container`) memoizes on **constructor
identity**, so HMR-produced class bindings re-validate
automatically — a reload that changes a class's identity gets a
fresh first-instance-of-class registration and a fresh check.

The subtle case: reloading the **mixin** alone is not enough.
`class Coin extends GlobbableMixin(Idea)` captures the mixin
function at evaluation time; reloading `Globbable.ts` produces a
new mixin function but doesn't rewire `Coin`'s prototype chain.
`Coin`'s constructor identity is unchanged, the WeakSet hit
short-circuits, and the new check never fires. To rotate the
validation, reload the **leaf** module too — that re-evaluates the
class expression against the new mixin and produces a fresh
`Coin` identity.

No cascade. Editing a mixin doesn't fan out to every dependent
leaf — that would re-instantiate live state while a player is
mid-action. The pattern when a dev does want to rotate dependents:
an MQL query (`world:[mixin.GlobbableMixin]`) to find the
population plus an explicit reload of each leaf class. Forgetting
this step is not a correctness bug — the old check keeps applying
and the new constraint silently doesn't tighten, but nothing
breaks.

See [mixins.md § Composition validation](./mixins.md#composition-validation-__validatecomposition__)
for the full mixin-side story.

## Lifecycle events

Four well-known events declared in `lib/events.ts`, all sharing the
`ReloadEvent` payload and all `emittableBy(HotReloadApi)`:

```ts
Events.ModuleReloaded     // 'module.reloaded'      — successful reload
Events.ModuleRolledBack   // 'module.rolledBack'    — rollback() succeeded
Events.ModuleUnloaded     // 'module.unloaded'      — unload(), incl. no-op-from-Empty
Events.ModuleReloadFailed // 'module.reloadFailed'  — reload threw

interface ReloadEvent {
  path: string;
  versionId: string | null;          // null on Unloaded
  previousVersionId: string | null;  // null when there was no prior
  exports: string[];                 // class export names; empty on Unloaded / Failed
  error?: { message: string; stack?: string };  // Failed only
}
```

`unload` from Empty is a no-op-with-event: subscribers see the intent
even when the registry was already empty.

`HotReloadApi` doesn't keep its own audit log. Anything that wants a
ledger (admin UI, mod-system observers, future code-audit collection)
subscribes via `EventApi.on(Events.Module*, listener)`.

## Failure handling

| Failure | What happens | Event |
|---|---|---|
| `path` doesn't exist / unreadable | `reload` throws, registry unchanged | `ModuleReloadFailed` |
| Compile error in new module | `reload` throws, registry unchanged | `ModuleReloadFailed` |
| Top-level execution error | `reload` throws, registry unchanged | `ModuleReloadFailed` |
| `@Final` violation in reloaded class | `reload` throws (caught from `ModuleApi.stamp`), registry unchanged | `ModuleReloadFailed` |
| Constructor throw on later clone | `clone` throws to its caller; registry stays on the new (broken) version. Caller may `rollback` and retry | (none — reload itself succeeded) |
| `rollback` from Empty / V1 | throws | (none) |
| `unload` from Empty | no-op, but emits intent | `ModuleUnloaded` |

## Memory growth across many reloads

Each successful reload creates new class objects under fresh
`?hmr=N` URLs. Old class objects are reachable from existing
instances via their prototype chain and from `ModuleApi`'s WeakMap.
They go away when both go away. In long dev sessions with frequent
reloads, accumulated stale class bindings grow until those refs are
released. Bounded in practice; not a v1 concern. If it ever becomes
one, the fix is bookkeeping the per-path `ModuleId` set so older
entries can be deliberately released.

## Path normalization

Linux-only. The registry keys absolute paths verbatim, no
case-folding. Two callers passing `/foo/Bar` and `/foo/bar` on
case-insensitive filesystems would produce duplicate entries.
Defer until a real cross-platform issue surfaces.

## Identity-check non-regression

`hot-reload.ts` does not use `instanceof` against any registered
class. A static test in `hot-reload.test.ts` reads the source bytes
and asserts zero `instanceof` tokens (after stripping comments and
strings). If you find yourself wanting `instanceof` for error
narrowing, prefer duck-typing:

```ts
err && typeof err === 'object' && 'message' in err
```

— survives across reloads where each new module evaluation produces
fresh constructor identities.

## What's intentionally out of scope

- **Api *classes* are not reloadable.** Direct ESM imports bind
  callers to a specific class object; reloading an Api file does not
  update those bindings. Treat changes to an Api's *surface* (its
  static signatures, the forwarding shell) as a server-restart concern.
  Changes to its *logic*, however, are reloadable: the surface-
  architecture refactor relocates Api logic into the `/platform/idea/api/<feature>`
  singletons described under [Integration](#api-logic-singletons-objapifeature),
  which `dest`-reload like any Stuff.
- **State preservation across reload.** Existing instances keep
  their old prototype chain. No state migration to a new blueprint.
- **CLI / HTTP / in-game command surfaces.** No user-facing trigger
  in v1. Tests drive the API directly.
- **Persistent audit log.** Subscribers to `Events.Module*` keep
  their own ledger if they want one.

The registry is path-agnostic — it works for any absolute path, not
only paths under `packages/server/src/mud/...`. A future mod system
shipping code under `mods/<modname>/...` works without changes.
