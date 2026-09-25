# Stuff Lifecycle

Every game-world object in Saxonberg is a `Stuff`. Stuff has a tightly
specified lifecycle that no class is allowed to bypass:

```
construct → wrap (Proxy) → register → hydrate → postRegister →
            (live) →
destruct → canDestruct (veto) → onDestruct (user cleanup) →
           cleanupOnDestruct walk (framework cleanup) →
           shadow detach → destroy → unregister
```

The whole sequence runs inside `StuffApi`. Subclasses participate via
three narrow extension points: `postRegister(context?)` for setup,
`canDestruct(): VetoResult` for refusing destruction, and
`onDestruct()` for cleanup. Mixin authors have a fourth extension
point — `static cleanupOnDestruct(stuff)` — for substrate-invariant
cleanup that subclass overrides cannot bypass. Everything else is
locked down.

`forceDestruct(target)` is the admin-gated bypass: it invokes the
`canDestruct` witness identically (so observers / audit hooks see the
call) but ignores the veto. v1 the gate is an always-deny stub —
the seam is in place, real permission-aware enforcement lands when
the permission framework does. See
[call-security.md](./call-security.md#admin-only-and-the-force-bypass-shape).

> **Death is not destruction.** A dying organism does NOT run this
> sequence. Death is a `lifecycleState` transition (`alive → dead`) that
> leaves the body in the world — a corpse is the *same* `Stuff` with
> `lifecycleState: 'dead'`, full readable vitals, and animacy gone (so
> `requiresAnimate` verbs stop dispatching). `StuffApi.destruct` is
> removal-from-the-world; death is not. The Vitals subsystem owns the
> death seams (the transition driver is deferred); see
> [vitals.md § Death & consciousness seams](./vitals.md).

This doc covers the lifecycle mechanics. The clone-side hydration
detail (templates, `Hydrator`, `hydratorClass`) lives in
[templates.md](./templates.md). The decorator and security mechanism
that makes the locks enforceable live in
[call-security.md](./call-security.md).

## Construction Sentinel

`Stuff` constructors **throw on direct `new`**. The base `constructor()`
body checks a static sentinel:

```typescript
constructor() {
  if (!Stuff.#expectingConstruction) {
    throw new Error(
      `Direct 'new' on a Stuff subclass is not allowed. ` +
      `Use StuffApi.create(() => new YourClass()) or StuffApi.clone(path).`
    );
  }
  Stuff.#expectingConstruction = false;
  this.stuffId = nanoid();
}
```

`StuffApi`'s creation paths flip the sentinel immediately around `new`:

```typescript
Stuff._beginConstruction();
let raw: T;
try { raw = factory(); }
finally { Stuff._endConstruction(); }
```

The sentinel must be set with no intervening `await` — otherwise a
parallel call could observe it set and bypass the gate. The pattern is
"flip → construct → flip back" inside a single synchronous scope.

`_beginConstruction` / `_endConstruction` are themselves protected by a
**stack-walk allowlist** (`Stuff.#assertConstructionGateAllowed`):

| Allowed file pattern | Why |
|---|---|
| `/mud/api/` | StuffApi's create / clone / createSync |
| `/mud/lib/security/__tests__/test-setup.(ts\|js)` | the `makeStuff` test helper |
| `*.test.(ts\|js)` | direct test usage |

Anything else throws. Cached per-URL after first check, so the runtime
cost is one stack walk per file ever.

## Creation Paths

Three ways to make a Stuff. All go through `StuffApi`. All flip the
sentinel, wrap in a Proxy, register, and (for the async paths) run the
synthetic constructor frame.

### `StuffApi.clone<T>(templatePath, context?): Promise<T>`

The production path. Loads a template from `Collections.Domain`,
dynamic-imports the backing class, optionally hydrates from
`template.data`, awaits `postRegister`. Full pipeline documented in
[templates.md § The Clone Pipeline](./templates.md#the-clone-pipeline).

### `StuffApi.create<T>(factory, context?): Promise<T>`

Caller-supplied factory; no template lookup; no hydration step. Same
register + `postRegister` tail as `clone`. Used for runtime-only
objects whose construction needs explicit arguments and don't
round-trip through the CMS pattern. `Interactive` is the canonical
example (`socketId`, `sessionId`, `user` flow through the closure):

```typescript
const interactive = await StuffApi.create(
  () => new Interactive(socketId, sessionId),
  { user }
);
```

### `StuffApi.createSync<T>(factory): T`

Sync sister of `create`. Same sentinel-flip + Proxy wrap + register
guarantees, no hydrate, no `postRegister` await. **Throws if the
constructed Stuff composes `PostRegistrationMixin`** — silently skipping
`postRegister` would yield a half-initialised object, so the throw
forces such classes onto the async `create()` path.

Used inside sync helpers where awaiting would force the caller (and its
callers) to become async too. `Exitable.addBidirectionalExit`'s
`new Exit(...)` calls are the typical trigger.

### `StuffApi.singleton<T>(templatePath, context?): Promise<T>`

The "one-and-only-one instance per path" lookup. Returns the existing
instance from the `byTemplatePath` index when present; otherwise
delegates to `clone()`. Pairs with `SingletonMixin` (a marker mixin
in `lib/stuff/Singleton.ts`) for enforcement: `clone()` itself does
a pre-flight `byTemplatePath` check on classes composing
`SingletonMixin` and throws on the second clone attempt. `singleton()`
is the convenient surface that respects the contract automatically;
shared-state Stuff (the starting room, the EventRegistry) should
use it instead of `clone()`.

## What Registration Actually Does

`StuffApi.register(proxy)` adds the proxy to `objectsById:
Map<string, Stuff>`. After registration, anyone with the `stuffId` can
resolve the object via `StuffApi.findById(id)`.

It also stamps the secondary `byTemplatePath: Map<string, Set<Stuff>>`
index when the proxy carries a templatePath, so subsequent
`findByTemplatePath(path)` and `singleton(path)` lookups find it.
`unregister(proxy)` (driven by `Stuff.destroy`) removes the entry
from both indexes and deletes the bucket when its size hits zero —
that empty-bucket cleanup is what the `singleton()` pre-flight
relies on, so the destroy path must run before the next clone of a
singleton template.

Registration happens **before** hydrate and `postRegister`. The
ordering is load-bearing: a hydrator might resolve the in-flight object
by id (e.g., a self-referencing exit), and that lookup must succeed.

If hydrate or `postRegister` throws, the object is unregistered before
the error propagates. Half-initialised objects never linger in the
registry.

## Synthetic Constructor Frame

Hydrate and `postRegister` run inside a synthetic frame planted by
`ExecutionContextApi.run`:

```typescript
await ExecutionContextApi.run(
  StuffApi,        // caller
  proxy,           // target
  'constructor',   // method
  { kind: FrameKind.Constructor },
  async () => {
    if (hydrate) await hydrate(proxy);
    if (MixinApi.isPostRegistration(proxy)) {
      await proxy.postRegister(context);
    }
  }
);
```

Anything those steps invoke sees `caller = StuffApi` and
`target = <new instance>`. Inner `this.foo()` calls then appear as
self-calls — the natural reading of construction-time
self-initialisation. Without the frame, those calls would have
`caller = null` and policies like `SelfOnly` would behave inconsistently.

`FrameKind.Constructor` is the typed marker. See
[call-security.md](./call-security.md) for the broader
`FrameKind` taxonomy.

## What `Stuff` Provides

Every `Stuff` carries:

- **`stuffId: string`** (readonly) — runtime ID, generated by `nanoid()`
  in the base constructor.
- **`templatePath: string | null`** — clone path, or `null` for
  `create`/`createSync`. Stamped by `StuffApi.clone()`. Identity-keyed
  security policies (notably `FromTemplate`) match against this.
- **`zone: Zone | null`** — universal subdivision. Stamped at clone
  time by `ZoneApi.resolveZoneForPath` from the template path. NOT
  back-filled by `ContainmentApi.move`: zone identity follows
  whichever template spawned the item, not whichever container it
  currently sits in.
- **`isDestroyed(): boolean`** — `@Final @Unshadowable`. Reads a
  private `_isDestroyed` flag.
- **`canDestruct?(): VetoResult`** — optional Witness hook. Subclasses
  declare it to refuse destruction (`{ ok: false, reason }`); the
  default absence is treated as `{ ok: true }`. Bypassable via
  `StuffApi.forceDestruct` (admin-gated).
- **`onDestruct(): void`** — Witness hook. Cleanup runs while the
  target is still live (mirror of how the retired `prepareDestroy()`
  ran before `destroy()`). `Stuff` ships a no-op terminal so a
  subclass overriding the hook can call `super.onDestruct()`
  unconditionally — no cast-to-optional dance required. Subclasses
  that have no cleanup to do simply don't override.
- **`destroy(): void`** — `@Final @Unshadowable @CallSecurity(ApiOnly)`.
  The body is FINAL; the call is privileged.

## Destruction

Destroy via `StuffApi.destruct(obj)`. The Api-layer entry point runs:

```typescript
public static destruct(object: Stuff): void {
  StuffApi.#destructCore(object, /* force */ false);
}

@CallSecurity(SecurityPolicies.AdminOnly)
public static forceDestruct(object: Stuff): void {
  StuffApi.#destructCore(object, /* force */ true);
}

static #destructCore(object: Stuff, force: boolean): void {
  // 1. canDestruct witness (refusal seam). Force still INVOKES the
  //    hook so observers / audit fire identically — only the
  //    assertion is skipped.
  const veto = callHook(object, 'canDestruct');
  if (!force) assertVetoOk(veto, 'canDestruct');

  // 2. onDestruct witness (cleanup). Runs while the target is still
  //    live — the proxy's destroyed-object guard fires only after
  //    `_isDestroyed` is set in step 5.
  callHook(object, 'onDestruct');

  // 3. Framework cleanup walk. Each mixin whose constructor has
  //    its OWN `static cleanupOnDestruct(stuff)` runs, most-derived
  //    first. Substrate-invariant cleanup (containment unhook,
  //    slot vacate, Spawner untrack) — subclass `onDestruct`
  //    overrides cannot bypass it because statics aren't on the
  //    prototype chain. Per-handler try/catch: a throw is logged
  //    and the loop continues. See `docs/ref-shapes.md` (R2.4) and
  //    `docs/subsystems/mixins.md`.
  for (const mixinCtor of MixinApi.queryMixins(object.constructor)) {
    if (Object.prototype.hasOwnProperty.call(mixinCtor, 'cleanupOnDestruct')) {
      try { mixinCtor.cleanupOnDestruct(object); }
      catch (err) { /* log + continue */ }
    }
  }

  // 4. Privileged shadow detach (bypasses @ShadowSecurity per spec —
  //    destruction is non-negotiable)
  ShadowApi._detachAllForHost(object);

  // 5. destroy() runs straight to the original body
  object.destroy();
}
```

And `Stuff.destroy()`:

```typescript
@Final
@Unshadowable
@CallSecurity(SecurityPolicies.ApiOnly)
public destroy(): void {
  if (this._isDestroyed) return;     // double-destroy guard
  this._isDestroyed = true;
  StuffApi.unregister(this);         // critical housekeeping
}
```

Order is rigid:

1. **`canDestruct()`** (optional Witness on the target). Vetoing
   shapes the same `VetoResult` discipline as `canMove` /
   `canEnter`; absence = `{ ok: true }`. `forceDestruct` skips
   only the assertion, not the invocation.
2. **`onDestruct()`** (optional Witness on the target). Cleanup
   hook — runs while the target is still live so it can touch
   `this` through the proxy. Replaces the retired
   `prepareDestroy()` hook. User customization first; framework
   cleanup second.
3. **`cleanupOnDestruct` walk** over the mixin chain. Each mixin
   whose constructor carries its OWN `static cleanupOnDestruct`
   runs, most-derived-first. Substrate-invariant cleanup that
   subclass `onDestruct` overrides cannot bypass (statics aren't
   on the prototype chain). Force and non-force paths run this
   identically. Per-handler try/catch: a throwing handler is
   logged and the loop continues; `destroy()` still runs. See
   [`docs/ref-shapes.md`](../ref-shapes.md) (R2.4) and
   [`docs/subsystems/mixins.md`](./mixins.md) for the contract.
4. **Privileged shadow detach** removes every shadow from the host.
   Bypasses `@ShadowSecurity({ detach })` because host destruction
   is unconditional.
5. **`destroy()`** runs. By the time the body executes, the host is
   shadow-free, so the call goes straight to the original body — no
   shadow can intercept and skip the unregister.

Direct `obj.destroy()` from outside `mud/api/` throws `SecurityError`
(via `@CallSecurity(ApiOnly)`). Subclass `destroy()` overrides throw
`FinalViolationError` at import time (via `@Final`'s loader-hook
validation). Shadows attempting to attach to `destroy()` throw
`ShadowError` (via `@Unshadowable`). Together these guarantee that
`StuffApi.unregister()` always runs — essential for GC.

## Why `onDestruct()` and `canDestruct()` are the Override Points

`destroy()` carries critical housekeeping that the system depends on:
mark the object destroyed, unregister from the global map. If a
subclass overrode `destroy()` and forgot to call `super.destroy()`,
the registry would leak forever.

Making `destroy()` `@Final` means the language refuses to compile the
override. Making it `@Unshadowable` means a runtime shadow can't
intercept it. Making it `@CallSecurity(ApiOnly)` means non-Api code
can't even call it directly — they have to go through
`StuffApi.destruct()`, which guarantees the full sequence
(canDestruct → onDestruct → shadow-detach → destroy).

`canDestruct()` and `onDestruct()` are the safe extension points.
`canDestruct` is an optional Witness hook (declare only when the
class needs to refuse). `onDestruct` is a guaranteed-present method
on `Stuff` (no-op terminal); subclasses override and call
`super.onDestruct()` to chain through layered cleanup. Both run
before any of the housekeeping in step 4, so they can touch `this`
through the proxy.

```typescript
class Avatar extends AvatarBase {
  // Cleanup only — no refusal reason for an avatar.
  public onDestruct(): void {
    PlayerApi.unregisterAvatar(this);
    this.interactives.clear();
  }
}

class SpatialZone extends Zone {
  // Refusal — drain locations before destruct.
  public canDestruct(): VetoResult {
    if (this.locations.size > 0) {
      return {
        ok: false,
        reason: `cannot destruct zone '${this.getName()}' with `
              + `${this.locations.size} live location(s); `
              + `destruct locations first`,
      };
    }
    return { ok: true };
  }
}

class VoidLocation extends SingletonMixin(Location) {
  // Bootstrap-pinned singleton — `ContainerMixin.cleanupOnDestruct`
  // evacuates HasInteractive bodies to `/world/void` via sync
  // `findByTemplatePath`, so the void must stay live for the
  // process lifetime. Same shape as EventRegistry / Clade.
  public canDestruct(): VetoResult {
    return {
      ok: false,
      reason: 'the void is a system singleton; use forceDestruct',
    };
  }
}
```

`EventRegistry` and `Clade` follow the same shape — they're
bootstrap-pinned and consumed by sync `findByTemplatePath` lookups
in hot paths. Destroying them mid-session would silently break
downstream lookups, so they refuse `canDestruct` unconditionally and
expose `forceDestruct` (admin-gated) as the escape hatch.

### Container cleanup policy

`ContainerMixin.cleanupOnDestruct` runs as part of the framework
cleanup walk. Its policy for each direct content item:

| Outer container of the destructing Container | Per-item policy |
|---|---|
| Non-null (the destructing Container is itself contained) | Re-parent every item to the outer via `ContainmentApi.move`. |
| Null AND item composes `HasInteractive` (live avatar) | Evacuate to the void singleton (`/world/void`) via `ContainmentApi.move`. Active sessions never end up with a null environment. |
| Null AND item is not `HasInteractive` | Cascade-destruct via `StuffApi.destruct(item)`. |

The HI escape leans on `/world/void` being live before any
container can destruct — bootstrap guarantees that (the
`{ templatePath: '/world/void' }` entry runs after
`/platform/idea/EventRegistry`).

The policy targets only the DIRECT contents of the destructing
host. A HasInteractive nested inside a non-HasInteractive Container
that itself sits inside a top-of-containment host gets
cascade-destructed when the inner Container destructs (which moves
it to the top-host while that's mid-destruct, then R2.3 self-heal
nulls its container). If that nested-HI case becomes real, lift
the rule into `ContainmentApi.move` as a chokepoint.

Mixin-side overrides chain via `super.onDestruct?.call(this)` — the
hook is optional, so the chain bottoms out cleanly at any class
that doesn't declare one. Don't call `super` unless an ancestor
mixin actually defines `onDestruct`.

## Failure Rollback

If hydrate or `postRegister` throws during creation:

```typescript
try {
  await ExecutionContextApi.run(/* ... */, async () => {
    if (hydrate) await hydrate(proxy);
    if (MixinApi.isPostRegistration(proxy)) {
      await proxy.postRegister(context);
    }
  });
} catch (error) {
  this.unregister(proxy);   // <-- rollback
  throw error;
}
```

The proxy is unregistered before the error bubbles up. Half-initialised
objects don't linger.

`createSync` short-circuits **before** registering when the class
composes `PostRegistrationMixin` — there's nothing to roll back because
the object never made it into the registry.

## The `makeStuff` Test Seam

Tests sometimes need to construct a `Stuff` without the full clone
pipeline. Direct `new` is blocked, so there's a single sanctioned
helper at `mud/lib/security/__tests__/test-setup.ts` (`makeStuff`).
Its file URL is on the construction-sentinel allowlist. Use it for
test setup; do NOT replicate its code elsewhere — that would just
re-open the bypass the sentinel exists to close.

## Idle Eviction — shipped (residency)

Self-eviction of the abandoned cold tail is now the **residency**
substrate — see [residency.md](./residency.md). It resolves the questions
this section used to pose:

- **Triggering** — a real-time `ScheduleApi.recurring` sweep over
  `StuffApi.getAllObjects()`, keyed on a per-instance recency timestamp
  (`Stuff.lastTouched`, bumped by the security gate on every successful
  non-getter dispatch + a presence walk). Not a per-instance TTL, not an
  ordered LRU list (a lazy O(n) scan — we sweep on a timer, not on access
  pressure).
- **Granularity** — opt-**out**: `Stuff.canEvict` defaults to cull;
  anything that must survive vetoes on the owning mixin/class (the
  `ApiLogic` categorical veto for logic singletons + the R2.x-derived
  relational roster). Game-world rooms (`Location`), session holders
  (`HasInteractive`), and non-empty containers veto; abandoned items cull.
- **Ordering vs `onDestruct`** — eviction *is* `StuffApi.destruct`: the
  full choreography (`onDestruct` → `cleanupOnDestruct` → shadow detach →
  unregister) runs unchanged. It is a garbage-culler, not a swapfile —
  culled objects are gone and re-clone fresh on next reference.
- **Visibility / safety** — ships in **observe mode** (logs candidates,
  culls nothing) until an operator flips `residency.mode` to `enforce`.

(Auth/CMS records — `User`, `Template`, `GoogleProfile` — remain
`Document`s, **not** Stuff, so they are never registered and outside
residency's scope entirely.)

## Cross-References

- [templates.md](./templates.md) — clone pipeline, `Hydrator`,
  `PostRegistrationMixin`, the context bag, `TemplateApi`, the
  folder/leaf invariant
- [persistence.md](./persistence.md) — the `Document` track (auth/meta
  records are not Stuff and have no lifecycle), around-save/delete hooks
- [call-security.md](./call-security.md) — `@Final`, `@Unshadowable`,
  `@CallSecurity(ApiOnly)` decorators; `ProxyApi.wrap`;
  `ExecutionContextApi.run`; `FrameKind`; how `destroy()` is locked
  down at runtime
- [state-model.md](./state-model.md) — what gets persisted across the
  lifecycle; Avatar's "self-contained" design
