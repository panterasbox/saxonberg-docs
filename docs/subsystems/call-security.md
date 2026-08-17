# Call Security

A unified interception layer that sits between any caller and any method
on a `Stuff`, and answers four questions before the method runs:

1. **Is the target still alive?** (destroyed-object guard)
2. **Is this caller allowed to make this call?** (security policy)
3. **Is anything wrapping this method right now?** (shadow chain)
4. **What is the call chain that led here?** (execution context)

It also mediates a handful of *privileged* operations —
`Stuff.destroy()`, shadow installation, manager-level access — so the
only legitimate way to perform them is through the Api layer.

The implementation is built on **AsyncLocalStorage** for context
propagation, a **Proxy with a registered interceptor pipeline** for
enforcement, and a **module-level loader transform** for caller
identity.

## The Six Pillars

| Pillar | Built? | Where |
|---|---|---|
| 1. Async-safe call stack | Yes | `ExecutionContextApi` |
| 2. Destroyed-object guard | Yes | `SecurityApi` interceptor |
| 3. Policy enforcement | Yes | `SecurityApi` + `SecurityPolicies` |
| 4. Function shadowing | Yes | `ShadowApi` + `Shadow` |
| 5. Audit / logging | No (gap) | — |
| 6. Privileged-surface mediation | Yes | `@CallSecurity(ApiOnly)` etc. |

Pillar 5 is a known gap: the framework has the necessary attribution
machinery (caller identity, callstack, command context, causing-command
id) but no structured audit sink is wired in. `MudlogApi` exists but
isn't wired to deny / shadow events. See "Audit (unimplemented)" below.

## Conceptual Model

Three things are always in scope when a guarded method runs:

```
   caller          target            method
  ┌──────┐       ┌────────┐         ┌─────────┐
  │ Stuff│──────▶│  Stuff │ . ─────▶│   fn()  │
  └──────┘       └────────┘         └─────────┘
     │              │                    │
     │              │                    └─ the function being called
     │              └─ the Stuff whose method is being invoked
     └─ the Stuff that initiated this call (or null = "system")
```

The framework's job is to peek at all three (plus the rest of the call
stack), decide whether the call is allowed, run any installed shadows,
then either invoke the original method or throw.

**`null` caller = the root of the stack.** Every frame chain has a
bottom; the bottom frame's caller is `null` because there's nothing
calling into it from the runtime — it's an entry point. The framework
has two root-frame creators:

- **Backend** plants the root frame for inbound network/auth events
  via `Backend.processUserMessage`, `handleUserConnect`,
  `handleUserDisconnect`, and the bootstrap script. Each calls into
  `Application` through `runRoot(Backend, methodName, fn)`, so the
  root frame's `target` is `Backend` / the named method and its
  `caller` is `null`.
- **`ScheduleApi`** plants its own root frames when a scheduled
  callback fires. The original synchronous chain is gone, so the
  callback runs under a fresh Root frame; the originating
  `causingCommandId` is re-planted onto that frame's metadata (see
  "Command Attribution" below).

Backend can also appear higher up the stack — Application calls into
Backend regularly (e.g., to send messages). Those calls push normal
frames for Backend methods, which behave as **leaves**: Backend never
calls back into Application from inside one of those calls, so a
Backend frame never has game-logic work nested under it. Stack shape
under Backend's runRoot:

```
[bottom] Backend.processUserMessage          ← root frame, caller = null
         …game logic, mixin methods, command dispatch…
         Application.someMethod calls backend.sendMessageToSocket
[top]    Backend.sendMessageToSocket          ← appears as a leaf
```

`null` caller is **not** a general "system context" escape hatch. The
Hydrator is itself a `Stuff` and pushes its own frames (see
"Hydrator is not an exception" below). Policies decide whether `null`
is privileged or anonymous; the answer is usually "privileged at the
Backend → Application boundary or scheduler tick, denied everywhere
else."

## Responsibility on the Call Stack (Design Rationale)

The call stack ("who called whom right now, in JS function terms")
*does* answer responsibility questions for synchronous chains, which is
the vast majority of gameplay. The framework deliberately leans on
this: caller identity is exactly the immediate previous frame's target.

The framework exposes one stack walker:

- **`getCurrentCommandGiver()`** — the most recent `CommandGiver`
  (Avatar or NPC) whose `executeCommand` body is currently running.
  Returns `null` outside a command pipeline.

Earlier sketches included `getActingAvatar()` / `getResponsibleAvatar()`
("most-recent vs bottom-most human-driven Avatar in the chain") and
matching policies (`Admin`, `ByCommandGiver`, `ByActingAvatar`,
`ByResponsibleAvatar`). Those aren't on the roadmap. The synchronous
chain is what `getCurrentCommandGiver` and frame walking give you;
anything beyond that lives outside this framework.

**Synchronous chains.** Everything in a single command pipeline —
reactive triggers (room fires `monster.kill`), force commands, stacked
force, NPC chains nested inside human commands — is reachable via
`getCurrentCommandGiver` and frame inspection. The most recent command
issuer is exactly what `getCurrentCommandGiver` returns.

**Cross-cycle attribution.** Work that happens outside the synchronous
chain that issued the command needs explicit propagation. `ScheduleApi`
handles delayed effects (banana peel dropped now → NPC slips next tick)
by re-planting `causingCommandId` from the originating frame onto the
Root frame of the callback, so `getCurrentCausingCommandId()` still
surfaces the originating id (see "Command Attribution" below). Cases
that don't have wiring yet:

- **Prompts.** Command pauses for user input, JS function returns,
  call stack unwinds. When the response arrives, the command resumes
  on a fresh stack — no propagation is wired up.
- **Cross-actor messaging.** A sends a message; B's handler runs
  later. B's stack doesn't include A.

**Terminology.** "CommandGiver" = the *immediate* issuer of the
currently-running command (might be an NPC). "Driving player" usually
means an Avatar with a connected Interactive — narrow with
`MixinApi.isHasInteractive(obj)` (`HasInteractive` is the connection
abstraction — `Avatar` and `Login` both compose `HasInteractiveMixin`).
Note that "HasInteractive" is the broad set, not the in-world set:
operations targeting an avatar specifically should narrow with
`instanceof Avatar` (or via Character if NPCs need to be in scope).

## Caller Identity

Real-world policies do not want to be written against `instanceof
Avatar`. The user wants to write rules like:

> "Reject calls that originated from anything cloned from a template
> under `/domain/narnia/**`."

> "Reject calls from any subclass of the class loaded from
> `/mud/domain/narnia/NarniaLocation.ts`."

> "Allow calls from `Application`, `Backend`, or `Interactive`."

To support that, every `Stuff` carries — and every guarded call
exposes — a small bundle of identity facets:

| Facet              | Source                                              | Example                                  |
| ------------------ | --------------------------------------------------- | ---------------------------------------- |
| **stuffId**        | runtime nanoid, on every `Stuff`                    | `aB7…`                                   |
| **module URL**     | the file the class was actually imported from, captured by the loader transform | `/api/stuff#StuffApi`, `/domain/narnia/NarniaLocation#NarniaLocation` |
| **template path**  | the CMS template path (only if cloned)              | `/obj/Avatar/abc123`, `/domain/narnia/cair-paravel` |

**Module URL is captured by the import machinery, not declared by the
class.** A class that says `static __isApi = true` or sets a
`Symbol.for('saxonberg.api')` marker would let an attacker drop a
`FooApi.ts` into some random folder, set the marker, and inherit
privileged access. We don't want a flag; we want "this class actually
came from `mud/api/`."

**How it's captured.** A source transform (Vite plugin in dev/tests, a
Node loader hook in the running server) appends a single call to the
bottom of every `.ts` file under `mud/`:

```typescript
import { ModuleApi as __callSecModuleApi } from '<...>/api/module';
__callSecModuleApi.stamp(import.meta.url, { Foo, Bar });
```

The export names come from an AST walk of the module's `export class`
/ `export default class` / `export { X }` declarations.
`import.meta.url` is set by the loader, not user code, so a class can't
lie about its source file.

> **The two paths do not see the same source, and that asymmetry has
> already cost one production-only outage.** The Vite plugin reads *raw
> TypeScript*. The Node loader hook chains **last**, so it reads tsx's
> *compiled* output — where `export default class Foo` has been lowered
> to `class Foo {…}` + `export { Foo as default };`. The resolver behind
> the injected `{ default: … }` field understood the raw form and the
> `export default Foo;` form but not the compiled clause, so it emitted
> **no stamp at all** for every default-exported class in the running
> server, while stamping them perfectly under Vitest. Since `FromModule`
> **fails closed on an unstamped caller**, every policy naming a
> default-exported class silently denied in dev and production and
> passed in every test. It surfaced as `eval` being unable to run a
> single statement in the live game — `ScriptApi.compileSandboxed` is
> gated to `/lib/script/EvalScript` — while all 19 of `EvalScript`'s own
> unit tests passed.
>
> Two rules follow. **Test the transform against the source the LOADER
> sees**, not only the source the plugin sees
> (`services/loader/__tests__/default-export-stamp.test.js`). And treat
> "the gate denies only in the running server" as the *expected* shape
> of a stamping bug: fail-closed plus a test-only transform path means a
> stamping gap is invisible to the suite by construction.

`ModuleApi.stamp` reads `Error.stack` to find its immediate caller's
URL and rejects (silently — no throw) if `declaredUrl` doesn't match.
A malicious file can stamp its own classes (no escalation — they get
stamped with their real URL) but can't stamp another file's classes or
declare a fake URL for its own. Stamps are kept in
`WeakMap<Class, ModuleId>` so unloaded classes get GC'd naturally.

**First-stamp-wins** for a given class. Subsequent stamps for the same
class are no-ops; hot reloads that re-evaluate a module bind the new
class under the same URL string.

The constructor-name fallback was considered and dropped: it's
forgeable. If a class has no module-id stamped (e.g., loaded before the
loader hook installed), identity-keyed policies fail closed —
`ModuleApi.lookup` returns `null` and `FromModule` / `FromTemplate`
deny the call.

**Canonical form.** The "module ID" stored is `path#exportName`,
covering both named and default exports:

| Class                                                 | Module ID                                     |
| ----------------------------------------------------- | --------------------------------------------- |
| `class Foo` exported as `Foo`                         | `/lib/Foo#Foo`                             |
| `class Foo` exported as `default`                     | `/lib/Foo` (bare path is the default-export convention) |
| `class StuffApi` in `mud/api/stuff.ts`                | `/api/stuff#StuffApi`                      |
| `class NarniaLocation` in `mud/domain/narnia/...`     | `/domain/narnia/NarniaLocation#NarniaLocation` |

`ModuleApi.lookup(cls)` returns the canonical id (or `null` if
unstamped). Module IDs are `/`-absolute, mud-rooted — the **same shape**
as the clone-namespace template path they parallel. `SecurityPolicies`
resolves the caller's two identities **independently**:
`resolveModuleId` (the caller's class module ID — code provenance) and
`resolveTemplatePath` (the caller's instance template path — clone
lineage). `FromModule` matches the former, `FromTemplate` the latter —
they discriminate on **which identity a policy reads**, not on string
shape. A cloned Stuff carries both, so `FromModule('/obj/…')` admits it
by its class even though it also has a template path.

## The Interception Pipeline

`ProxyApi` owns the get-trap mechanics; `SecurityApi` plugs into it.
The two are deliberately separated: `ProxyApi` knows about descriptors,
wrapper caching, and pipeline ordering, but nothing about security
policies. `SecurityApi` registers a single `#`-private interceptor
against `ProxyApi` at module-load time and provides every other
framework concern (destroyed-object guard, policy lookup, shadow
dispatch, frame push) inside that one interceptor.

### `ProxyApi.wrap`

Every Stuff that goes through `StuffApi.create` / `clone` /
`createSync` is wrapped:

```typescript
const proxy = ProxyApi.wrap(rawStuff);
```

The handler installs a `get` trap. There is **no `set` trap in v1** —
property writes pass through unchanged.

Two passthrough escape hatches:

- **`RAW_TARGET`** (`Symbol.for('saxonberg.proxy.rawTarget')`) — reading
  this symbol off the proxy returns the underlying Stuff. The
  introspection seam tests use; production code should reach for it
  rarely. `ProxyApi.unwrap()` is the typed wrapper.

- **`PASSTHROUGH_KEYS`** — a small set of property names that bypass
  the interceptor pipeline entirely:
  - `then` (would break Promise resolution)
  - `constructor` (mixin introspection must see the raw class)
  - `stuffId` (instance field the framework itself reads constantly;
    mediation would add noise)
  - `host`, `interceptedMethods` (shadow-side framework getters;
    mediation would dispatch them through the shadow chain — wrong)
  - `RAW_TARGET`

  Note: `zone` and `templatePath` are **not** passthrough keys —
  the ref-shapes lockdown made them hard-private (`#zone` /
  `#templatePath`). Access goes through `getZone()` /
  `getTemplatePath()`, not bracket reads off the proxy.

### Wrapper cache

The handler maintains a per-target `WeakMap<Function, Function>` so
repeated reads of the same method return the same wrapper:

```typescript
obj.foo === obj.foo   // true — same wrapper, not two new ones
```

This prevents identity-comparison patterns like
`removeEventListener(handler)` from breaking, and lets vitest-style
mock identity work.

### Mock-spy passthrough

Vitest / Jest spies carry an `_isMockFunction` flag. Wrapping them in
the interceptor pipeline breaks `toHaveBeenCalled` because the test
sees the wrapper, not the spy. The handler short-circuits when the
property's value carries that flag — a method swapped for a spy is the
test's intentional escape hatch.

### `Interceptor` and `InterceptionContext`

```typescript
export interface InterceptionContext {
  readonly target: Stuff;           // the raw Stuff
  readonly proxy: Stuff;            // the wrapping Proxy
  readonly prop: string;            // method/property name
  readonly args: readonly unknown[];
  readonly isGetter: boolean;       // distinguishes getters from method invocations
}

export type Interceptor = (
  ctx: InterceptionContext,
  next: () => unknown
) => unknown;
```

`registerInterceptor(fn)` appends to the pipeline; interceptors run in
registration order. Calling `next()` advances to the next interceptor,
or — at the end of the chain — runs the raw operation
(`descriptor.get` for getters, `function.apply` for methods).

The pipeline is the cross-cutting extension point: a future tracing
interceptor that logs every call, an instrumentation interceptor that
timestamps invocations, etc., plug in here without touching the
framework's core.

### `SecurityApi.installInterceptor`

`SecurityApi`'s static initializer calls `installInterceptor()` at
module-load time. The registered interceptor is `#securityGate`, which
runs (in order) per dispatch:

1. **Bypass marker check.** `ShadowApi._consumeBypass()` returns
   `true` exactly once after `Shadow.callBypass()` or `callDown()` at
   the bottom of the chain set it; the interceptor returns `next()`
   immediately, skipping every check below. Single-shot, consumed
   atomically.
2. **Destroyed-object guard.** `target.isDestroyed()` throws
   `DestroyedObjectError`. `isDestroyed` and `toString` are exempt so
   error formatters and the destroyed-state read invariant don't trip
   it.
3. **Entry policy.** `SecurityApi.resolveCallPolicy(target, prop)`
   walks the prototype chain (closest method-form `@CallSecurity`
   wins, then class-form, then Public). The current target
   (`ExecutionContextApi.getCurrentTarget()`) is the caller. A `false`
   result throws `SecurityError`.
4. **Shadow dispatch.** If `ShadowApi._shadowsFor(proxy, prop)`
   returns shadows, dispatch through them via `_withDispatch`. The
   chain is a complete replacement for the raw call — `next()` is not
   called in this branch.
5. **No shadows.** Push the host's frame via `ExecutionContextApi.run`
   and call `next()` to continue the pipeline.

The shadow lookup is keyed by `proxy`, not `target`: `ShadowApi.attach`
stored the proxy, so the lookup must use the same identity.

## ExecutionContextApi

Pillar 1. Carries a stack of `CallFrame`s through `await`,
`setTimeout`, and `Promise.then` boundaries via `AsyncLocalStorage`, so
any guarded method body can ask "who called me?" without threading a
context parameter through every signature.

```typescript
interface CallFrame {
  caller: unknown | null;
  target: unknown | null;
  method: string;
  timestamp: number;
  kind?: FrameKind;
  metadata?: Record<string, unknown>;
}

type CallStack = readonly CallFrame[];
```

`caller` and `target` are `unknown` rather than `Stuff` to avoid an
import cycle; the framework treats them as opaque object identities for
stack walking.

### `FrameKind` taxonomy

A small vocabulary of frame *kinds* marks the meaning of specific
frames so stack walkers can find them without string-matching method
names. Most frames carry no kind; only the ones that participate in a
known cross-frame contract do.

```typescript
export const FrameKind = {
  Root: 'root',
  Constructor: 'constructor',
  Command: 'command',
} as const;
```

- **`Root`** — synthetic frame planted by `runRoot` at a network →
  Application boundary. `caller = null`.
- **`Constructor`** — synthetic frame planted by `StuffApi.create` /
  `clone` around hydrate + `postRegister` (see
  [lifecycle.md](./lifecycle.md)).
- **`Command`** — tagged by `CommandGiverMixin.executeCommand` so
  `getCurrentCommandGiver` can find it.

Tagging happens one of two ways:

1. **At push time.** Synthetic frames the framework plants
   (`runRoot`, `StuffApi`'s constructor wrap) get their kind via the
   `kind` option to `run` / `runRoot`.
2. **After the fact via `tagCurrentFrame(kind)`.** Frames the proxy
   already pushed for a real method invocation can be re-tagged from
   inside that method's body. This is what
   `CommandGiverMixin.executeCommand` does — the proxy already pushed
   an `executeCommand` frame; the body just labels it.

`findFrame(kind)` walks the stack top-to-bottom looking for the most
recent frame with that kind — the generic primitive that all
kind-specific helpers (`getCurrentCommandGiver`, etc.) wrap.

### `run` vs `runRoot`

```typescript
ExecutionContextApi.run(caller, target, method, opts, fn);
ExecutionContextApi.runRoot(target, method, fn);
```

- **`run`** pushes a frame on top of the existing stack (or creates a
  fresh stack if none exists). Used by the Proxy when intercepting a
  method call (no opts), and by `StuffApi.#registerAndInit` for the
  synthetic constructor frame (`{ kind: FrameKind.Constructor }`).
- **`runRoot`** plants a synthetic Root frame whose `caller` is `null`.
  Distinct from `run` so the call-site intent is unambiguous: this is
  "I am the boundary, plant a root here," not "I'm pushing a frame on
  whatever happens to be on the stack." Used by Backend at the network
  → Application boundary, and by `ScheduleApi` when a scheduled
  callback fires.

### Read methods

```typescript
ExecutionContextApi.getCaller(): unknown | null;
ExecutionContextApi.getCurrentTarget(): unknown | null;
ExecutionContextApi.getCallStack(): CallStack;
ExecutionContextApi.dumpCallStack(): string;     // pretty-printed; do not parse
ExecutionContextApi.assertCaller(SomeClass): void;
ExecutionContextApi.findFrame(kind: FrameKind): CallFrame | null;
ExecutionContextApi.getCurrentCommandGiver(): unknown | null;
ExecutionContextApi.getCurrentCommandContext(): CommandContext | null;
ExecutionContextApi.getCurrentCausingCommandId(): string | null;
```

### Frame-mutator allowlist

`run`, `runRoot`, `tagCurrentFrame`, and `updateCurrentFrameMetadata`
mutate the call stack — the core trust surface every policy keys off.
Forging a frame is the same threat model as forging a module-id: an
attacker who can plant a frame whose `target` is an admin Avatar, or
whose `kind` is `Root`, sidesteps every policy.

Defence: stack-walk the immediate caller's file URL on entry; reject
if the URL doesn't match the framework allowlist:

```typescript
const _frameMutatorAllowlist: ReadonlyArray<RegExp> = [
  /\/mud\/lib\/security\//,                      // the framework itself
  /\/mud\/api\//,                                // every Api class
  /\/backend\//,                                 // Backend's runRoot
  /\/mud\/lib\/command\/CommandGiver\.(ts|js)$/, // CommandGiver tags Command frames
  // Singleton Stuff registries that plant root frames for their
  // ApiOnly-gated downstream calls (timer/event-listener dispatch).
  /\/mud\/obj\/(EventSubscriptions|MqlSubscriptionRegistry|SchedulerRegistry|WorldClockRegistry)\.(ts|js)$/,
  /\.test\.(ts|js)$/,                            // test seam
];
```

Any mismatch throws `SecurityError`. If `Error.stack` doesn't yield a
URL the call **fails closed** — better to break the call than silently
allow a forge. Per-URL cached so warmed-up cost is one Map lookup.

New framework files that legitimately need to push frames must be
added here with a one-line note in code review. Keep the list narrow.

## Static-method Api Wrapping

Api classes (`StuffApi`, `ContainmentApi`, etc.) are static utility
modules — there's no instance to push. The framework synthesizes a
frame for them when a call enters an Api function.

`SecurityApi.decorateApiClass(cls)` walks every own static on `cls`
and wraps each one via `_wrapStaticDescriptor`. The wrapper:

1. Resolves the static call policy via
   `resolveStaticCallPolicy(cls, methodName)`.
2. Checks `policy.allows(caller, cls, methodName)` — caller is the
   current `getCurrentTarget()`, target is the Api class itself.
3. On allow, pushes a frame via `ExecutionContextApi.run(caller, cls,
   methodName, undefined, () => original.apply(this, args))`.
4. On deny, throws `SecurityError`.

Decoration is a **module-scope tail** — each `api/*.ts` facade ends with
`SecurityApi.decorateApiClass(FooApi)`. This is one of the two sanctioned
exceptions to the no-module-scope-statements rule (see
[architecture.md § Module scope declares](../architecture.md)): an `*Api`
class is a thin, non-HMR-able *interface* imported directly, so the
module tail IS its registration — there is no lifecycle for it to join
the world through. The four bootstrap-special Apis
(`execution-context`/`module`/`security`/`proxy`) omit the tail (see
§ Why Some Api Files Don't Self-Decorate below). (The 2026-07 sweep
briefly routed decoration through a `ModuleApi.stamp` hook to avoid the
module-scope call; that was reverted.)

Wrappers carry a `_callSecWrapped` marker so re-decoration is a no-op.
The class-form `@CallSecurity` decorator does the same thing under the
hood (sets the policy, then calls `_wrapStaticMethods` to do the
descriptor wrapping).

## The api↔logic-singleton recipe

The surface-architecture refactor relocated each convertible Api's
*logic* into a stateless `Stuff` **logic singleton** at
`/obj/api/<feature>` (e.g. `MaterialLogic` at `/obj/api/material`); the
`FooApi` statics became thin forwarders. The HMR side lives in
[hot-reload.md](./hot-reload.md); the **gating** side is here, because it
exercises a specific caller/callee shape.

### The caller/callee shape (verified)

The forward is a **static-Api caller → instance-singleton callee**. Both
halves are supported:

- **Callee** is a proxy-wrapped `Stuff` instance method — the ordinary
  gateable shape.
- **Caller** is the static Api class. `resolveModuleId`'s function branch
  (`if (typeof caller === 'function') return ModuleApi.lookup(caller)`)
  treats a static-method synthesised frame's class as a first-class
  caller identity, and `decorateApiClass`'s static wrapper runs the
  forwarder body with frame **target = the Api class**. So when
  `MaterialApi.compositionOf()` calls `logic().compositionOf(...)`, the
  singleton sees `MaterialApi` as its caller.

Each logic method therefore carries
`@CallSecurity(FromModule('/api/<feature>#<Feature>Api'))` — the Api
forwards through, anything else is denied.

### Gate per method, not at class level

The gate is applied **per public method**, not as a class-level default.
A class-level default would also cover the inherited `Stuff`/`Idea`
framework methods (`getTemplatePath`, `isDestroyed`, …) that the
*framework itself* invokes during `register()` and lifecycle — whose
caller is `StuffApi`, not `FooApi`, and which the gate would then deny.
The per-method gate leaves inherited framework methods on their own
policies. (This corrects the original plan, which set the gate once at
class level — that broke `register()`.)

### Logic singletons are `@Unshadowable`

Relocating the logic onto a `Stuff` introduces one surface the former
static Api did not have: a `Stuff`'s methods can be intercepted via
`ShadowApi.attach`, whereas a static method cannot be shadowed. So every
logic singleton carries **class-level `@Unshadowable`** — the dispatch /
query / registry logic behind an Api face cannot be shadowed. The
class-form mark covers every method (own + inherited): `isMethodUnshadowable`
walks the prototype chain checking `#classUnshadowable`. `@Final` is *not*
applied — it's method-only, the logic classes are never subclassed, and
injecting a subclass into `singletonSync`'s factory already requires
code-execution access, so it would add churn without a real threat
reduction. (State integrity is a non-issue independently: the singletons
keep their state in module-scope `const`/`let`, not instance fields, so
it's unreachable — equivalent to `#` — and the singleton instance itself
is reached only by its own face via `singletonSync`, never by production
code path-lookup.)

### The intra-singleton self-call gotcha

A former static that called another static can't become a bare-gated
`this.x()`: inside the singleton, the caller of `this.x()` *is* the
singleton (its `/obj/api/<feature>` template path), **not** the Api — so
`FromModule(own Api)` denies it. Two fixes, in preference order:

1. **Extract a module-private free function** both methods call. Free
   functions are off-class and un-gated, but un-callable from outside the
   module, so they carry no protection cost. This is the default — most
   shared sub-logic is stateless (see `MaterialLogic`'s
   `computeComposition` / `everyMaterial`).
2. **Gate the method `AnyOf(FromModule(own Api), SelfOnly)`** when the
   helper genuinely must be an instance method needing `this`. `SelfOnly`
   (caller === target) passes for `this.x()` since the caller and target
   are both the singleton.

`#`-private instance methods are **not** an option on a `Stuff`: the
call-security proxy makes `this.#x()` throw (the slot lives on the raw
target, not the proxy) — hence the free-function route.

### The `ApiOnly` widening

A logic singleton's caller identity resolves to its `/obj/api/<feature>`
**template path**, not a `mud/api/` module id. So `ApiOnly`-gated
downstream calls (e.g. `ContainmentApi.move`, `placeDirect`) made *from*
a logic singleton would fail the old `FromModule('/api/**')` matcher.
`ApiOnly` was therefore widened (`lib/security/SecurityPolicies.ts`) to:

```
FromModule('/api/**', { includeSubclasses: true })  OR  FromTemplate('/obj/api/**')
```

`/obj/api/` holds *nothing but* logic singletons, so the new arm admits
exactly the Api tier and never content — it only **adds** admitted
callers; every prior allow/deny decision for non-logic callers is
unchanged.

### Two-singleton (stateful) Apis

Where an Api has state, a stateless logic singleton sits **between** the
Api statics and the pinned state singleton (the state survives reload;
the logic `dest`-reloads). Inserting the layer **re-points the state
singleton's caller** — its `FromModule('/api/<feature>#<Api>')` gate
becomes `AnyOf(FromModule(...), FromTemplate('/obj/api/<feature>'))`.
Adoption:

- **`scheduler`** (`SchedulerRegistry`), **`worldclock`**,
  **`mql-subscription`** (pinned registries) — existing gate re-pointed.
- **`soul`** (`SoulCatalogue`) had **no** gate; one was *added* (with a
  `SelfOnly` arm for its `postRegister`→`warmCache` self-calls).
- **`access`** (`AccessRegistry`), **`group`** (`GroupRegistry`),
  **`belief`**, **`prompt`** (per-Interactive registry) were converted in
  the bootstrap-cycle-unblock pass (below); their backing-state gates
  were re-pointed to `AnyOf(FromModule(...), FromTemplate('/obj/api/<feature>'))`
  where the state object carried a `FromModule` gate (`AccessRegistry`),
  or left alone where it didn't (`GroupRegistry` is ungated).

Special per-method Api gates (e.g. `SystemRoot` on
`WorldClockApi.boot/shutdown`) stay on the Api forwarder.

### The Api ↔ logic-singleton split is the hot-reload boundary

The two hops are **not** an optional optimization — they *are* the
hot-reload boundary, and the split is mandatory for every substrate Api.
The `XApi` facade is imported directly across the codebase, so it is
**not** HMR-able: anything living on it is frozen until a full restart.
The `XLogic` singleton lives at `/obj/api/<feature>` and is resolved
fresh per call (`StuffApi.singletonSync` + `HotReloadApi.getCurrentExport`),
so editing it hot-reloads into every caller.

Do **not** collapse the tier on the theory that "the logic only forwards
to a registry, so it earns nothing." A logic singleton that resolves a
state registry still owns the resolution, caching, and fail-open/
fail-closed policy — and even a genuine pure-forward tier must stay, so
that logic *added later* lands in the HMR-able unit rather than on the
frozen facade. State registries (`AccessRegistry`, `OfficeRegistry`) hold
durable *state* and are gated to admit their logic singleton
(`FromTemplate('/obj/api/<feature>')`) — they are not a substitute for
the logic tier. The 2026-07 sweep briefly collapsed `access`/`office`
this way; it was **reverted**. The method-level cut that sweep *should*
have made — deleting empty public predicates off the Api *surface* in
favor of Stuff-to-Stuff contracts — is a separate, correct move (see
[antipatterns.md § Thin Api Wrappers over Object Methods](../antipatterns.md)).

### Override hooks are ungateable — `@hook`, not policy

The recipe gates everything the consumer *calls*. There's a second tier
the author *implements* and the framework *invokes* — override hooks
(`onDestruct`, `canDestruct`, `postRegister`, `aroundSave`/`aroundDelete`,
`onLinkdead`, `save`, the Hydrator `apply<Field>` appliers). These are
**public, ungated, and ungateable**: a subclass's `super.onDestruct()`
is *author code calling the hook*, so a `FromModule(framework)` gate
would deny the legitimate super-chain. The consumer/extension split is
therefore **not** policy-derivable for these — they're marked with a
human-placed TSDoc **`@hook`** tag instead (see the doc-projection in
the doc-gen section of CLAUDE.md). This is the one
spot where *callable == visible == cared-about* needs a marker rather
than falling out of the gate.

### What stays a static class (not converted)

The bootstrap-import-cycle set (any `api/` face value-reachable from
`lib/stuff/Stuff.ts`, where `class XLogic extends Idea` would run while
`Idea` is still `undefined`) was **unblocked and converted** — the
keystone `shadow → command` value-edge was severed via a late-bound DI
seam (`ShadowApi._registerCommandShadowHook`), which freed
`array, belief, command, command-line, containment, group, message, mql,
mudlog, prompt, prose, quantity, recognition, shell, access` for
conversion.

What remains a **static class** after that:

- **Bootstrap-special (6):** `security`, `module`, `proxy`,
  `execution-context`, `stuff` (hosts `singletonSync`), `mixin` — the
  substrate every Idea is built from.
- **`event`** — structurally impossible to convert: `EventApi.emit`
  fires inside `StuffApi.create`/`destruct` on every Stuff creation, so a
  logic singleton would recurse at its own `singletonSync`→`createSync`→
  `register`→`emit` creation.
- **`hot-reload`** — the singleton machinery itself calls
  `HotReloadApi.getCurrentExport`; a logic singleton can't depend on the
  thing that builds logic singletons.
- **`shadow`** — the method-dispatch shadow substrate (every proxied
  call routes through it).
- **`path-pattern`** — a `lib/` glob primitive, not an Api shell.
- **`grammar`** — convertible only behind a DI seam on the core
  `lib/stuff/Stuff.ts` (`getPresentation`→`pluralize`); deferred as not
  worth destabilizing the hottest core method for a small utility.

`mml` is **not** in this list because `Mml` is a value class (factory
statics + instance methods, like `Quantity`/`Prose`), not an `*Api`
forwarding shell — there is no `MmlApi` to convert.

`schedule` is a deliberate **partial** (see below).

`schedule` is a deliberate **partial**: its surface is frame-mutator-
bound (`planRun` → `ExecutionContextApi.runRoot` must stay in `mud/api/`
per the [frame-mutator allowlist](#frame-mutator-allowlist)) and its
timers are per-handle closures (no shared state to pin), so it stays
Api-static.

## Why Some Api Files Don't Self-Decorate

Four Api classes deliberately skip `SecurityApi.decorateApiClass` on
themselves: `ExecutionContextApi`, `ModuleApi`, `SecurityApi`, and
`ProxyApi`. Each has class-specific reasons rather than one shared
rule. Per-method `@CallSecurity` is a possible future granularity (the
audit below identifies one candidate) but isn't done today — the
bootstrap timing for individual decorators on these four classes is
non-trivial.

### `ExecutionContextApi`

Every method on this class either reads or modifies the call stack via
`AsyncLocalStorage`. Wrapping any of them creates **stack pollution**:
the wrapper's own `ExecutionContextApi.run(…)` call pushes a frame, so
when the original method body inspects the stack it sees that synthetic
frame — `getCurrentTarget` returns `ExecutionContextApi` instead of the
true current target, `getCallStack` includes itself, and so on.
`run`/`runRoot` additionally cannot be wrapped without recursion (the
wrapper IS what calls `run`).

Verdict: **none of its methods can be decorated.**

### `ModuleApi`

- `lookup(cls)` — called by every policy that uses `FromModule` /
  `FromTemplate` / `ApiOnly` to determine the caller's module ID.
  Wrapping causes recursion: policy resolution → calls `lookup` →
  wrapper resolves a policy for `lookup` → calls `lookup` again.
- `stamp(cls, moduleId)` — called by the loader transform during
  module evaluation. Decoration would push a frame at every module
  load. Possible in principle but adds boot-time overhead and there's
  no useful policy today (the loader transform lives in
  `services/loader/`, outside the `mud/api/**` scope `ApiOnly` covers).
  Future: a "loader-only" policy could justify wrapping `stamp`, but
  that policy doesn't exist yet.
- `_*ForTest` methods — gated by `assertTestOnly` already; wrapping
  adds a Public-default policy check and a frame push, neither of
  which adds value.

Verdict: **all undecorated for now**, but `stamp` is a candidate when
a "loader-only" policy is introduced.

### `SecurityApi`

This class IS the wrapping engine. Its methods fall into three roles
all incompatible with self-wrapping:

- **Wrapping infrastructure** (`decorateApiClass`, `_wrapStaticMethods`,
  `_wrapStaticDescriptor`). The wrappers themselves call these — loop.
- **Resolvers** called by every wrapped method on every Api class
  (`resolveCallPolicy`, `resolveStaticCallPolicy`,
  `isMethodUnshadowable`, `resolveShadowSecurity`, `getFinalMethods`).
  Wrapping creates a loop on every guarded call.
- **Decorator-stampers** (`_setMethodPolicy`, `_setClassDefaultPolicy`,
  `_markMethodUnshadowable`, etc.) called during decorator evaluation
  at class-definition time. At that point `decorateApiClass` may not
  even be reachable yet, depending on import order.

`assertTestOnly` and `_*ForTest` could in principle be wrapped (they
aren't called by the framework itself), but wrapping a test seam to
add a Public-default policy + a frame push has no upside.

Verdict: **all undecorated.**

### `ProxyApi`

- `wrap(stuff)` — called by `StuffApi.create`/`clone`/`createSync`
  during construction to install the proxy. Wrapping the install
  function with the same proxy machinery it's installing is a
  conceptual recursion; even if it didn't loop, it would need a proxy
  to mediate the wrapping function.
- `unwrap`, `_resetInterceptorsForTest` — narrow utility / test seams.
  Wrapping adds nothing.
- **`registerInterceptor(interceptor)`** — only called from
  `SecurityApi.#securityGate`'s own installation block. A real
  candidate for individual `@CallSecurity` (e.g.,
  `FromModule('/api/security')`) — anything else registering an
  interceptor can bypass the entire security gate. Held off because
  the bootstrap timing is fiddly: `proxy.ts` and `security.ts` import
  each other and resolve via late binding; adding a decorator that
  calls `SecurityApi._setMethodPolicy` at proxy.ts class-definition
  time risks the binding not being usable yet.

Verdict: **`wrap`, `unwrap`, `_resetInterceptorsForTest` undecorated**;
**`registerInterceptor` should be locked down** but is deferred until
the bootstrap timing is verified safe.

### Summary

These four classes are deliberately bare today. The audit confirms
they're warranted. Per-method decoration is a future option for two
specific cases:

- `ModuleApi.stamp` — needs a "loader-only" policy first.
- `ProxyApi.registerInterceptor` — needs verified-safe bootstrap
  timing first; ideally a `FromModule('/api/security')` policy.

Everything else stays bare on principle: framework primitives that
mediate calls cannot themselves be mediated without recursion or stack
pollution.

## Test Seams

Every Api class exposes test-only seams (e.g.
`StuffApi._unregisterForTest`, `SecurityApi._classDefaultPolicyForTest`,
`ProxyApi._resetInterceptorsForTest`). Production code that
accidentally — or deliberately — reaches one bypasses framework
invariants.

`SecurityApi.assertTestOnly(op)` walks `Error.stack` for a
`.test.{ts,js}` frame anywhere in the call chain. If none is found, it
throws `SecurityError`. Required at the top of every `_*ForTest` /
`_*ForTesting` method on every Api class:

```typescript
public static _classDefaultPolicyForTest(cls: object): SecurityPolicy | undefined {
  SecurityApi.assertTestOnly('_classDefaultPolicyForTest');
  return SecurityApi.#classDefaultPolicies.get(cls as ClassKey);
}
```

The check is per-URL cached, so warmed-up cost is a Map lookup. The
allowlist is "any frame anywhere in the stack is in a test file" —
looser than `ExecutionContextApi`'s "immediate caller must match," but
right for test seams. Tests legitimately call framework code which
might call further framework code; what matters is that some test is at
the bottom of the chain, not that every frame between is a test.

## Built-in Policies

A `SecurityPolicy` is just `{ name, allows(caller, target, method,
args?) }` — the fourth parameter carries the **call arguments** (empty
for getter reads), threaded through by both dispatch sites (the
instance security gate passes `ctx.args`, the static Api wrapper its
`args`) and forwarded by the combinators. Policies that don't care
simply ignore it. `SecurityApi` resolves the policy attached to the
called method (per-method, then class-level fallback, then framework
`Public` default) and runs it before invoking the body.

| Policy | Allows |
|---|---|
| `Public` | Anyone, including untrusted modder code. The framework default when no decorator is present. |
| `SystemRoot` | `caller === null` only — the synthetic root frame planted by `runRoot`. |
| `SelfOnly` | `caller === target` — only the target can call itself. |
| `ApiOnly` | Sugar for `FromModule('/api/**', { includeSubclasses: true })`. |
| `FromTemplate(glob)` | Caller's **clone-instance template path** (`getTemplatePath`) matches `glob` — trust by clone lineage. A caller that isn't a cloned Stuff has no template identity and fails closed. |
| `FromModule(glob, opts?)` | Caller's **class module ID** (`ModuleApi.lookup` on the class) matches `glob` — trust by code provenance, independent of any template path. With `{ includeSubclasses: true }`, walks the prototype chain so any ancestor whose module ID matches passes. Module IDs are `/`-absolute (same shape as template paths); the two policies are told apart by which identity each reads, not by the slash. |
| `FromClass(() => Cls, opts?)` | **Participant policy** — the caller is an *instance* of the thunked class (subclasses included, ordinary `instanceof`). The thunk defers class resolution past import cycles, fail-closed if it throws. Cross-HMR-reload safety: falls back to comparing module ids along the caller's class chain against the thunked class's module id when `instanceof` misses (a reloaded class is a fresh identity; the module id is the stable one). `opts.where(caller, target, method, args)` adds the **relational half** of the contract — e.g. "the Party calling me must be writing *its own* path and already roster me". |
| `FromMixin(name, opts?)` | **Participant policy** — the caller *composes* the named mixin, checked by walking the caller's class chain for the `_mixinName` static marker (pure string identity: HMR-stable, import-cycle-free). Class composition only — a Shadow-granted mixin does not confer caller privilege. Same `opts.where` relational half as `FromClass`. |
| `FromController(...controllers)` | Sugar over `FromModule` keyed by a controller class's stamped module id. For one controller, lazy `FromModule(moduleIdOf(c))`. For many, `AnyOf(FromModule(idOf(c1)), …)`. The lazy form resolves `ModuleApi.lookup(cls)` at call-time, fail-closed if the class isn't stamped yet — handles the cyclic-import edge case where the controller class isn't stamped at decorator-evaluation time. The **narrow-entry pattern**'s policy half — see [access.md](./access.md). |
| `Custom(pred, name?)` | Wrap an arbitrary predicate `(caller, target, method) => boolean | Promise<boolean>`. |
| `AllOf(...)` | Composition: every policy passes. |
| `AnyOf(...)` | Composition: at least one policy passes. |
| `Not(p)` | Composition: invert. |

### Participant contracts — the preferred gate for object-owned surfaces

`ApiOnly` says "someone in the Api tier is calling" — a *module*
allowlist that tells you nothing about whether the call makes sense.
The participant policies (`FromClass` / `FromMixin` + `where`) instead
express the contract in the OO terms the object graph already speaks:
**which Stuff is on the other end of this call, and is it in the right
relationship to me for these arguments**. The exemplar is the party
subsystem: `PartyMemberMixin._setActivePartyPath` is gated on
`FromClass(() => Party)` with a `where` requiring that the path being
written is the *calling party's own* path and that the member is
already on that party's roster — the `Party.admit()` transition is the
code that satisfies it. A narrow `FromTemplate('/obj/api/<feature>')`
arm may ride along for the owning logic's janitorial cases (stale
state with no live participant to act).

New object-owned mutators should reach for a participant contract
first; `ApiOnly` on an object method is the legacy shape. The planned
expansion is **trust-layer policies** (ownership via `ParcelApi`,
authorship via `ProvenanceApi`, group membership via `GroupApi`) as
sibling policies of the same shape — `allows` is already
async-capable, so relationship-derived trust needs no new machinery,
and new code qualifies by relationship instead of by joining a module
list.

### Async `allows`

`SecurityPolicy.allows` returns `boolean | Promise<boolean>`. The
security gate detects the return shape and only takes the async
branch when a Promise comes back; existing sync policies continue
to run sync through the gate. The widening lets future policies do
async lookups (group membership, zone inheritance walks, etc.)
without forcing every existing call site through a microtask.

All identity-keyed policies (`FromTemplate`, `FromModule`, `ApiOnly`)
**fail closed** when the caller has no matching identity (`resolveModuleId`
/ `resolveTemplatePath` returns `null`) — a class that wasn't stamped by
the loader transform can never be trusted.

### Deferred policies

`Admin`, `ByCommandGiver`, `ByActingAvatar`, `ByResponsibleAvatar` are
specified in the design but **not implemented in v1** — no consumer in
the current codebase needs them, and building them speculatively means
designing against assumptions. The corresponding stack-walk helpers
(`getActingAvatar`, `getResponsibleAvatar`) are deferred for the same
reason.

## Decorators

Five decorators ship in v1. All live in
`mud/lib/security/decorators.ts` as thin wrappers over `SecurityApi`'s
metadata writers.

### `@CallSecurity`

Mandatory entry policy. Polymorphic.

```typescript
// Method form: mandatory policy for that method.
class Foo {
  @CallSecurity(SecurityPolicies.ApiOnly)
  destroy(): void { ... }
}

// Class form: default policy for unannotated methods on the class.
@CallSecurity(SecurityPolicies.Public)
class Application { ... }
```

Method-form always wins over class-form; class-form wins over the
framework `Public` default. The decorator inspects how many arguments
TypeScript handed it (3 = method, 1 = class) and forks behavior.

Class-form additionally calls `SecurityApi._wrapStaticMethods(cls)` so
static Api calls push frames — otherwise an inner `ApiOnly` check would
misfire (no Api frame on the stack to match against). Instance methods
are mediated by the Proxy at create-time and need no descriptor
wrapping.

The static-method form of `@CallSecurity` (3 args, `target` is a
class) calls `_wrapStaticDescriptor` to wrap that single descriptor.

### `@Unshadowable`

Forbid shadow attachment. Polymorphic — method or class form. Same
3-args-vs-1 fork as `@CallSecurity`.

```typescript
class Stuff {
  @Unshadowable
  isDestroyed(): boolean { ... }
}

@Unshadowable
class PropertiedMixin { ... }       // every method on the class is unshadowable
```

Class form is inherited: a class with `@Unshadowable` OR any of its
ancestors counts. There is no separate `@UnshadowableClass` — the
class form goes through `SecurityApi._markClassUnshadowable` via the
same decorator.

`ShadowApi.attach` consults both maps at attach time and throws
`ShadowError` on conflict.

### `@Final`

Method-only. Subclass overrides are caught at import time by the
loader-hook validator. See the "Final" section below.

### `@Shadowing`

Mark a method on a Shadow class as intercepting a host method.

```typescript
class TraceShadow extends Shadow {
  @Shadowing
  addXp(amount: number) {                  // intercepts host.addXp
    return this.callDown(amount);
  }

  @Shadowing('take')
  loggedTake(item: Stuff) {                // intercepts host.take
    return this.callDown(item);
  }
}
```

Stored on the Shadow class as a static `_callSecShadowing` Map keyed
by HOST method name → LOCAL method name. `ShadowApi`'s dispatcher
consults the map to find the actual method to invoke.

### `@ShadowSecurity`

Per-method gate on shadow attach / detach operations. Two forms:

```typescript
@ShadowSecurity(SecurityPolicies.SystemRoot)
sensitiveMethod() { ... }                  // attach AND detach

@ShadowSecurity({ attach: P1, detach: P2 })
sensitiveMethod() { ... }                  // independent
```

Read at attach/detach time by `ShadowApi`. Inheritance + resolution
mirror `@CallSecurity`: closest decorator on the prototype chain wins;
mixin defaults are fallback.

### Mixin defaults + class override

A mixin can annotate the methods it contributes, setting a default
policy; the implementing class can redeclare the method with its own
annotation to override. Three states are possible when a subclass
interacts with a mixin method:

1. **Don't redeclare** → mixin's policy applies.
2. **Redeclare with a new `@CallSecurity(...)`** → the new policy
   replaces the mixin's.
3. **Redeclare without a decorator** → mixin's policy *still* applies.
   The redeclaration overrides the body, not the security contract.
   To weaken or strengthen the policy you must redeclare *both* the
   method *and* its decorator.

This makes the contract safe by default: you cannot accidentally drop
security by overriding behavior. The framework's resolver walks the
prototype chain at dispatch time and uses the closest decorator it
finds (subclass first, mixin's as fallback, framework default of
Public only if nothing in the chain has one).

## `@Final`

`@Final` is not strictly a CallSec feature, but it rides on the same
loader-hook infrastructure and addresses an existing pain point:
methods marked "DO NOT OVERRIDE" by comment have no enforcement.
TypeScript has no `final` keyword. This decorator gives one.

### Purpose

Prevent subclasses from overriding a method declared `@Final` on an
ancestor. The motivating case is `Stuff.destroy()`, whose correctness
invariant ("never override; always unregisters") was previously a
comment.

### Storage

```typescript
SecurityApi._markFinalMethod(cls, propertyKey);
// → #finalMethods: WeakMap<ClassKey, Set<string>>
```

The decorator stamps method names under the declaring class. Read by
`ModuleApi.#validateNoFinalOverrides` (the loader-hook validator).

### Enforcement timing

**Class load time, via the loader transform.** When `ModuleApi.stamp`
binds a newly-imported class into the module-id `WeakMap`, it also
runs `validateNoFinalOverrides(cls)` on it. If validation fails, the
loader throws — which means **the import itself throws**. The bad
subclass never makes it into the module graph; no instantiation is
needed for the violation to surface. CI catches it even for classes
that aren't constructed in tests.

This is preferred over instance-creation-time enforcement (which would
only fire when someone actually `new`s the offending class).

### Validation algorithm

```typescript
static #validateNoFinalOverrides(cls: object): void {
  if (typeof cls !== 'function' || !('prototype' in cls)) return;
  const ctor = cls as { name?: string; prototype: object };
  let proto = ctor.prototype;
  while (proto && proto !== Object.prototype) {
    const ancestor = Object.getPrototypeOf(proto);
    if (!ancestor || ancestor === Object.prototype) break;
    const ancestorCtor = ancestor.constructor;
    if (typeof ancestorCtor === 'function') {
      const finals = SecurityApi.getFinalMethods(ancestorCtor as object);
      if (finals) {
        for (const name of finals) {
          // Walk every layer between cls.prototype and ancestor; if
          // any layer has its own descriptor for `name`, it overrides
          // the ancestor's @Final.
          let walker = ctor.prototype;
          while (walker && walker !== ancestor) {
            if (Object.hasOwn(walker, name)) {
              throw new FinalViolationError(
                ctor.name ?? '<anonymous>',
                `${ancestorCtor.name ?? '<anonymous>'}.${name}`
              );
            }
            walker = Object.getPrototypeOf(walker);
          }
        }
      }
    }
    proto = ancestor as object;
  }
}
```

Multi-level inheritance is handled correctly: `A` declares `@Final
foo`, `B extends A` doesn't override, `C extends B` overrides `foo` —
`C`'s import throws because the walker reaches `A` and finds `foo`
marked final there.

### Independence from other decorators

`@Final`, `@Unshadowable`, and `@CallSecurity` are orthogonal. Compose
freely:

| Decorator | Prevents | Checked when | Concerns |
|---|---|---|---|
| `@Final` | Subclass redefining the method | Class load | Class hierarchy |
| `@Unshadowable` | Runtime shadow attaching to the method | Shadow attach | Runtime composition |
| `@CallSecurity(p)` | Unauthorized callers invoking the method | Each call | Access control |

`destroy` needs all three — the override is structural, the shadow is
runtime, the caller is authorization.

### Caveats

- **Doesn't prevent runtime monkey-patching.** Someone can still do
  `Stuff.prototype.destroy = (...) => …` from JS. That's a
  deliberately hostile act, not an accidental override. The goal of
  `@Final` is to catch *honest mistakes* (subclass authors not
  realizing they shouldn't override) and *test-time accidents*, not to
  defeat malicious code that already has full JS access.
- **Doesn't prevent shadowing.** A shadow can still intercept a
  `@Final` method (unless `@Unshadowable` is also applied) — shadows
  aren't class hierarchy modifications, they're runtime interception.
- **Static methods aren't covered.** Same rationale as the rest of the
  framework — instance behavior only.

## Shadow Subsystem

A shadow modifies behavior on a *single instance* without subclassing.
Multiple shadows on the same method form a stack, last-installed runs
first. Each shadow can:

- modify args before calling `callDown`
- skip `callDown` to fully replace behavior
- wrap `callDown` in try/catch
- mutate the return value

**Shadows are `Stuff`.** Not interfaces. They get their own `stuffId`,
lifecycle (`destruct()` removes them from every target they're attached
to), can be cloned from templates, and live in the class hierarchy as
a top-level branch alongside `Idea` and `Agent`:

```
Stuff
 ├── Idea (User, Player, Location, Door, …)
 ├── Agent (Avatar, NPCs, …)
 └── Shadow                       ← top-level
      ├── XpDoublerShadow
      ├── TraceLoggerShadow
      └── …
```

Buffs/debuffs are cloneable game objects, not throwaway closures. A
shadow can be inspected via `MqlApi`, listed in admin tools, etc.

### Class shape

```typescript
abstract class Shadow extends Stuff {
  /** The single host this shadow is attached to, or null if unattached.
   *  Read-only — the framework owns the binding. */
  public get host(): Stuff | null;

  /** Set of host method names this shadow currently intercepts. */
  public get interceptedMethods(): ReadonlySet<string>;

  /** Invoke the next thing down the dispatch stack. Args explicit. */
  protected callDown<T = unknown>(...args: unknown[]): T;

  /** Run a method on this shadow's host bypassing ALL shadows
   *  (including this one). Privileged. Throws if host is null. */
  protected callBypass<T = unknown>(method: string, ...args: unknown[]): T;
}
```

Both directions of the host↔shadow link live in framework-private
WeakMaps inside `ShadowApi`:

```typescript
static #hostShadows: WeakMap<Stuff, Map<string, Shadow[]>>
static #shadowHost:  WeakMap<Shadow, Stuff>
static #shadowMethods: WeakMap<Shadow, ReadonlySet<string>>
```

User code reads `shadow.host` through a read-only getter; writes are
impossible (no setter, no field). The framework module is the *only*
code that can mutate either map, and it always mutates both in the
same call within `attach`, `detach`, and the privileged-detach used
during destruct. There is no API surface — public, protected, or
otherwise — that would allow the two sides to drift out of sync.

### Authoring shape — explicit declaration declares the surface

A Shadow's intercept set is whatever **methods** it explicitly declares
in its own class body, plus any `@Shadowing`-decorated methods.
Composition alone does NOT auto-enrol the composed mixin's methods —
they're part of the shadow's *type contract*, not its *intercept
set*. **Accessor pairs (`get foo()` / `set foo()`) are also excluded**:
accessors are host-internal implementation tools (invariant
maintenance, sync/async splits) and aren't part of the inter-stuff
contract surface. The shadow chain dispatches behavior, which lives
on methods.

```typescript
// Declares one method in its own body — `describe` is the intercept set.
class CountingShadow extends Shadow {
  public count = 0;
  describe(): string {
    this.count++;
    return this.callDown<string>();
  }
}

// Composes NamedMixin without declaring any own methods — empty
// intercept set, ShadowApi.attach throws 'no surface'.
class BareShadow extends NamedMixin(Shadow) {}

// Only override is an accessor — accessors don't enrol, so the
// intercept set is empty and attach throws 'no surface'.
class AccessorOnlyShadow extends NamedMixin(Shadow) {
  override get fullName(): string { return 'IGNORED'; }
}
```

The model: **a shadow intercepts what it explicitly says it
intercepts, and only methods participate.** Either declare/override
a method in the class body, or use `@Shadowing('hostMethod')` to
remap a differently-named local method onto a host method. Methods
inherited through composition are reachable on the shadow (mixin
defaults still run on the shadow's state when called) but they don't
enrol into the intercept chain on the host. Accessors and field
declarations don't enrol either.

Two reasons for these rules:

1. **Witness mixins make auto-enrolment unworkable.** A shadow that
   composes `Containable` to react to `onMoved` would otherwise
   auto-intercept `setContainer` / `getContainer` / etc. as
   no-op-defaults, masking the host's real behavior. Explicit
   declaration sidesteps the trap.
2. **Accessors are field-shaped.** A getter is a typed, side-effecty
   read; a setter is an invariant point. Letting shadows replace
   them at the inter-stuff boundary turns "this is the host's data"
   into "this is whatever the most recent shadow felt like
   returning." Accessors stay host-internal; if you want extension,
   wrap the accessor in a method (`getFoo()` returning `this.foo`)
   and shadow that.

`@Shadowing` adds individual methods to the surface without
composing a mixin (useful for one-off intercepts). It is for value
methods only — tagging a getter or setter results in a dispatch-time
"shadow has no method" error from `_invokeOn`.

### `ShadowApi` public API

```typescript
ShadowApi.attach(host: Stuff, shadow: Shadow): void;
ShadowApi.detach(shadow: Shadow): void;
ShadowApi.getShadows(host: Stuff, methodName: string): readonly Shadow[];
ShadowApi.getAllShadows(host: Stuff): ReadonlyMap<string, readonly Shadow[]>;
```

Note: `attach`/`detach` instead of `add`/`remove` since one call
covers all of the shadow's methods. `clear` is not currently part of
the public surface.

### Attach algorithm

1. **Reject misuse.** Re-attach throws; destroyed host throws;
   destroyed shadow throws.
2. **Compute the intercept set.** Take every own property name on
   `shadow.constructor.prototype` (minus `constructor`) whose
   descriptor is a value method — accessor pairs are filtered out.
   This is what the shadow's class body explicitly declares as
   *behavior*. Then merge in any names from the class's
   `_callSecShadowing` map (i.e. methods tagged with `@Shadowing`).
   Methods inherited from composed mixin layers are NOT
   auto-enrolled, and accessor pairs declared in the shadow's body
   are NOT enrolled either.
3. **Reject if empty.** A no-surface shadow is a bug.
4. **Validate `@Unshadowable`.** Method-form on the host method or
   class-form on the host's class (or any ancestor) → `ShadowError`.
5. **Run `@ShadowSecurity({ attach })` checks.** Per intercepted
   method, resolve the host's spec via the prototype chain. Deny
   throws `SecurityError`.
6. **Atomic install.** All mutation inside the framework module.
   Both maps written in the same synchronous call — no
   externally-reachable intermediate state.

If steps 1–5 fail, no state has been mutated.

### Dispatch (proxy invocation)

When `host.method(...args)` is invoked through the proxy and reaches
the security gate (after the destroyed-object guard and entry policy):

1. Look up shadows: `_shadowsFor(proxy, method)`. Empty → just push
   the host's frame and call `next()` (raw method runs).
2. If shadows exist, dispatch via `_withDispatch` which initialises
   ALS dispatch state:
   ```typescript
   { host, methodName, shadows, originalArgs, bypassNext: false }
   ```
3. Push a CallFrame for the topmost shadow. The shadow runs as **its
   own target** — see "Shadow identity on the call stack" below.
4. Invoke the topmost shadow's method. Only value methods are
   dispatched; the intercept set has already filtered out anything
   else.

If the topmost shadow's body calls `this.callDown(...)`, see "callDown"
below.

### Shadow identity on the call stack

A naïve dispatch could push a single CallFrame `{ caller: outsider,
target: T, method }` and run the entire shadow chain "as T." That's a
privilege-escalation hole: an untrusted shadow loaded from
`/mods/sketchy/`, attached to a trusted target T, would inherit T's
caller identity for any onward calls.

The dispatcher pushes a CallFrame **per shadow**, with the shadow
itself as the target:

```
[proxy intercepts T.method; runs T's policy with caller=outsider]
  ↓
push  { caller: outsider,  target: S_top,    method }
  S_top.method runs; this.callDown()
  ↓
push  { caller: S_top,     target: S_below,  method }
  S_below.method runs; this.callDown()
  ↓
push  { caller: S_below,   target: T,        method }
  T's original body runs
  ← pop, pop, pop
```

Consequences:

- Onward calls from inside `S_top` see `S_top` as their immediate
  caller — so `someApi.doSomething()` invoked from a sketchy shadow's
  body correctly fails `ApiOnly` when the shadow's module URL isn't
  trusted.
- Policies can directly target shadows by their identity:
  `FromModule('/mods/**')` denies onward calls from mod-loaded
  shadows.
- T's `@CallSecurity` policy is checked **once**, at entry, against the
  outside caller. It is not re-checked when T's body runs at the
  bottom (that would force every `callDown` to revalidate). Two
  different "callers," both correct.

### `callDown`

`Shadow.prototype.callDown(...args)` walks the dispatch stack via
`ShadowApi._callDown`:

1. Read dispatch state from ALS. Absent → throw "callDown called
   outside of dispatch."
2. Find self in `state.shadows` via `indexOf`. Not found → throw.
3. **`idx > 0`** → call the next shadow down. Push CallFrame `{
   caller: callingShadow, target: state.shadows[idx - 1], method }`,
   invoke the next shadow's value method, pop.
4. **`idx === 0`** → at the bottom. Set `state.bypassNext = true`,
   push CallFrame for the host, invoke the host method through the
   proxy. The security gate's `_consumeBypass()` returns true exactly
   once, skipping every check, so the raw method runs.

`indexOf` lookup means re-entry is fine: a shadow that calls down,
returns, and calls down again gets the same downstream behavior both
times. No position counter to manage.

Args are explicit: `callDown()` invokes the next layer with no args;
`callDown(modifiedArg)` substitutes; `callDown(...args)` passes
through. We don't infer or default — explicit is honest about what's
flowing through.

Reentrancy is handled by ALS — nested dispatches push their own state.
Concurrent dispatches on the same Shadow attached to multiple targets
are fine because state is per-async-context, not per-instance.

### `callBypass`

`Shadow.prototype.callBypass(method, ...args)` runs the host's
*original* method, bypassing every shadow attached to the host
(including self). For the narrow case where the shadow needs to read
the unmediated value — a tracer that wants to log the "real" return
value regardless of other shadows, an internal sanity-check, etc.

Implementation: opens a fresh dispatch context with `bypassNext: true`,
pushes a frame with the shadow as caller and the host as target, then
calls `host[method].apply(host, args)`. The proxy's get-trap, seeing
the bypass marker via `_consumeBypass`, returns the raw method
directly, bypassing both the entry policy and the shadow chain.

Use sparingly. `callBypass` deliberately bypasses the host's
`@CallSecurity` — it's a privileged operation, only callable from the
shadow itself (via `this`).

### Late-binding shape between SecurityApi and ShadowApi

`security.ts` and `shadow.ts` form a load-time cycle if either tries
to import the other directly: `shadow.ts` imports `SecurityApi` and
runs `SecurityApi.decorateApiClass(ShadowApi)` at module-bottom, which
would crash mid-cycle if `security.ts` were also depending on
`shadow.ts` to load first.

`SecurityApi` declares an internal `ShadowApiLike` interface and
exposes `_registerShadowApi(impl)`. `ShadowApi` calls the registration
once at its own module-bottom:

```typescript
SecurityApi.decorateApiClass(ShadowApi);
SecurityApi._registerShadowApi(ShadowApi);
```

The interceptor dereferences the registered slot at runtime only, by
which point all imports have resolved. The `ShadowApiLike` interface
is the late-binding contract — `_consumeBypass`, `_shadowsFor`,
`_withDispatch`, `_invokeOnShadow`. Anything the security gate needs
from `ShadowApi` flows through here.

### Authority defaults

| Operation | Default |
|---|---|
| `ShadowApi.attach(host, shadow)` | Anyone — but each method's `@ShadowSecurity({ attach })` (if any) runs |
| `ShadowApi.detach(shadow)` | Anyone with a reference — but each method's `@ShadowSecurity({ detach })` (if any) runs |
| `ShadowApi.getShadows / getAllShadows` | `Public` |
| `Shadow.callDown(...)` | Internal — only from inside a shadow's method during dispatch |
| `Shadow.callBypass(method, ...)` | Internal — only from the shadow itself (`this`) |

`ShadowApi.attach` and `detach` themselves are gated by the framework's
default `Public` policy because the actual authority gating happens
via `@ShadowSecurity` per-method. Wrapping `attach`/`detach`
themselves in `Admin` would prevent legitimate gameplay (self-buffing,
opting out of effects).

### Lifecycle ordering on host destruct

`StuffApi.destruct(host)`:

1. **`host.canDestruct()`** Witness fires through the proxy. If the
   host has shadows on `canDestruct`, the chain runs (the hook is
   shadowable). A `{ ok: false, reason }` result throws
   `DestructError`. The `forceDestruct` admin entry invokes the
   witness identically (so observers / audit fire) but ignores the
   veto.
2. **`host.onDestruct()`** Witness fires through the proxy. Cleanup
   hook — runs while the host is still live.
3. **Privileged shadow detach.** `ShadowApi._detachAllForHost(host)`
   reads every shadow attached to the host (snapshot, deduplicated)
   and removes each via `#removeAtomic`. Bypasses `@ShadowSecurity({
   detach })` because host destruction is unconditional.
4. **`host.destroy()`** runs through the proxy. `@CallSecurity(ApiOnly)`
   enforces that only Api-layer code can reach it.
   `@Unshadowable` guarantees no shadows can ever have been attached,
   so dispatch finds an empty shadow array and proceeds straight to
   the original body. It marks `_isDestroyed`, unregisters from
   `StuffApi`.

After detach, **shadows live on.** They can be reattached to a
different host. The shadow itself is only destroyed when something
explicitly calls `StuffApi.destruct(shadow)`.

### Persistence

Shadow is a sibling of Idea (not under it), and it isn't a `Document`,
so it has no CRUD surface (`save`/`find`/`findById`/`delete`).
Persistence happens through the **template/clone pipeline only** (see
[templates.md](./templates.md)).

A shadow class that composes mixins with persistent fields (e.g.
`PropertiedMixin(Shadow)` for a buff with stored `duration`,
`magnitude`) can be backed by a template:

```yaml
path: /system/buffs/regeneration
class: /lib/some/Regeneration   # extends Shadow
hydratorClass: /obj/persistence/PersistentHydrator
data:
  duration: 30
  magnitude: 5
```

Cloning the template via `StuffApi.clone('/system/buffs/regeneration')`
hydrates the shadow's persistent fields the same way it would for any
Stuff. There is no `save()` for a Shadow — to update its persistent
template, edit the template doc directly via `TemplateApi.saveTemplate`.

What is **NOT** included: **attachment records** — i.e., persisting
"shadow X is attached to host Y." On server restart, runtime shadow
instances are gone; hosts come up shadow-free. Re-attaching after
restart is the responsibility of whatever subsystem owns the
buff/shadow (e.g., a buff manager that loads buff records from its own
collection and re-attaches on bootstrap). The *attachment relationship*
is not persisted by the shadow framework itself.

## Command Attribution

Three pieces of state on the Command frame's metadata thread command
identity through the synchronous chain:

- **`commandContext: CommandContext`** — the live `CommandContext`
  object stamped onto the frame's metadata.
- **`causingCommandId: string`** — the originating command's id. For
  a directly-issued command this is the same as `commandId` (the
  command is its own cause). For a scheduled/delayed callback,
  `ScheduleApi` re-plants the originating frame's `causingCommandId`
  onto the callback's Root frame, so it points back to the command
  that scheduled the work — even though the synchronous chain is gone.

`CommandGiverMixin.executeCommand`:

```typescript
async executeCommand(commandText: string, context: CommandContext): Promise<void> {
  // The proxy already pushed a frame. Tag it.
  ExecutionContextApi.tagCurrentFrame(FrameKind.Command);

  const commandId = nanoid();
  context.commandId = commandId;
  ExecutionContextApi.updateCurrentFrameMetadata({
    commandContext: context,
    causingCommandId: commandId,
  });
  // …
}
```

Reads:

- **`getCurrentCommandContext()`** finds the Command frame and returns
  its `metadata.commandContext`. Used by `Scene.send()` and `MudlogApi`
  to stamp `meta.commandId` and default the recipient to the command
  giver.
- **`getCurrentCausingCommandId()`** walks the stack top-down and
  returns the first `metadata.causingCommandId` it hits.

`updateCurrentFrameMetadata(patch)` merges `patch` into the top-of-stack
frame's `metadata`, gated by the same allowlist as `tagCurrentFrame` —
only framework files may mutate frame metadata.

### `ScheduleApi` interaction

When a scheduled callback fires, the original synchronous chain is
gone. `ScheduleApi.fire` plants a fresh Root frame via `runRoot`, then
re-plants the originating `causingCommandId` onto the new frame's
metadata via `updateCurrentFrameMetadata`. Either way — direct command
or delayed callback — `getCurrentCausingCommandId()` surfaces the live
"originating command id."

## Privileged Surfaces

Some methods exist but should never be called from outside the Api
layer. They become `@CallSecurity(ApiOnly)` (often combined with
`@Unshadowable` and `@Final`):

| Surface | Decorator stack |
|---|---|
| `Stuff.destroy()` | `@Final @Unshadowable @CallSecurity(SecurityPolicies.ApiOnly)` |
| `Stuff.isDestroyed()` | `@Final @Unshadowable` (read invariant) |
| `Containable.setContainer()` (when exposed) | `ApiOnly` (only `ContainmentApi.move`) |
| Manager-layer mutations | `ApiOnly` |

Direct `obj.destroy()` from outside `mud/api/` throws `SecurityError`.
Subclass `destroy()` overrides throw `FinalViolationError` at import
time. Shadows attempting to attach to `destroy()` throw `ShadowError`.
Together these guarantee that `StuffApi.unregister()` always runs —
essential for GC.

See [lifecycle.md § Destruction](./lifecycle.md#destruction) for the
full destruct → canDestruct → onDestruct → shadow-detach → destroy
ordering.

## The narrow-entry pattern (force-bypass entries)

Force-bypass entry points on lowest-layer Apis
(`StuffApi.forceDestruct` and `ContainmentApi.forceMove`) carry a
parallel-API shape: `forceX` lives alongside `X`, sharing a private
core that invokes every witness identically and branches only on
whether to `assertVetoOk`.

The witness itself is **force-unaware**: `canX(): VetoResult` takes
no force argument. Force still fires the witness so any side effects
the target attaches (audit hooks, observers, telemetry) see every
call uniformly — `forceX` only skips the assertion, not the
invocation. `onX` post-hooks fire unchanged.

The pattern only fits operations that act on a **target** with state
to consult. Destruct (target = the Stuff being destroyed) and move
(target = the item + source + destination) both qualify. Clone and
reload do not: clone is "willing something new into existence" with
no instance to ask, and reload operates on modules / prototypes, not
on Stuff. Both have failure paths (permissions, singleton
enforcement, frozen blueprints, compile errors) but those live in
the right places already; they don't shape into the witness pattern,
so neither `clone` nor `reload` carries a force variant.

The **narrow-entry pattern** is the access build's framing for who
authorizes `forceX`:

1. The Api method is decorated with `FromController(...controllers)`
   (string-form `FromModule('/obj/command/X#X')` in the shipped
   wiring to avoid a value-level static-import cycle). Only the
   listed verb controllers can reach the entry point at all;
   everything else throws `SecurityError`.
2. The verb controller does the access check via
   `AccessApi.can(giver, 'force-X', target)` (or `canMutateZone`
   when the target is a Zone Template) BEFORE invoking the
   `forceX` Api.

Combined, the mutation has exactly one legitimate entry path AND
that path enforces who is authorized.

Adoption sites today:

- `StuffApi.forceDestruct` →
  `FromModule('/obj/command/author/DestructController#DestructController')`.
  `DestructController` does the access check.
- `ContainmentApi.forceMove` →
  `AnyOf(FromModule(TeleportController), FromModule(GotoController))`.
  Each controller does its own access check.

Full pattern + verb-controller gate matrix:
[access.md](./access.md).

Why parallel methods rather than `{ force: true }` options:

- Auditable. `forceX` calls grep cleanly; flag arguments don't.
- Greppable. The policy decorator sits on the method, not the
  argument list.
- Policy-replaceable. Swap policies once; every `forceX` re-gates
  uniformly.

Why one force entry per operation, not parallel at every wrapping
layer (e.g., not `Mobile.forceTeleport`, no `MobileApi.forceGoto`):
duplicating the seam at every wrapper doubles the security-decorator
surface and gives admins multiple bypass entries that must each be
audited separately. One enforcement point per operation, composed by
verb controllers, is the cleaner shape. Verb controllers orchestrate
"try the polished/high-level path; on veto, fall back to `forceX`
when `-f` is set and access allows."

## Hydrator is Not an Exception

Earlier framework drafts treated the Hydrator as a privileged "system
context" that bypassed policies. The shipped position is the opposite:
Hydrators are themselves `Stuff`. They have stuffIds, they have a class
path (`/obj/persistence/PersistentHydrator`), they get pushed onto the
call stack like anything else when invoked. Policies that care about
the hydrator can match on
`FromModule('/obj/persistence/PersistentHydrator')` or with
`{ includeSubclasses: true }`.

Bracket-assignment of persistent fields invokes the field's setter.
The setter's own `@CallSecurity` annotation (if any) decides — no
CallSec-specific carve-out needed. This is also why per-field
invariants belong on setters, not in post-hydrate `normalize()` hooks
— see [antipatterns.md § Per-Field
Invariants](../antipatterns.md#per-field-invariants-belong-on-setters-not-in-normalize-hooks).

There is **no `runAsSystem()` escape hatch.** The whole point of the
framework is that there is no privileged out-of-band caller identity.

## What a gated call costs

Worth knowing, because it was never priced and it is the engine's most
common operation. Measured under **`tsx`, the production runtime**
(`deployment.md` — the server runs from TypeScript source, not a
compiled build):

| | µs |
|---|---|
| field read through the proxy | 0.3 |
| **gated method call** | **50** (was 68) |
| the same method, unwrapped | 0.1 |

It is flat in world size, flat in call depth, and not a first-touch
cost — a constant, paid per dispatch.

> ⚠ **~37 µs of it is one `new Error().stack`.**

Every gated dispatch runs `ExecutionContextApi.run` →
`_assertFrameMutatorAllowed` → `ModuleApi.getImmediateCallerUrl`, which
captures a stack to answer "who called me". Building the stack text
runs the runtime's `prepareStackTrace`, which **source-map remaps every
frame** — the same capture costs 9.4 µs on plain node and ~37–52 µs
under `tsx`.

The remap is **per frame**, so `#walkExternalFrames` takes a
`maxFrames` bound and the immediate-caller lookups pass one: ten frames
were being formatted to answer a question about one.
`getImmediateCallerUrl` and `#findCallerUrl` **retry unbounded** if the
bound finds nothing, so `IMMEDIATE_CALLER_FRAMES` is a performance knob
and never a correctness one — a truncated capture must not turn into a
`SecurityError`.

> ⚠ **Do not extract the capture into a helper.** A helper adds one
> stack frame, and one frame is enough to push a legitimate caller out
> of V8's 10-frame default window. `assertTestOnly` scans for a test
> frame *anywhere* below it, and the chain through a nested
> `_clearAllForTesting` lands on exactly 10 — extracting the capture
> broke **438 tests**. The hazard is pre-existing and already noted in
> `assertTestOnly`'s own comment; the inline capture is what keeps it
> latent.

### The optimization that does NOT work

Swapping in a raw `prepareStackTrace` (`(_e, frames) => frames`) skips
both the text build and the remap, and takes a gated call from 68 µs to
**14 µs** — a 4.9× win. **It was tried and reverted**, because
`CallSite.getFileName()` and the source-mapped rendered stack disagree
somewhere that changes a **policy decision**: `combat-gym`'s
feint-vs-turtle cell flips from `A` to `draw`, deterministically, in
isolation. Probing both forms side by side at several depths showed
byte-identical URLs, so whatever differs is narrower than the probes —
and a security gate is the wrong place to ship an unexplained
behavioural delta. The 4.9× is real and still on the table; it needs
the disagreement identified first.

**The remaining capture cost is the source-map hook**, which exists
because production runs from source so authors can `write`/`reload`
live. That is a deliberate trade, not an oversight — it just has a
price, and this is it.

## Errors

All extend `Error`.

- **`SecurityError`** — policy denied the call. Carries `stuffId`,
  `methodName`, `policyName` (all optional).
- **`DestroyedObjectError`** — touching a destroyed `Stuff`. Carries
  `stuffId` and `methodName` (both required).
- **`ShadowError`** — shadow install / remove / dispatch failed.
  Carries `shadowId`, `hostId`, `methodName` (all optional).
- **`FinalViolationError`** — class load detected a subclass overriding
  a `@Final`-marked method. Carries `className` and `qualifiedMethod`.

## Audit (Unimplemented)

Pillar 5 of the framework is essentially unimplemented. The original
design called for structured audit entries on every denied call, every
shadow install/detach, and every destroyed-object hit:

```
{ kind: 'security_deny', caller, target, method, policy, callStack }
{ kind: 'shadow_attached', target, method, by: caller }
{ kind: 'destroyed_access', target, method, by: caller }
```

The shipped code emits none of these. `MudlogApi` exists as a separate
subsystem and is wired to `Scene.send()` via `getCurrentCommandContext`
(so per-command output gets a `meta.commandId` stamp), but it is not
wired to the security gate, the shadow attach/detach paths, or the
destroyed-object guard.

The attribution machinery is in place: caller identity (module URL +
template path), the full callstack via `dumpCallStack`, the live
`causingCommandId`. The missing piece is the sink — a registered audit
interceptor on `ProxyApi`, plus an explicit log call in
`ShadowApi.attach` / `detach`. Nothing prevents adding it; nothing
yet drives the priority.

## Cross-References

- [lifecycle.md](./lifecycle.md) — construction sentinel, synthetic
  constructor frame (`FrameKind.Constructor`), destruct →
  canDestruct → onDestruct → shadow detach → destroy ordering,
  `Stuff.destroy` decorator stack, `canDestruct` / `onDestruct`
  extension points
- [templates.md](./templates.md) — `ProxyApi.wrap` in the clone
  pipeline, `templatePath` stamping (feeds `FromTemplate` policies),
  Hydrator's role
- [persistence.md](./persistence.md) — `Document` records are not Stuff,
  so they're not mediated by the Proxy / security gate (auth records
  have no game identity); setter-based field invariants
- [state-model.md](./state-model.md) — `templatePath` and `zone`
  stamping at clone time
- [antipatterns.md § Per-Field
  Invariants](../antipatterns.md#per-field-invariants-belong-on-setters-not-in-normalize-hooks)
  — setter-based invariants the Hydrator rides on

## The sandbox boundary's read aperture (`SecurityApi.projectAcross`)

The sandbox build added a Layer-4 dispatch check to the gate (see
[sandbox.md](./sandbox.md)) and, with it, one new `SecurityApi`
surface worth knowing about from this side:

```ts
SecurityApi.projectAcross(a, b, fn, principal)
```

Runs `fn` under an omni root **iff** `a` and `b` sit on opposite sides
of a circle (`b === undefined` compares against the ambient context).
Same-side calls take an identity branch and see no widening at all.

The rule it encodes: the layers contain durable **mutation**, but a
read-only *projection of a person* — what they are called, what they
can sense, what they are doing — is neither durable nor a mutation, and
yields text or a display row, which the doctrine already lets cross.
Its consumers are `RecognitionApi.describe*` / `perceivedKeywords` /
`salientFeatures`, `PerceptionApi.sensorium` / `canPerceive`, and
`SocialApi.statusOf` / `composeRow`.

Two things about the shape are load-bearing:

- **The root is planted AS the calling facade** (hence `principal`). A
  fresh root discards the frame that identified the caller, so the
  logic singleton's own per-facade `FromModule` gate would otherwise
  refuse its own facade.
- **A projection is the unit, not a call.** Exempting method *names*
  does not compose: an exempt method that dispatches on `this`
  re-enters the proxy, and the inner hop is a fresh check with the same
  cross-scope context, so the denial just moves down one frame
  (`getPresentation` → `getDisguise` → `getAllOccupants`). Anything
  with a body wraps its body; only genuine leaves belong on a list.

**Exemption lists are CI-checked** — `pnpm lint:boundary`. The
base-class exemptions are typechecked for free; the template-path
strings were not, so a renamed singleton could silently change the
security surface (the same problem shape `lint:gates` exists for). It
also asserts the symmetric and inbound-only method sets stay disjoint.
