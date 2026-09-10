# Event System

This document describes the engine's two-layer reactivity story —
the **Witness pattern** for object-local hooks and **EventApi** for
global pub/sub — plus the supporting `EventRegistry` Idea that
backs the bus and the framework prerequisites that make it all
work.

## Scope

Two complementary mechanisms for "X happened, code Y wants to react":

1. **Witness pattern** — object-local notifications via named methods on
   receivers. Use this for any event where the receiver(s) are known
   targets (a specific Stuff). Most "event-shaped" reactions in the game
   should use this.
2. **EventApi** — global pub/sub bus for events that have no specific
   target to shadow (engine-level lifecycle, no-target-yet bootstrap
   events, cross-cutting audit streams).

The two are complementary, not redundant. Same call site can fire both
when both apply.

## Motivation

Current code has many "event-shaped" patterns — `Sensor.receiveMessage`,
`Mobile` message hooks, `AroundSaveHook` — but no broadcast notification
primitive. Mod hooks, achievements, audit logging, hot-reload
notifications, and engine lifecycle observers all want fan-out semantics.
`MobileMixin` already has hook stubs that motivated the roadmap entry
under "Event System (Framework 9)."

## When to use which

| Use case | Mechanism |
|---|---|
| Movement, attack, death of a known target | Witness on the target |
| Quest "kill the dragon" | Shadow on the dragon (per-target subscription) |
| "Bob watches Phil" | Shadow on Phil |
| Achievement: "Phil visited 100 rooms" | Shadow on Phil's `onMoverDeparted`, installed at login |
| Tracking aggregate movement | Don't. Use shadows on each player; install via the `PlayerLoggedIn` event |
| **Login / logout** | **EventApi** (no Connection object exists before login) |
| **`StuffCreated`** (any new Stuff of any kind) | **EventApi** (no target exists yet) |
| **`Module*` lifecycle (reloaded / rolledBack / unloaded / reloadFailed)** | **EventApi** (no target object) |
| **Audit / mudlog / server-wide metrics** | **EventApi** (cross-cutting, no target) |
| **Mod plugin hooks** for engine-wide phenomena | **EventApi** |
| Around-save / around-delete | Existing `AroundSaveHook`, not events |
| Sensor message filter, Mobile messaging | Existing local methods, not events |

Rule of thumb: **shadows + Witness for things with targets; EventApi for
things without.** Don't dual-emit movement globally just to make
subscribe-by-event easier — force the shadow path because the
shadow-on-the-target version has natural lifetime cleanup and zero cost
when no one's listening.

---

## Witness pattern

The codebase's idiom for "X happened to a known target object."

### Contract

- Publisher has a chokepoint API (`Mobile.traverse`, `ContainmentApi.move`,
  `MessageApi.send`, etc.).
- After the operation commits, publisher dispatches to **named methods** on
  relevant receivers.
- Receivers opt in by composing a mixin that declares the methods.
  Methods have void / return-ignored signatures (notification, not
  interception).
- Publisher narrows via `MixinApi.isX(receiver)` before calling.
- Shadows override per-method, individually targetable.

### Naming convention

Two flavors of hook per event:

- **After-hook** (notification): `on<Past-participle>` — e.g., `onMoverEntered`
- **Before-hook** (veto): `can<Infinitive>` — e.g., `canMoverEnter`

Tense rules:
- Before-hook verb is **infinitive/imperative**: `moverEnter`, `doorOpen`,
  `itemTake`.
- After-hook verb is **past participle**: `moverEntered`, `doorOpened`,
  `itemTaken`.

Return types:
- Before-hook returns `{ ok: true } | { ok: false, reason: string }`
  (matching the existing `Exit.canPass` shape).
- After-hook returns `void`.

Not all events have both. Most will. When both exist, the publisher
calls all `can<X>` hooks first, short-circuits on any veto, and only
fires the operation + `on<X>` hooks if every veto passed.

### Rule for where hooks live

If the receiver capability already exists as a mixin (`Mobile`,
`Container`, `Containable`, `HasInteractive`), the hooks live as
**overridable methods on that mixin**. A new "Witness" mixin is only
created when the receiver capability is genuinely new — e.g., "this
object watches movement happen to *other* things."

This keeps the mixin tree from exploding into one mini-mixin per signal
while still using mixins as the canonical detection mechanism.

### Catalog of initial Witness hooks

All hooks are **optional methods on existing mixin interfaces** — no
new "Witness" mixin is created. The receiver mixins themselves
declare the methods as optional in the interface; the chokepoint
Apis dispatch via `typeof === 'function'` so absent methods cost
nothing.

Two layers fire from two chokepoints:

- **Containment layer** (fires from `ContainmentApi.move`): every
  relocation, including drops, gets, teleports, and the inner move
  inside `Mobile.traverse`. Hooks live on `Containable` and
  `Container`.
- **Traversal layer** (fires from `Mobile.traverse`): only when a
  Mobile creature crosses an Exit. Hooks live on `Mobile` and
  `Exitable`.

A Mobile traversing an exit fires both layers. A sword being dropped
fires only containment. A creature being teleported fires only
containment.

```text
Containable (anything that can be in a container):
  canMove?(to: Container | null): VetoResult
  onMoved?(from: Container | null, to: Container | null): void

Mobile (creatures, vehicles — composes Containable):
  canTraverse?(via: Exit): VetoResult
  onTraversed?(via: Exit): void
  // inherits Containable hooks — both layers fire on traversal

Container (any container — chest, location, backpack):
  canAddContainable?(thing: Containable): VetoResult
  canRemoveContainable?(thing: Containable): VetoResult
  onContainableAdded?(thing: Containable): void
  onContainableRemoved?(thing: Containable): void

Exitable (locations with exits):
  canEnter?(mover: Mobile, via: Exit): VetoResult
  canExit?(mover: Mobile, via: Exit): VetoResult
  onEntered?(mover: Mobile, via: Exit): void
  onExited?(mover: Mobile, via: Exit): void

HasInteractive (any Stuff that can hold a connection):
  onConnectionAttached?(conn: Interactive): void
  onConnectionDetached?(): void
  onLinkdead?(): void
  onLinkRestored?(): void
```

**Stuff lifecycle**: `PostRegistration.postRegister(context)` and
`onDestruct()` cover construction and teardown. Use those for
new-instance / about-to-be-destroyed reactions; they predate the
Witness pattern but fit the same shape.

**Naming notes**: Mobile uses `canTraverse` / `onTraversed`
(mirrors the existing `Mobile.traverse(exit)` method name) and
Exitable uses the symmetric `canEnter` / `canExit` / `onEntered` /
`onExited` pair. No method-name collision if anything ever composes
both. The Mobile-side hooks deliberately drop the "Mover" prefix
that an earlier draft used — Mobile is the receiver here, no
prefix is needed for disambiguation.

**State-mutation chokepoint**: `Containable.setContainer` is the
atomic chokepoint that orchestrates `removeContainable` (old
container) → `addContainable` (new container) → field set. It is
`@CallSecurity`-gated to `ContainmentApi` callers only and `@Final
@Unshadowable` because inventory state going out of sync is
catastrophic. `ContainmentApi.move` is the public surface that adds
policy + Witness dispatch above. `Container`'s `addContainable` /
`removeContainable` are similarly locked down — only callable from
inside `setContainer`. See `mud/api/containment.ts` and
`mud/lib/spatial/Containable.ts` / `Container.ts` for the call
graph.

**Vocabulary**: `Container`'s state mutators are `addContainable` /
`removeContainable` (matching the `Containable` mixin name and the
hook names). The `getContents()` read accessor stays as-is. Both
follow the `contain*` root family.

### Shadow interaction

Shadows participate in Witness dispatch via the proxy chain. A shadow
that wants to react to an event composes the relevant mixin and
implements the method directly in its class body. See **Framework
prerequisites → Shadow callDown by default** below — that change is
required for this to work without traps.

`MixinApi.hasMixin(obj, Mixins.X)` should walk the shadow stack so a
publisher checking `isMovementWitness(room)` returns true if the host
OR any shadow on the host composes the mixin. (Confirm this is already
the semantic; if not, it's a small addition.)

---

## EventApi

Global pub/sub for events with no specific target.

### Surface

```ts
EventApi.emit<T>(eventName: string, payload: T): void
EventApi.on<T>(eventName: string, listener: Listener<T>, opts?: SubOpts): Subscription
EventApi.once<T>(eventName: string, listener: Listener<T>): Subscription
```

```ts
type Listener<T> = (payload: T, ctx: ListenerContext) => void | Promise<void>;

interface SubOpts {
  filter?: (payload: T) => boolean;   // pre-dispatch predicate; false = skip
  until?: (payload: T) => boolean;    // post-dispatch predicate; true = unsub
}

interface Subscription {
  unsubscribe(): void;
  readonly eventName: string;
  readonly lastPayload: T | null;     // replay-on-subscribe (see below)
}

interface ListenerContext {
  triggeringContext: ExecutionContextSnapshot | null;  // who caused the originating emit
}
```

`once(name, listener)` is sugar for `on(name, listener, { until: () => true })`.

### Well-known events (initial set)

```ts
export const Events = {
  StuffCreated:        'stuff.created',
  StuffDestructed:     'stuff.destructed',
  ConnectionAttached:  'connection.attached',
  PlayerLoggedIn:      'player.loggedIn',
  PlayerLoggedOut:     'player.loggedOut',
  ModuleReloaded:      'module.reloaded',
  ModuleRolledBack:    'module.rolledBack',
  ModuleUnloaded:      'module.unloaded',
  ModuleReloadFailed:  'module.reloadFailed',
} as const;
```

The four `module.*` lifecycle events all share the `ReloadEvent`
payload shape:

```ts
interface ReloadEvent {
  path: string;                       // absolute fs path of the module
  versionId: string | null;           // truncated sha256 of source bytes; null for Unloaded
  previousVersionId: string | null;   // version this transition replaced; null when none
  exports: string[];                  // class export names from the new module; empty for Unloaded / Failed
  error?: { message: string; stack?: string };  // ReloadFailed only
}
```

`emittableBy(HotReloadApi)` on all four — only `HotReloadApi`'s public
methods can fire them. Subscribe is open by default. See
[hot-reload.md](./hot-reload.md) for the state machine and
integration points.

String-keyed with **hierarchical dot-notation** values. The TS keys
(`Events.PlayerLoggedIn`) are ergonomic identifiers; the underlying
strings (`'player.loggedIn'`) are the namespaced paths used as
property keys on `EventRegistry` and as the wire-format event names.
The dot-notation sets up a path forward for wildcard subscriptions
(`EventApi.on('player.*', ...)`) without breaking the v1 API —
wildcards aren't on initial ship but the naming makes them
straightforward to add. Each event has an associated payload type
declared via a parallel type-map (`EventPayloads`) for type-safe
subscribers.

### Custom events

Custom events are first-class — any string `name` works with
`EventApi.emit` / `EventApi.on`. The `Events.*` table is just
documentation + an opportunity to install a tighter per-event
policy; everything else is treated the same.

The mechanism: on first `emit` or `on`, EventApi calls
`reg.initProp(name, { transient: true, checkAccess: emittableBy() })`
— the default open-public policy that still requires the
EventApi-mediated path (so a direct `reg.setProp('forged', ...)`
from anywhere else stays rejected). Idempotent: calls for
already-registered events (e.g. anything from the well-known
table) no-op.

Well-known events differ only in being **frontloaded**:
`EventRegistry.postRegister` declares them at registry creation
with their specific allowlists from `lib/events.ts`
(`emittableBy(StuffApi)` for `StuffCreated`, etc.). If you find
yourself wanting a non-default policy for a custom event, promote
it into the `Events.*` table and add a `POLICIES` entry. There is
no separate "custom event index" — registration is uniform.

`EventRegistry.defaultPropAccess` is overridden to deny everything,
which closes the auto-init bypass: PropertiedMixin's `setProp`
auto-initialises unknown props by calling `initProp(prop)` with no
options, which falls back to `defaultPropAccess`. With deny-all
defaults, that path produces an inaccessible prop — only the
EventApi-mediated path (which calls `initProp` with a real
`emittableBy()` policy first) survives.

This list is intentionally small. Anything Stuff-shaped should go
through Witness, not here. Add new events only when there's no
single-target alternative.

### EventRegistry — the underlying storage

`EventRegistry` is an Idea in `mud/platform/idea/EventRegistry.ts`, composing
`PostRegistrationMixin(PropertiedMixin(Idea))`. Singleton. Lives at
template path `/platform/idea/EventRegistry`. Bootstrapped by `BootstrapManager`
(see [bootstrap.md](./bootstrap.md)).

**Not a persisted record.** Every event prop is `transient: true`, the
`checkAccess` closures don't survive serialization, and there's
nothing meaningful to round-trip. Composition deliberately stops at
`PropertiedMixin(Idea)`.

Each well-known event is a property on the registry, initialized at
bootstrap time inside `postRegister`. The property's `checkAccess`
function is what gates publishers and subscribers.

`EventApi` reaches into the registry to set/get props; nothing else
should touch it. The registry itself does not export an "emit" or
"subscribe" surface — those are EventApi's job.

**Singleton lookup pattern**: EventRegistry doesn't own its own
static accessor. EventApi uses
`StuffApi.findByTemplatePath('/platform/idea/EventRegistry')` to resolve the
instance, caching the reference on the API class. The same pattern
generalizes to every future Registry singleton — no per-class
boilerplate.

### EventSubscriptions — the runtime subscriber store

There are **two event singletons, two responsibilities.**
`EventRegistry` (above) holds the event-property *declarations* — the
well-known events and their `checkAccess` gates. `EventSubscriptions`
(an Idea at `/platform/idea/EventSubscriptions`, `mud/platform/idea/EventSubscriptions.ts`)
holds the runtime *subscriptions* — who's listening — plus the bounded
recent-payload history ring buffer.

```ts
class EventSubscriptions extends Idea {
  private subs: Map<string, Set<SubscriptionRecord>>;
  private history: Map<string, HistoryRecord[]>;  // HISTORY_LIMIT = 100
}
```

Both state slots are `private`, not `#`-private: the singleton is a
Stuff host whose instance methods dispatch through the call-security
proxy, where `#`-private slots are unreachable. Every public method
carries `@CallSecurity(FromModule('/api/event#EventApi'))`, so
`EventApi` is the singleton's only legitimate caller; the Api itself is
a thin facade. Moving this state off the Api class lets subscribers and
history survive an HMR reload of the Api file (a reload of *this* file
still resets the table — listeners re-register after such a reload).

### Dispatch mechanism

```ts
EventApi.emit<T>(name: string, payload: T): void {
  const ok = EventRegistry.instance.setProp(new Property<T>(name), payload);
  if (!ok) throw new SecurityError(`Not allowed to emit ${name}`);
  // Snapshot the current subscribers off EventSubscriptions, then
  // queueMicrotask the fan-out so listeners fire on a clean frame.
  const subs = EventSubscriptions.instance.snapshot(name);
  if (subs.length === 0) return;
  queueMicrotask(() => { /* fresh-frame dispatch per listener */ });
}

EventApi.on<T>(name: string, listener: Listener<T>, opts?: SubOpts): Subscription {
  const propOpts = EventRegistry.instance.checkProp(new Property<T>(name));
  if (!propOpts) throw new SecurityError(`Not allowed to subscribe to ${name}`);
  const lastPayload = EventRegistry.instance.getProp(new Property<T>(name));
  // Build the InternalSubscription record and hand it to the runtime
  // subscriber singleton.
  EventSubscriptions.instance.addSubscription(name, record);
  return subscription;
}
```

Subscribers live on the `EventSubscriptions` singleton Idea
(`/platform/idea/EventSubscriptions`) — `Map<eventName, Set<SubscriptionRecord>>`
keyed by event name. NOT in the EventRegistry's props (subscribers
aren't declarations; they're runtime registrations) — see the two-
singletons note above.

### Last-payload cache

`emit` stores the payload via `setProp` as a useful side effect. `on`
reads it via `getProp` and exposes it on the returned Subscription as
`lastPayload`. New subscribers can optionally use this for initial state
("what was the last login?") without needing a separate replay
mechanism.

The Get permission gates both subscribing AND reading the last value —
same intent (the subscriber is allowed to receive payloads on this
event), same gate.

### Disambiguating "never emitted" from "denied"

`getProp` returns null for both "access denied" AND "no value yet
stored." `EventApi.on` disambiguates via `checkProp`: with the Phase 1
Get-gate, `checkProp` returns null when EITHER the prop doesn't
exist OR the caller can't see it, and a non-null result confirms
both existence and visibility. After a successful `checkProp`,
`getProp` returning null cleanly means "no payload yet" — no
sentinel value is needed (an earlier draft proposed a `NEVER_EMITTED`
sentinel; the Get-gated `checkProp` makes it redundant).

### Execution model

- **Non-blocking:** `emit` returns synchronously after queueing.
  Listeners fire on the next microtask. This sidesteps reentrancy stack
  depth and gives every listener a clean call frame.
- **Fresh frame per listener:** each listener runs in a new
  `EventDispatch` `FrameKind` via the existing `ExecutionContextApi`
  machinery. Distinct frame kind so the security layer can tell "this
  is a listener firing" from "this is a direct call."
- **Attribution preserved by reference:** the originating context is
  available via `ctx.triggeringContext`. Listeners run as system code,
  not as the originator (so they don't inherit the originator's
  permissions), but can attribute the event back.
- **Errors isolated:** thrown errors / rejected promises in one
  listener don't propagate to siblings or to the emitter. Logged via
  `MudlogApi`.
- **Promise-returning listeners:** the bus catches `.catch()` for error
  logging but does NOT await. Fire-and-forget.
- **Dispatch order:** registration order. Not a public guarantee.

### Filter and until

Subscription options, not a boolean-return convention:

```ts
EventApi.on(Events.ModuleReloaded, listener, {
  filter: (evt) => evt.path.startsWith('/foo/bar'),
  until:  (evt) => evt.path === '/done',
});
```

Why options instead of boolean return:
- Boolean return is a footgun. Most listeners return `undefined`; an
  accidental `return foo` could unsubscribe silently.
- `filter` runs before the listener, keeping listener bodies focused.
- Both options are introspectable on the Subscription record (useful
  for admin tooling: "show me all listeners on `StuffCreated` with
  their filters").

Per-subscription `filter`/`until` are not shadowable. If they ever need
to be, that's a future concern.

### History / introspection

A bounded ring buffer per event symbol (`HISTORY_LIMIT = 100`
entries), resident on the `EventSubscriptions` singleton and accessed
via `EventApi.history(eventName, limit?)`, which delegates. Stores
`{ payload, timestamp, triggeringContext }`. For admin debugging.

Note: this is *event* history (recent fired payloads), distinct from
the `lastPayload` cache (which is exactly the most recent payload).
Last-payload is for replay-on-subscribe; history is for diagnostics.

---

## Permission gates

### emittableBy helper

Every event needs a `checkAccess` function on its property. Writing
the two-level check inline per event is error-prone. Use a helper:

```ts
// Lives in mud/api/event.ts
export function emittableBy(
  ...allowed: Function[]
): PropAccessCheck<PropValue> {
  return (_prop, op) => {
    const stack = ExecutionContextApi.getCallStack();

    // Defense: somewhere on the stack, EventApi (the API class
    // itself, target on a static-decorator frame) must be present.
    const eventApiIdx = findClassFrame(stack, EventApi);
    if (eventApiIdx < 0) return false;

    if (op === PropOperations.Get) {
      // Open subscribe by default. Tighten with eventPolicy({ emit,
      // subscribe }) if a particular event needs it.
      return true;
    }
    if (op !== PropOperations.Set) return false;
    if (allowed.length === 0) return true; // open emit

    // The originator is the frame just BELOW the EventApi frame.
    const originator = stack[eventApiIdx - 1]?.target;
    return originatorMatches(originator, allowed);
  };
}

// Per-event registration is a one-liner:
reg.initProp(Property.of(Events.PlayerLoggedIn), {
  transient: true,
  checkAccess: emittableBy(PlayerApi, ConnectionApi),
});
```

Two-level check inside the helper:

1. **Defense**: at least one frame on the stack must have
   `target === EventApi` (the API class). This catches anyone who
   gets hold of the EventRegistry instance and tries to call
   `setProp` directly — without the EventApi-mediated path, the
   defense fails. Implemented as a stack walk rather than
   "current frame == EventApi" so intervening frames (the
   PropertiedMixin's own `checkAccess` proxy mediation, etc.) don't
   break the check.
2. **Allowlist (originating caller)**: the frame immediately below
   the EventApi frame is the originating caller. Match its class
   (or the class it belongs to) against the allowed list. The
   originator may be either a class object (static-decorator
   frame) or an instance (proxy frame); the helper handles both.

### subscribableBy helper

Same shape, gates `Get`. Default for events not using it: open
subscribe.

```ts
export function eventPolicy(opts: {
  emit: Function[];
  subscribe?: Function[];  // default: any
}): PropAccessCheck<PropValue> { ... }
```

---

## Framework prerequisites

These three framework adjustments shipped alongside the event system
in Phase 1. They benefit the codebase regardless of events.

### 1. Shadow callDown by default

**Today** (`ShadowApi.#computeInterceptSet` in `api/shadow.ts`):
inferred surface = "own properties of the shadow's own class
prototype" + any `@Shadowing` decorations. Methods inherited from
composed mixin layers are part of the shadow's *type contract* but
NOT its *intercept set*. A shadow that composes `Containable` to
satisfy the type interface but only declares its own `onMoved`
intercepts only `onMoved`; the inherited `setContainer` from
ContainableMixin doesn't enrol.

This was a behavior change from an earlier rule that walked the
prototype chain for mixin marker layers and added every method on
every layer. The new rule is what lets a shadow opt into ONE Witness
hook without accidentally capturing every other method on the same
mixin.

### 2. `checkProp` extended to consult access

`PropertiedMixin.checkProp` returns the prop's options when the prop
exists AND the caller has Get access; otherwise null. Single answer
to "does this prop exist and can I read it?" — what `EventApi.on`
needs to distinguish "no such event / not allowed to subscribe" from
"declared but no payload yet." See
[properties.md](./properties.md#checkprop--getallpropnames--generateuniquepropname).

### 3. `MixinApi.hasMixin` walks the shadow stack

The instance-form of `MixinApi.hasMixin(host, mixinName)` (and every
`MixinApi.isX(obj)` predicate that wraps it) walks shadows attached
to `host` first, falling back to the host's own class. A host
without `Mobile` but carrying a shadow that composes `Mobile` is
detected as Mobile.

The constructor-form `MixinApi.hasMixin(SomeClass, mixinName)` is
unchanged for back-compat.

---

## Open / deferred concerns

These are flagged for the future but not blocking event-system work:

- **Composable access checks** (chain of `checkAccess` functions on a
  single property). Not needed for events; flagged for future
  Propertied work if a use case appears.
- **callDown / `next` semantics for masks.** Masks chain via
  value-passthrough but can't short-circuit. Same — not needed for
  events; future Propertied concern.
- **Per-event-method emit allowlists** (e.g., only
  `PlayerApi.handleLogin` can emit, not `PlayerApi` as a whole). Defer
  until a real use case demands this granularity.
- **Subscription persistence** — explicitly NOT a goal. Subscriptions
  are runtime-only; if a future use case wants "tell me about events I
  missed while disconnected," that's a separate subsystem (outbox
  table, not the event bus).
- **Multiple buses / blocking bus variant.** Roadmap-level "extensible
  later." Internal structure (registries, dispatch, history) should be
  factored so a future blocking bus could reuse the machinery; do NOT
  expose a `createBus()` API on initial ship.
- **Topic-pattern subscriptions** (`EventApi.on('Module.*', ...)`).
  Not on initial ship; `filter` covers the use case adequately.

---

## What we explicitly decided NOT to do

- **Mini-mixins per individual signal method.** Rejected as
  duck-typing-with-extra-steps — the marker carries no information
  beyond "method exists" for single-method mixins.
- **Bundle mixins with all-required-abstract methods.** Rejected
  because it forces shadows that want to react to one event to
  stub-implement all the others.
- **Bundle mixins with optional methods + duck-typed dispatch.**
  Rejected because the marker becomes meaningless for dispatch.
- **Single `handleEvent()` dispatcher on receivers.** Rejected because
  it's non-shadowable per-event — shadows can only intercept the one
  method, masking the host's switch logic for ALL events.
- **New `PropOperation.Emit`.** Rejected because PropertiedMixin should
  stay universal; events shouldn't add a dimension to a mixin used
  everywhere else in the codebase.
- **Storing subscribers in EventRegistry props.** Subscribers live on
  the `EventSubscriptions` singleton, not in props.
- **Hot-reload re-running bootstrap.** Different concerns; hot-reload
  handles its own re-init story when it lands.
- **Dual-emit for movement** (firing both Witness hooks AND a global
  `Events.Moved` event). Force the shadow path; don't pay always-on
  emit cost for tracking that shadows handle better.
- **Object-local EventApi** (`obj.on('moved', fn)`). Object-local
  notifications are Witness-pattern only; the global bus is for
  no-target events.
- **Boolean-return-to-unsubscribe convention.** Subscription options
  (`until`, `once`) are explicit and not footguns.

---

## Implementation history

The subsystem shipped in seven phases:

1. **Framework prerequisites** — shadow callDown rule, `checkProp`
   Get-gating, `MixinApi.hasMixin` shadow-walk.
2. **Bootstrap & Seeding skeleton** — `SeederManager`,
   `BootstrapManager`, the `#indexes` struct and
   `StuffApi.findByTemplatePath` helper.
3. **EventApi surface + EventRegistry Idea** — emit / on / once /
   history, `EventDispatch` `FrameKind`, side-table dispatch with
   per-listener fresh frames, error isolation.
4. **EventRegistry seeded + bootstrapped** — first manifest entry,
   well-known events declared.
5. **Witness hook surface** — optional methods on Containable /
   Mobile / Container / Exitable / HasInteractive; the
   `setContainer` chokepoint refactor; renames; lockdown
   decorators on the containment trio.
6. **EventApi production wiring** — emit sites in `StuffApi`,
   `ConnectionApi`, `Application`, `PersistenceManager`.
7. **Documentation** — this revision; the bootstrap doc; updates to
   antipatterns, properties, roadmap.
