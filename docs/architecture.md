# Architecture

This doc covers the cross-cutting architectural conventions that apply to
every subsystem in Saxonberg 2.0: how layers separate, how `Manager` vs
`Api` classes differ, how mixins are organized, and how the codebase is
laid out.

For specific subsystems, see subsystems/.
For coding antipatterns to avoid, see [antipatterns.md](./antipatterns.md).

## Three-Layer Architecture

```
┌─────────────────────────────────────────────────┐
│ Mudlib Objects (Avatars, Locations, NPCs, …)   │  ← packages/server/src/mud/{lib,obj,cmd}
│  ↓ (always go through)                          │
│ Api Layer (public, static utilities)            │  ← packages/server/src/mud/api/
│  ↓ (may delegate to)                            │
│ Managers (privileged singletons with state)     │  ← packages/server/src/backend/, services/
└─────────────────────────────────────────────────┘
```

The split is enforced at the call-security layer: Api classes are
decorated with `SecurityApi.decorateApiClass(XApi)` and all entry points
pass through that gate. Mudlib code MUST go through the Api layer to
reach managers; direct calls to managers from mudlib are blocked.

The server itself is structured as:

```
packages/server/src/
├── services/        Auth, WebSocket, module loader (services/loader)
├── backend/         Backend (I/O bridge), Application (game logic),
│                    PersistenceManager (singleton)
└── mud/             Mudlib
    ├── api/         ~22 static Api classes (the public surface)
    ├── cmd/         YAML command views (declarative)
    ├── config/      constants
    ├── lib/         Substrate ONLY — abstract roots, mixins, value
    │                objects, framework attachments. Never instanced.
    └── obj/         Everything instanceable — anything a template's
                     `class:` names (Avatar, Topic, Weapon, Room, …)
        ├── command/   CommandController implementations
        └── hooks/     persistence around-hooks
```

## api/ vs lib/ — which layer owns this code?

When you're adding a method and trying to decide whether it goes on a
`lib/` class or in an `api/` static, the deciding question is **who has
authority over the answer**.

- **`lib/`** is for behavior whose answer is the host's to choose.
  Subclasses can override it; mixins compose into it; instance state
  changes it. The class or mixin is the authority over what the method
  means for that host.
- **`api/`** is for behavior with a single guaranteed answer. Same
  inputs, same outcome, same caller-visible semantics — every time,
  for every caller. The method body may branch on mixin presence or
  read instance state, but the *contract* doesn't vary by host
  identity.

The diagnostic: **"Could a subclass legitimately want to change what
this method does?"** If yes, it belongs in `lib/`. If no, it belongs
in `api/`. "Has to work the same way every time, with no override
seam" is what an Api guarantees; "has an override seam" is what a
`lib/` class offers.

### Orchestration lives one layer up from raw steps

The corollary that prevents `lib/` from drifting into Api territory:
when a method needs to *walk*, *compose*, or *retry* polymorphic
steps, the orchestration belongs in `api/` even though each
individual step belongs on a `lib/` class.

- **Raw step** = the polymorphic answer. Stays on the class or mixin
  that owns the contract (`Mobile.traverse`, `Zone.lookupField`,
  `Visible.getLongDescription`).
- **Orchestration** = code that calls raw steps, walks results, threads
  state across calls (`LocomotionApi.engageAround`,
  `ZoneApi.getEnclosingZone`, `ContainmentApi.move`).

The smell that flags an inverted layering: **a `lib/` method
`await import(...)`-ing into `api/` to take its next step.** The fix
is to pull the orchestration up into `api/` and have the `lib/` method
call the Api by static import (the Api can statically import the class
as a type-only reference).

Worked example — Zone field inheritance:
- Polymorphic step: `Zone.lookupField` reads own value, defers to
  `Zone.lookupAncestorField`. Subclasses (`RootedZone`) override the
  ancestor step to root the walk. **Lives in `lib/zone/Zone.ts`.**
- Orchestration: "find the nearest enclosing-zone template by walking
  `Template.ancestorPaths`, skip non-folders, singleton-resolve the
  hit." **Lives in `ZoneApi.getEnclosingZone(zone)`** — no
  polymorphism, pure plumbing through `Template` + `StuffApi`.

Before the split, `Zone.getEnclosingZone()` was on the class and had
to `await import(...)` `Template`, `StuffApi`, and `ZoneApi` from
inside the method body to break the load-time cycle. Pulling the
orchestration up to `ZoneApi` removed the dynamic imports from the
`lib/` side; the only remaining cycle-break (a `SpatialZone` lazy
lookup) lives in `api/zone.ts`, where it belongs.

### Three flavors of each layer

`lib/` and `api/` each cover three distinct shapes; the polymorphism
question above resolves cleanly only when you know which kind you're
adding.

> **The placement invariant: nothing instances `/lib/`.** A class that
> a template's `class:` resolves to lives in `obj/`, on both axes —
> the `.ts` file and the template path. `lib/` holds only what is
> inherited. Enforced by `pnpm lint:instanceable`; the full rule and
> the eight split classes are in
> CLAUDE.md § Instanceable Lives in `obj/`.

`lib/` holds:
- **Mixins** — capability shapes (`Containable`, `Sensor`, `Posed`).
  Polymorphic by design.
- **Abstract Stuff roots** — role identity that subclasses inherit and
  nothing clones (`Stuff`, `Thing`, `Location`, `Zone`, `Character`,
  `Creature`, `Modality`). Overridable. When one of these is *also*
  being cloned generically, it splits: the base stays here and a thin
  concrete subclass in `obj/` takes the clones (`Thing` → `obj/Prop`,
  `CartesianLocation` → `obj/location/Room`).
- **Framework attachments** — objects that ride a Stuff, model nothing
  on their own, and are never template-backed. `lib/stuff/Shadow` is
  the exemplar and a **permanent** exception to "obj/ holds Stuff
  classes": it has its own identity but is never cloned from a
  template, so the placement rule genuinely does not reach it.
  Also here: the *instanced but never stamped* fixtures —
  `BoundaryAnchor` (minted onto its boundary), `SandboxCrossingExit`
  (minted by the crossing), and `LightningStrike`, whose source says
  it plainly: *"a transient single-use vessel — minted, conducted, and
  reaped inside this one call, never authored, never persisted."* The
  test is **does an instance carry a template-path stamp**, not *is it
  ever `new`'d*. `ExitableVessel` sits here too, deferred: it has no
  `fieldMeta` and no documented authoring path, so it moves to `obj/`
  when a consumer demands a concrete class, not before.
- **Value objects** — pure data + small per-instance math (`Light`,
  `Quantity`). Not Stuff. Lives in `lib/` because it's a domain
  primitive the Api layer consumes.
- **Named value-object / vocabulary / registry modules** — the
  sanctioned home for a substrate primitive that isn't an instanceable
  `Stuff` but is still *the concept the module exists for*: a value
  class (`Light`, `Quantity`, `Reserve`), an enum-like vocabulary plus
  its validation array (`lib/persistence/Collections.ts`'s
  `Collections` — collection *names*, no driver and no I/O, which is
  why it lives here and `backend/PersistenceManager` re-exports it
  rather than owning it), or a platform-wide registry (`lib/mixin.ts`'s
  `Mixins`, `lib/paths.ts`'s `TemplatePaths`). This is the fourth
  category named so that an orphan type/constant has a home other than
  the forbidden `types.ts` / `constants.ts` reflex — see
  "One concept per module" below.

`api/` holds:
- **Pure utilities** — no Manager, just functions
  (`NavigationApi.parseDirection`, `MmlApi.escape`,
  `PathPatternApi.match`).
- **Manager wrappers** — gates to a privileged singleton
  (`PersistApi`, the call-security chokepoint over `PersistenceManager`).
- **Stuff orchestration** — operates on Stuff, threads polymorphic
  steps (`StuffApi`, `ContainmentApi`, `LocomotionApi`, `ZoneApi`).
  This is where the orchestration-vs-step decision bites.

> **`api/` is functions + the types they speak — nothing instanceable.**
> An `api/` file exports its static `FooApi` class and the call-shape
> types its signatures use, and nothing else. Value/builder classes,
> collections, and error classes that once lived in `api/` are `lib/`'s
> domain, concept-colocated, re-exported (value re-export) from the
> face so callers still reach them through the Api: `Scene` →
> `lib/message/Scene.ts`, `Mml` → `lib/message/Mml.ts`, `Prose` →
> `lib/prose/Prose.ts`, `PathTrie` → `lib/collections/PathTrie.ts`, and
> the error classes (`ContainmentError`, `PromptCancelledError`,
> `SourceTreeSandboxError`, `TemplateError`) to their concept's `lib/`
> module. Domain-concept *types* are likewise *defined* in `lib/` with
> their concept and re-exported type-only from the face; only pure
> call-shapes with no domain home (the `ScheduleHandle` precedent) are
> defined in the api file.

### The Api ↔ logic-singleton split

An `Api` carries identity three ways an unsettled organizational layer
shouldn't — it holds the logic, it anchors the generated docs, and it's
where types live. The surface-architecture refactor relocates all three
so the `Api` becomes what it actually is: a **thin, typed, secured
forwarding shell**. Each convertible Api's public statics forward to a
stateless `Stuff` **logic singleton** that holds the implementation:

```
FooApi (api/foo.ts)              static forwarders + decorateApiClass + types
   ↓  StuffApi.singletonSync('/obj/api/foo', factory)
FooLogic (obj/api/FooLogic.ts)   the logic — a stateless Stuff, @internal,
                                 instance methods gated to FooApi, HMR-able
```

The logic singleton is a `Stuff` (a runtime class, so `obj/api/`),
extends **`ApiLogic`** (a thin `Idea` subclass shared by every logic
singleton; its job is to make them residency-exempt — see
[residency.md](./subsystems/residency.md)), composes **no**
`PostRegistrationMixin` (statelessness is load-bearing — `dest` is the
reload invalidator), is marked
`@internal` *on the class declaration*, and gates each public method
`@CallSecurity(FromModule('/api/<feature>#<Feature>Api'))`. Protected
internal logic that used to crowd the Api's TypeDoc surface now lives
here, off the documented surface. See
[hot-reload.md § Api logic singletons](./subsystems/hot-reload.md#api-logic-singletons-objapifeature)
for the HMR mechanics and
[call-security.md § The api↔logic-singleton recipe](./subsystems/call-security.md#the-apilogic-singleton-recipe)
for the gating recipe (per-method gate, self-call gotcha, the `ApiOnly`
widening). A handful of bootstrap-cycle and bootstrap-special Apis stay
static classes (documented in call-security.md).

### One concept per module — definition site vs import site

A module exists to define exactly one **named concept**; the filename is
that concept's name; the concept is the primary export. Every other
export is a supporting member of that one concept — the types its surface
speaks, the constants it's parameterized by. A type or constant is never
itself the reason a module exists (unless the concept *is* a value object
/ registry — then the module is named for it). `types.ts`,
`constants.ts`, and barrel `index.ts` are forbidden by construction on
the consumed surface.

This separates two questions that the old "where does this type live?"
debate conflated:

- **Definition site** is dependency-driven — a type may be *defined* on a
  cycle-breaking dependency leaf, wherever the runtime graph forces it.
- **Import site** is the *face*: any author-facing type is re-exported
  (type-only — erased, no runtime edge, can't cycle) from *every* Api or
  mixin whose signature speaks it. "Look where you'd use it" always
  finds it; you never have to out-guess a canonical home.

Constants are the asymmetry: a re-exported constant is a runtime value
(a real `api → definition` edge that *can* cycle), so **constants are
placed, not re-exported**. See [antipatterns.md](./antipatterns.md) for
the `<Concept><Role>` naming rule that lets you guess a type's face from
a bare name.

### Path-resolved modules — brains, and now wiki components

Two module families are **not imported by anything**. They are resolved
by path, per invocation, through `StuffApi.resolveExport(path, name)`,
so adding one is dropping in a file and there is no registry to edit —
and so hot-reload propagates without a restart.

| family | home | sole export | resolved by |
|---|---|---|---|
| **brains** | `lib/behavior/<verb>.ts` | `export const brain = class {…}` | `BehavedMixin`, per fire |
| **wiki components** | `lib/wiki/components/<name>.ts` | `export const component = class {…}` | the wiki render pipeline, per occurrence |

Both are **named class-expressions**, and that is load-bearing rather
than stylistic: the hot-reload registry retains only class-like exports,
so a plain object or an arrow function would silently fail to
re-resolve after a reload. Both are duck-typed at the call site rather
than `instanceof`-checked, for the same reason — after a reload the
constructor identity differs from any reference held elsewhere.

⚠ A component's tag name becomes its **module basename**, so the
`[a-z][a-z0-9-]*` charset rule in `api/mml/tags.ts` is a security
boundary, not tidiness: `../`, slashes and dots must be unrepresentable
*before* any resolver sees the string, not sanitised after.

⚠ A component receives **no reader identity** — see
[wiki.md](./subsystems/wiki.md) § the reveal model. That absence is what
keeps the reveal model to one gate instead of N.

### Export discipline & the sanctioned-exception registry

The surface is now normalized: **every module exports classes and types
only** — the one concept it defines (a class) plus the types and
constants its surface speaks. **There are no free-floating exported
helper functions.** A would-be helper folds into the owning `Api` (as a
static method), the owning class, or a value-object — never a loose
`export function`. This is what makes the whole surface discoverable and
what feeds the `help`/doc system: *callable == visible == cared-about*.

Three kinds of exported function are **recognized categories**, not
exceptions — they're how those categories are spelled:

1. **Mixin factories** — `export function FooMixin(Base)` in
   `lib/<subsystem>/`. The mixin module category (the `Mixin` name suffix
   is the marker the lint keys on).
2. **Decorators** — the factories in `lib/security/decorators.ts`
   (`CallSecurity`, `Unshadowable`, `Final`, `Shadowing`,
   `ShadowSecurity`) and `lib/security/RequiresActive.ts`. A decorator is
   a function by nature; these two files are the decorator homes.
3. **Sealed-subdir pipeline internals** — `api/mql/**` and `api/mml/**`
   are an Api's private impl package: only the parent face (`api/mql.ts`
   / `api/mml.ts`) may import them (a separate `no-restricted-imports`
   rule enforces the seal), so inside the seal they're ordinary modules
   that export functions freely.

Beyond those, a genuine **ad-hoc exception** must be marked **at its
site** with `eslint-disable-next-line no-restricted-syntax -- <reason>`.
The disable comments *are* the registry — grep
`no-restricted-syntax` under `src/mud/{api,lib}` to enumerate the live
set. Two kinds are recognized:

- **Test-only white-box exports** — a function exported solely so a
  white-box unit test can exercise an internal stage. Today:
  `api/prompt.ts#buildPromptContext`,
  `api/command.ts#validatePhaseEffect`/`collectPhaseEffects`,
  `api/mql-subscription.ts#resolveFieldSet`/`collectSubscribableFields`/`projectFocus`,
  `lib/command/parsers/msh.ts#detectEmotePrefix`/`stripEmotePrefix`,
  `lib/description/Visible.ts#senseStripAugmenter`.
- **DI injection seams** — a backend→mudlib wiring slot a free function
  fills at boot. Today: `lib/connection/HasInteractive.ts#setClientStateUpdatePush`.
- **Logic-singleton sibling cross-imports** (`no-restricted-imports`) — two
  logic singletons in one subsystem that must reference each other's class
  *identity* for a framework allowlist, which the facade can't broker.
  Today: `obj/api/ConsumerLogic.ts` ↔ `obj/api/ProducerLogic.ts` (the
  class identities feed `EventApi.restrictSubscribe`'s subscriber
  allowlist for the shared `CommandDispatchedEvent` consumer/producer
  tap; cycle-safe). Each opts out with
  `eslint-disable-next-line no-restricted-imports -- <reason>`.

**Enforcement.** ESLint rules (`.eslintrc.js`): `api/*.ts` bans exported
functions / function-consts; `lib/**/*.ts` bans the same, exempting
`*Mixin` factories by name and the two decorator files by path; a
`no-restricted-imports` rule forbids importing `obj/api/*Logic`
singletons anywhere except each logic's own `api/*.ts` facade (and
`__tests__`, which white-box logic internals). All opt out only via an
`eslint-disable` + justification. (The gate-string resolver and the
sealed-subdir rule round out the [lint family](#cross-references).)

**The ask-first rule.** Introducing a *new* exception — anything that
needs a fresh `eslint-disable no-restricted-syntax`, or a file that
doesn't fit the [module taxonomy](#) — is a drift risk by definition.
**Do not add one autonomously: surface it to the user and get explicit
sign-off first**, then record it here with its reason. The lint failing
on a new exported helper is the tripwire that forces this conversation.

### Where types live — colocate with the author

A type's "author" is whoever defines what fields, methods, or
variants it has. Types live with their author, never in a separate
shared barrel.

| Type describes... | Lives in |
|---|---|
| A mixin's public shape (`Containable`, `Sensor`, `Vocal`) | The mixin file in `lib/<subsystem>/` |
| A `lib/` class's surface or its value-object form (`LightBand`, `VisibilityDetail`, `VisionProfile`, `Unit`, `TagTableEntry`, `RecencyBucket`) | The class file in `lib/` |
| An Api's contract — inputs, outputs, options, accumulators (`CommandContext`, `StartResult`, `EmissionData`, `CreateBoundaryOptions`, `ShadowQuality`) | The Api file in `api/` |
| Cross-process wire format | `@saxonberg/types` |

No `types/` folder under `mud/`. The colocation rule keeps the contract
next to whoever can evolve it; a barrel would let any file scatter
new types anywhere and rot the authorship signal.

> **Definition vs import, restated for types.** The table above is the
> *definition* site (the author's module). The *import* site for any
> author-facing type is the **face** — re-exported type-only from every
> Api/mixin that speaks it (see "One concept per module" above). Where a
> dependency cycle forces a type's definition off its conceptual home,
> it still re-exports from the face, so consumers never chase it.

The recurring mistake to avoid: a type drifts to `api/` because
"the Api uses it first" — but the type describes a `lib/` concept
the Api just *consumes*. Move it back to where the concept is
authored, and `import type` it from the Api file.

### Dynamic imports as a cycle smell

`await import(...)` has two legitimate uses and one antipattern.

Legitimate:
1. **Loading authored content** — controller URLs, HMR module reloads,
   `StuffApi.loadClassByPath` resolving a template's `class:` field at
   runtime. These paths aren't in the static graph by design; the code
   they reach is content, not framework.
2. **Cycle-breaking at the `api/` layer** — when an Api orchestrator
   uses `instanceof X` against a `lib/` class that itself extends a
   class the Api references (`api/zone.ts` instanceof-checking
   `SpatialZone`, where `SpatialZone extends Zone` and `ZoneApi` is
   reachable from `Zone.lookupAncestorField`). Lazy-loading the
   subclass inside the method body is the correct fix; document the
   cycle with a comment so it isn't deleted later.

Antipattern:
3. **`lib/` dynamic-importing `api/` to take its next step.** This is
   inverted layering; the orchestration belongs above the polymorphic
   step. Fix by moving the orchestration up into the Api and calling
   the Api by static import from `lib/`.

When you find a `lib/` file with `await import('../../api/...')` inside
a method body, treat it as a refactor target, not an established
pattern.

## Manager vs Api

The codebase uses two distinct class shapes for non-domain behavior. The
distinction is **how privileged the class is**, not how stateful.

### Manager — Privileged Singleton

A `Manager` owns state, performs business logic, and is reachable only
from privileged code (other managers, `Backend`, `Application`, the Api
layer). Typical shape:

```typescript
export class XyzManager {
  private static instance: XyzManager;
  private someState: Map<string, SomeType>;

  private constructor() {}

  static get(): XyzManager {
    if (!this.instance) this.instance = new XyzManager();
    return this.instance;
  }

  // business logic methods
}
```

Examples: `PersistenceManager`, `ConnectionManager`, `Backend`.

### Api — Public Static Utility

An `Api` class is the public-facing entry point that mudlib code uses.
It's a class of static methods, decorated by
`SecurityApi.decorateApiClass(XApi)`. Some Apis are pure delegators
(thin wrappers over a Manager); others own significant logic
(`StuffApi`, `MixinApi`, `MessageApi`).

```typescript
export class XyzApi {
  static getSomething(id: string): SomeType | undefined {
    return XyzManager.get().getSomething(id);
  }
}

SecurityApi.decorateApiClass(XyzApi);
```

Examples: `StuffApi`, `ConnectionApi`, `MixinApi`, `MessageApi`,
`ContainmentApi`, `MqlApi`, `MqlSubscriptionApi`, `PromptApi`,
`MudlogApi`, `CommandApi`, `CommandLineApi`, `ProxyApi`,
`SecurityApi`, `ShadowApi`, `ExecutionContextApi`, `ModuleApi`,
`NavigationApi`, `PathPatternApi`, `ScheduleApi`, `SchedulerApi`,
`TemplateApi`, `ZoneApi`, `MmlApi`, `PlayerApi`,
`PersistApi` (the gated chokepoint over `PersistenceManager`,
`lint:pm`-locked — see [persistence.md](./subsystems/persistence.md)),
`RenownApi`, `ForumsApi`, `SubjectApi` (the latter two each forwarding
to a `ForumsLogic` / `SubjectLogic` logic singleton; see
[forums.md](./subsystems/forums.md)), `ConsumerApi` (forwarding to
`ConsumerLogic`; the consumer influence stock — see
[participation.md](./subsystems/participation.md)), `ProducerApi` (→
`ProducerLogic`; the producer/make stock), `ConvictionApi` (→
`ConvictionLogic`; the conviction spend half), `ProvenanceApi` (→
`ProvenanceLogic`; the authorship ledger) — see
[influence.md](./subsystems/influence.md) and
[provenance.md](./subsystems/provenance.md). `InfluenceApi` is a thin
cross-stock *dispatcher* with no logic singleton (it delegates to the
per-stock Apis). `SocialApi` (→ `SocialLogic`; the attention-management
layer — the shared `ruleFor` resolution, the display-lensing occupant
formatter, the login presence relay — see
[social-graph.md](./subsystems/social-graph.md)). `StudioApi` (→
`StudioLogic`; the CMS **composition** surface — effective-mixin-set + authorable-field
description, the blueprint catalogue, and the class scaffold/commit path — the
`CmsApi`/`CmsLogic` twin; see [studio.md](./subsystems/studio.md)).

`MqlSubscriptionApi` is the second wire channel alongside prose /
dispatch-response. Inbound `mql-subscribe` / `mql-unsubscribe`
messages route through `Application.processUserMessage` to the
substrate, which projects an MQL query's result + re-projects on
relevant state changes. Outbound envelopes ride the same
`MessageApi.sendEnvelope` path the dispatch-response framework uses.
See `docs/subsystems/mql-subscription.md`.

`PromptApi` is the third wire-side substrate. Server callers
`await` an interactive prompt (`choice` / `confirm` / `text` /
`mqlObject` / `mqlMany`); inbound `prompt-response` /
`prompt-cancel` messages route directly to the substrate (bypassing
the command bus); outbound envelopes ride the same
`MessageApi.sendEnvelope` channel as dispatch-response and
subscription deltas. Disconnect cleanup cancels MQL subscriptions
THEN prompts THEN removes the Interactive (so envelopes can
address the Interactive throughout the cancellation). See
`docs/subsystems/prompt.md`.

### When to Create a New One

- **Manager** when you need a singleton with privileged state and
  business logic; access restricted to core infrastructure.
- **Api** when you need a public interface; stateless or
  safely-cacheable; mudlib should be able to call it.
- **Both** when you have privileged state/operations *and* need safe
  mudlib access — the Api wraps the Manager.

### Notes on the Pattern

- **Backend is I/O only**, not a database passthrough. `Backend.send`,
  `Backend.broadcast`, and `Backend.handle*` are the surface.
  Persistence calls go through `PersistenceManager.get()` directly from
  privileged code.
- **`StuffApi` does own state** (the `objectsById` registry). That's
  the documented exception — registry membership is a public concern.

## Backend → mudlib import discipline

Backend is the privileged layer; mudlib is the upper layer. Imports
flow upward (mudlib → Api → Manager). Backend code that reaches
into `mud/` for shape or data inverts the layering and should be
held to a tiered standard:

1. **`mud/api/*` — always free game.** Apis are the engineered
   handshake; backend Managers naturally call into them. Importing
   `StuffApi`, `EventApi`, `ExecutionContextApi`, etc. is the
   intended pattern.

2. **`mud/lib/*` types via `import type` — fine.** Shape information
   only; no runtime dependency. `import type { Stuff, User,
   Interactive, CommandContext }` from backend is normal — these
   are framework primitives, not gameplay. Runtime `import { ... }`
   of a `mud/lib/` value is a smell unless it's a cross-cutting
   framework primitive (`CallSecurity`, `SecurityPolicies`).

3. **`mud/lib/*` runtime values that are mudlib data or
   configuration — invert it.** When a manager needs a list, table,
   or manifest authored in mudlib (so lower-level developers can
   edit it), the *type* belongs to the manager (the consumer); the
   *data* belongs in mudlib and imports the type back. Mudlib →
   backend for the type, never backend → mudlib for the type.

   `BootstrapManager` + `mud/bootstrap.ts` is the worked example:
   `BootstrapEntry` is exported from `BootstrapManager`;
   `mud/bootstrap.ts` imports it and declares the manifest array
   against it. Backend doesn't reach into mudlib for the shape;
   mudlib doesn't have a competing definition.

4. **`mud/obj/*` classes — tech debt.** `instanceof Avatar`,
   `Avatar.getTemplatePath()` from `Application` couples backend to
   gameplay. Pragmatic for now (the network → gameplay bridge has
   to land somewhere) but flag and migrate to mixin predicates
   (`MixinApi.isHasInteractive`) and Api lookups when the surface
   stabilises.

For load-time cycle breakage, `await import(...)` inside a method
body (see `PersistenceManager.dispatchSave` and friends) is the
established escape hatch — keeps the static graph clean while the
runtime call still resolves.

## The import boundary

The rule in one line: **nothing under `src/mud/` imports anything from
outside `src/mud/` — Node built-ins included — except the Api layer,
which imports and wraps.**

This is the classic driver/mudlib split stated for our tree. Mudlib code
**cannot import a capability**: no filesystem, no network, no child
process, no code-eval. It asks the gated Api surface, or it does without.
(The rule checks imports; see *What this rule does not cover* below for
what that leaves open.) The
rule is the import-graph twin of call-security — call-security governs
*who may call what at runtime*, this governs *what a module can reach at
all* — and together they are what makes the sandbox and wizard
code-trust story checkable rather than aspirational. Authored code that
can only import mud modules can only do what the Apis gate.

Enforced by `pnpm lint:imports` (`scripts/check-mud-imports.ts`),
CI-gating. A script rather than an ESLint rule for the same ESLint-8
reason as `check-gate-strings`, and because `no-restricted-imports`
cannot express "a relative import that escapes this subtree" without a
resolver plugin. `--report` prints crossings grouped by module and file.

### The tiers

| Tier | Files | May import |
|---|---|---|
| **mudlib** (the default) | `lib/`, `obj/` outside `obj/api/`, `cmd/`, `domain/`, … | relative imports resolving inside `src/mud/`, `@saxonberg/types`, and any `import type` |
| **Api** | `mud/api/**` **and** `mud/obj/api/**` | the above, plus an enumerated set of Node built-ins and npm packages, plus `backend/` |
| **test** | `**/__tests__/**` | unrestricted |

The Api tier is **both halves of the `XApi` ↔ `XLogic` split**, not just
`api/`. `api/` is the thin non-HMR forwarding shell; the logic singleton
is where a capability actually gets used (`CommandLogic` enumerating YAML
views, `PackLogic` reading pack files, `GitLogic` driving `simple-git`).
Exempting only `api/` would force the wrap to forward *backwards* into
logic that could not do the work. This tier is already the `@internal`,
gated one — the boundary and the doc-visibility boundary coincide, which
is the `callable == visible == cared-about` invariant again.

Two deliberate softenings:

- **`import type` is exempt everywhere.** It is erased at compile time
  and confers no runtime capability. The rule is about capability, not
  vocabulary — `import type { BootstrapEntry }` from backend is fine.
- **Both allowlists are enumerated, not open.** A new built-in or npm
  dependency in the Api tier is a deliberate edit to the script. That is
  the tripwire that forces the conversation, the same discipline as the
  export-discipline registry. Driver-level packages (mongodb, express,
  ws, passport) are on no tier's list: they belong to `backend/`.

Dynamic `import()`, `require()` and `createRequire` ride the same matrix
— a soundness hole there would make the whole rule decorative.

### Naming a file without reaching for one

The recurring mudlib need is "load the authored data file that ships next
to me" — a char-gen roster, a theme catalogue, a command schema. The
answer is the synchronous shipped-resource face on `SourceTreeApi`:

```typescript
SourceTreeApi.readYamlResource<CharGenConfig>(
  import.meta.url,
  '../../../config/char-gen.yaml'
);
```

`import.meta.url` is a **language construct, not an import**, so a mudlib
module can name its own location without leaving the tree; the read
happens in the Api tier, sandbox-checked. Siblings: `readResource`,
`readJsonResource`, `parseYaml` (for text already in hand — a command
argument, a CMS field), and the pure path arithmetic `toMudPath` /
`resolveFrom`.

### No exceptions

The registry in `check-mud-imports.ts` is **empty**, and the rule is
enforced with no carve-outs at all. Getting there took one more pass than
expected, and the pattern that dissolved every hard case is the same one:

> When a mudlib module holds a capability, the part that *needs* the
> capability is almost always smaller than it looks — and the part that
> doesn't is the part worth keeping in the mudlib.

Four modules looked like irreducible exceptions because their whole
purpose was the library they imported. Each split cleanly anyway:

| Module | Held | Now |
|---|---|---|
| `lib/script/EvalScript` | `node:vm` | `ScriptApi.compileSandboxed` / `runSandboxed` over an opaque handle. The **sandbox allowlist and the receiver bindings stay in the mudlib** — deciding what the sandbox contains is policy, and assembling it is plain object work. Only `createContext` / `new Script` / `runInContext` moved. |
| `lib/prose/Prose` | `liquidjs` | The Liquid engine and its filter set moved to `ProseLogic`; `Prose` kept the value-object half (source string, opaque compiled handle, Mml-typed render). |
| `lib/persistence/EncryptedStringMarshaller` | `crypto` | `PersistApi.sealString` / `unsealString` — encrypt-at-rest is a persistence concern. The marshaller kept the envelope validation and the `Marshaller` shape. (A standalone `CryptoApi` was considered and rejected long before this; folding into the owning subsystem's face is what that decision implies.) |
| `lib/command/CommandDefinition` | `ajv` | `CommandApi.validateCommandView` — the command subsystem validating its own spec format. The value object kept every structural invariant it checks *after* the schema. |

The two persistence-framework files (`lib/persistence/Document`,
`lib/stuff/Template`) that imported `PersistenceManager` on the grounds
that they *are* the data layer now route through `PersistApi`, whose
surface already covered every call they made — a migration its own
docstring had anticipated. That also let `check-pm-access.ts` drop two
allowlist entries: **the mudlib now has no path to persistence except
the facade.**

The recurring shape of the fold is an **opaque handle**: the Api compiles
or seals, hands back a branded type with no structure
(`CompiledSandbox`, `CompiledProse`), and the mudlib holds it only to
hand it back. The mudlib keeps the policy and the vocabulary; the Api
keeps the capability.

**Ask before adding the first exception.** The mechanism is still there,
and a genuine one is a one-entry edit with its reason recorded — but the
answer has so far always been "fold it into an Api," and the lint failing
is the tripwire that forces that conversation.

### What this rule does not cover

It governs **imports**, so ambient globals stay reachable from the
mudlib: `process` (and `process.env`), `Buffer`, `console`, `globalThis`,
and timers. Some of those are load-bearing and fine (`Buffer` is inert
data handling; `ScheduleApi` already owns timers by a separate rule).

`process.env` is the one worth naming, because mudlib code reads it
today: `obj/AccessRegistry.ts` (the wizard / archwizard / streamer
allowlists), `obj/OfficeRegistry.ts` (the founder identity), and
`obj/ReactionRegistry.ts` (`NODE_ENV` / `VITEST`). Those are deliberate
and unaffected by this rule — but note the shape: a module that can read
`process.env` can also write it, and `globalThis` reaches further still.
Closing that needs a different mechanism (a lint on identifiers, or a
real module sandbox), not this one.

So: a strong **architectural** boundary — it makes the capability
surface finite, reviewable, and gated — but not a security perimeter.
Read the two claims separately.

**Tests are blanket-exempt** (`**/__tests__/**`), deliberately: they are
not shipped mudlib and already reach for white-box seams by design. If
that wants tightening later, the target is banning `../backend/` *value*
imports from test files (~10 at the time the rule landed) while leaving
`import type` and the test-only deps alone.

## Class Hierarchy

### Top-level branches

Every concrete `Stuff` subclass MUST extend through exactly one of
five top-level branches, each capturing a distinct role:

```
Stuff (base — runtime ID, FINAL destroy, construction sentinel)
  ├── Idea          incorporeal identity (Exit, Login, Zone, …)
  ├── Thing         physical matter (an item; describable, Tangible, Wet)
  │     └── Vessel  a Thing that also holds things (Container + interior
  │                 Atmospheric) — matter outside, a place inside
  ├── Location      stationary place — pure space, NOT matter (not Tangible)
  ├── Agent         runtime active object (Creature → Character → Avatar)
  └── Shadow        function-shadowing host — see call-security.md
```

The space/matter axis is load-bearing: `Thing` (and its `Vessel`
subtype) and `Agent` are **matter** — they compose `TangibleMixin`
(material + mass) and therefore `WetMixin` (can get wet). `Location` is
**space** — a room has no material or mass, so it is deliberately not
`Tangible`. `Vessel` is the one dual citizen: matter you can also be
*inside*. (A `Vessel` is not its own branch — it traces through `Thing`;
"you can't pocket a ship" is a mass gate, not a type gate.)

Under `Agent` the hierarchy splits **body** from **agent**:
`Agent → Creature → Character → Avatar`. `Creature` (`lib/creature/`)
is the body layer — a living physical thing that can break, with or
without agency: it carries `PropertiedMixin` (innermost, just outer of
`Agent` — the general dynamic per-instance property store, so every body
/ Character / Avatar can hold state other objects park on it; e.g. the
banking Circle-membership flag rides a `<corpoKey>.circle` prop here) +
`OrganismMixin` + `VitalsMixin` +
`ReservedMixin` + `MetabolicMixin` (intake/chemistry driver, outer of
vitals/reserve) + `RespirationMixin` (the air-exchange / `spo2` death
driver, outer of metabolism) + `ThermalMixin` + `ThermalRegulationMixin`
(the heat-exchange substrate + Option-C body driver, outer of
respiration/metabolism; drives `coreTemperature`) + `LoadBearingMixin`
(the encumbrance gauge) + `ConcealableMixin` (presence-
concealment, so a lurking creature can be hidden — see
[concealment.md](./subsystems/concealment.md)) + `PostmortemMixin`
(outermost — the clock a body runs on AFTER it stops: decay stages,
forensic readability, and the `canEvict` veto that keeps a corpse in the
world long enough to be examined; inert on the living — see
[mortality.md](./subsystems/mortality.md)) + `DisguisableMixin`
(creature masking, outer of `Visible`) + the anatomy-slot / posture /
description / containment mixins. `Character`
extends it with the **agency** mixins (commands, perception, speech,
movement, engagement) + the social-identity mixins (`PersonaMixin`,
`GenderedMixin`) + the per-viewer concerns (`BeliefStoreMixin`,
`StatusMixin`). The split exists because **vitals are
body-state, not agent-state** (a corpse / sessile animal is a body with
reduced agency) — see [vitals.md](./subsystems/vitals.md). The identity
line is sex (body, `SexedMixin` on Creature) vs. gender/persona (social,
on Character). `Creature` is concrete, so a bare non-agent body (a frog,
a corpse) is valid. `Character` has two leaf subclasses: `Avatar`
(player-driven, in `obj/`) and the thin `NPC` (`lib/npc/NPC.ts` =
`Character` + `BehavedMixin`) for authored, automation-driven characters —
which keeps `Behaved` off players. See [behavior.md](./subsystems/behavior.md).

The five branch constructors are registered with `Stuff` **by class
identity** through `Stuff.registerTopLevelBranches([...])`, called once
from `BootstrapManager.installFrameworkWiring` (production boot's
first step, and the vitest setup file) — a lifecycle call, not a
module-scope statement (the no-module-scope-statements rule). Identity,
not module id: the vite-plugin, tsx, and the production node-hook stamp
default exports differently, so a module-id check that passes under one
loader silently fails under another — an identity comparison is loader-
agnostic and a foreign same-named class (a different constructor
object) never matches. The call is caller-gated to the framework wiring
layer, so the branch set can't be silently extended from content.

`Stuff`'s constructor walks the prototype chain at instance-creation
time and throws if no registered branch is found. Class-time work is
done once per class (cached); the per-instance cost is a single
WeakSet lookup. The error message lists the five branches and
points readers here.

> **Not a branch: `Document`.** Plain MongoDB-backed records (`User`,
> `GoogleProfile`, `Template`) extend `Document` (`lib/persistence/`),
> which is **not** in the Stuff hierarchy at all — no proxy, no
> registry, no lifecycle. A `Document` is value-like persisted state; a
> `Stuff` is an identity-like live entity *hydrated from* a Document.
> See [persistence.md](./subsystems/persistence.md).

### Why the invariant

- **Roles, not capabilities.** Class identity (`instanceof`) is for
  role checks (`instanceof Avatar`, `instanceof Vessel`); capability
  checks (`is this place-like / item-like / navigable?`) go through
  `MixinApi.isContainer` / `isContainable` / `isExitable`. The branches
  give every role a clean `instanceof` that doesn't lie.
- **Vessel is a `Thing`, not its own branch.** A container-object (bag →
  cart → ship) *is* physical matter — describable, `Tangible`, `Wet`,
  carryable, `Containable` — that additionally holds an interior
  (`Container` + `Atmospheric`). So it extends `Thing` and adds those two
  mixins: matter from the outside, a place from the inside. "You can't
  pocket a ship" is a **mass** gate (encumbrance), not a type gate —
  nothing consumes `instanceof Thing` — so the earlier "mobile place,
  sibling of `Location`" framing bought nothing and was dropped in review.
  `instanceof Vessel` still identifies the vessel role (it is still a
  distinct class).
- **Persisted records are NOT a branch.** Auth/CMS records (`User`,
  `GoogleProfile`, `Template`) are plain `Document`s, outside the Stuff
  hierarchy entirely — they only need to persist, not to participate in
  the world's method/event/shadow/perception machinery, so they don't
  pay the proxy/registry/lifecycle cost. (An earlier design made them a
  `Persistable extends Idea` branch; that's been retired — see
  [persistence.md](./subsystems/persistence.md).)

### What each branch composes

| Branch | Composition | Notes |
|---|---|---|
| `Idea` | `Stuff` | No spatial mixin. Default for incorporeal entities. |
| `Thing` | `Concealable(Wet(Visible(Perceptible(Tangible(Containable(Stuff))))))` | Physical matter: describable, `Tangible` (material + mass), `Wet` (can get wet), `Containable` ("I live somewhere"), and `Concealable` (default `obvious` — the presence-concealment axis so any Thing can be hidden, see [concealment.md](./subsystems/concealment.md)). |
| `Location` | `Addressable(Atmospheric(Adornable(Container(Stuff))))` | "I'm a place." Pure *space* — NOT `Tangible` (a room has no material/mass). Subclasses (`CartesianLocation`, …) layer on coordinate / Visible / Exitable mixins. |
| `Vessel` | `Atmospheric(Container(Thing))` | **Extends `Thing`** — a container-object that adds `Container` (holds things) + `Atmospheric` (interior climate) on top of Thing's matter baseline. A describable container is a plain `DetailedMixin(Vessel)`. `ExitableVessel` etc. layer on navigation. |
| `Agent` | `Stuff` | Subclasses (Character → Avatar) layer on Mobile / Container / Containable / Sensor / Vocal / etc. |
| `Shadow` | `Stuff` (abstract) | Framework-internal — not in-world Stuff. See [call-security.md](./subsystems/call-security.md). |

Persisted records (`User`, `GoogleProfile`, `TwitchProfile`, `KickProfile`, `Template`,
and the forums-build records `Subject` / `Board` / `Entry` / `Vote` /
`ForumEvent`) are **not** Stuff — they extend the standalone `Document`
base (`lib/persistence/`), loaded via `findById` / `find` rather than the
template/clone/hydrate pipeline, with none of the proxy/registry/
lifecycle machinery. See
[subsystems/persistence.md](./subsystems/persistence.md). The forums
build also adds two singleton registries — `SubjectCatalogue` (the
Subject-layer index) and `ForumSubscriptionRegistry` (the `forum_events`
document-change observer) — and the `ForumsUpdate` `AetherHosted`
implant carrying `ForumsMixin`; see [forums.md](./subsystems/forums.md).

The CMS-composition (**Studio**) build adds `Blueprint` (`lib/studio/`) — a
reference-data `Document` (`blueprints` collection, the `Recipe` precedent,
never cloned: a named structural composition `<baseClass>|<sorted mixins>`) —
plus the boot-warmed `BlueprintCatalogue` singleton (`obj/`, the
`RecipeCatalogue` shape: id + signature indices) it's loaded into, and the
`BlueprintSeeder` (`backend/`, a derived skeleton from every backing class + a
curated `config/blueprints.yaml` overlay). See [studio.md](./subsystems/studio.md).

The **concealment / traps** build adds two capability mixins and one class,
both deliberately **without a new Api**: `ConcealableMixin`
(`lib/concealment/`, the presence-concealment carrier, baseline on
`Thing`/`Creature`/`Exit`) — its detection *surface* is folded onto the
existing `PerceptionApi`/`PerceptionLogic` (no `DetectionApi`, since
detection *is* concealment-gated perception); and `HazardMixin`
(`lib/hazard/`, a **self-resolving** trap capability) plus the concrete
`Trap = HazardMixin(DetailedMixin(Thing))` — with **no `HazardApi`/
`HazardLogic`**, because a hazard owns its own state + resolution and the
powerful steps it orchestrates (`ConditionApi.inflict`,
`PerceptionApi.perceives`) are each already gated, so an orchestrator gate
buys nothing. Both are the "an Api is earned by a *powerful primitive*, not
by every feature" rule read the other way — a capability with nothing new to
protect stays a mixin. See [concealment.md](./subsystems/concealment.md) /
[hazard.md](./subsystems/hazard.md).

The **stealth & deployables** build (the actor-face of concealment) adds
`HidingMixin` (`lib/concealment/`, a `Character`'s dynamic `hide` state,
composed *outside* `Creature`'s `ConcealableMixin` so it overrides
`getConcealment()`) and the `TrapKit` class (`lib/hazard/`, the carried
deployable), and — earning an Api by the *powerful-primitive* rule — the
gated `AccountabilityApi`/`AccountabilityLogic` pair (`lib/accountability/`,
the unified harm-consent ledger combat migrated onto and the trap feeds).
The hide-*level* rule and the observer-side `motionExposure` fold onto the
existing `PerceptionApi` (no `StealthApi` — mirroring "no `DetectionApi`"),
and MQL's scope-walk gained a code-only `MqlContext.attention` term so a
detection consumer resolves `peers` at an active attention. See
[stealth.md](./subsystems/stealth.md) /
[accountability.md](./subsystems/accountability.md).

## Mixin Organization

Mixins are higher-order functions that extend a base class with new
fields and methods. The Saxonberg convention is strict about where
they live and how they're declared.

The full subsystem doc is [subsystems/mixins.md](./subsystems/mixins.md)
— infrastructure (`MixinConstructor`, `Mixins` registry, `MixinApi`),
authoring conventions, composition rules, and cross-cutting
integration. The summary below covers the layout rules every
contributor needs to know.

### Subsystem-Folder Layout

Mixins live in the `lib/<subsystem>/` folder that owns the concern they
model, alongside any related classes. **There is NO `lib/mixins/`
folder** — that grouping is explicitly prohibited. "Mixin" is an
implementation technique, not a subsystem. If a new mixin doesn't fit
an existing subsystem folder, propose a new subsystem folder for it
rather than grouping by mixin-ness.

Shared mixin infrastructure (`MixinConstructor` type, `Mixins` name
registry) lives in `lib/mixin.ts`.

### Available Mixins

| Folder | Mixin | Purpose |
|---|---|---|
| `lib/chattel/` | `EstateMixin` | owner-based persistence — every stamped good its host holds title to, wherever it sits; routes restore on `place` (furnishing.md) |
| `lib/character/` | `GenderedMixin` | pronouns (he/she/they/etc.), persistent |
| `lib/description/` | `NamedMixin` | proper names — `name`, `surname`, `nameSuffix`, `honorific`, `alternateNames`, `fullName`, persistent |
| `lib/description/` | `VisibleMixin` | shortDescription, longDescription; provides `look` command |
| `lib/description/` | `PerceptibleMixin` | MQL keyword management, persistent |
| `lib/description/` | `DetailedMixin` | hierarchical detail management, persistent |
| `lib/spatial/` | `ContainerMixin` | inventory; provides `inventory`/`get`/`drop` |
| `lib/spatial/` | `ContainableMixin` | environment reference, plus the auxiliary `restingOn` pointer for on-surface placement |
| `lib/spatial/` | `SurfacedMixin` | "things rest on this" host-side marker; lazy `getResting()` walk; `userFacingDetail` MQL bridge; `canRest()` per-host gate. Requires Containable. |
| `lib/spatial/` | `MobileMixin` | `travel()` between locations (requires Containable) |
| `lib/spatial/` | `ExitableMixin` | exit map host; `addExit`, `getObviousExits`, etc. |
| `lib/location/` | `CartesianCoordinatesMixin` | `[x,y,z]` position carrier |
| `lib/location/` | `SphericalCoordinatesMixin` | `{rho,theta,phi,radius}` position carrier |
| `lib/location/` | `WarrenMemberMixin` | optional member-side back-ref to a `Warren` (the MultiLocation elastic-graph coordinator); Pattern-B live ref, Warren-owned. See [location.md](./subsystems/location.md). |
| `domain/lounge/` | `LoungeMixin` | lounge-room behavior (self-register, population witnesses, over-capacity re-seat); requires `WarrenMemberMixin`. A content mixin under `/domain/lounge/`, not the generic substrate. |
| `lib/spatial/` | `SealableMixin` | open/closed state (doors) |
| `lib/` | `BistateMixin` | shared guarded-boolean base (typeof-boolean guard, round-trip persistence) under `Sealable` / `Switchable` / `Foldable`. **Unregistered / unmarked** — an implementation base, not a registry mixin. See [boundary.md](./subsystems/boundary.md). |
| `lib/boundary/` | `SwitchableMixin` | generic two-state on/off toggle over `BistateMixin`; the `device`-category `switch`/`toggle` verb. Consumed by `Beacon` (walk/stop). See [boundary.md](./subsystems/boundary.md). |
| `lib/boundary/` | `LockableMixin` | lock state over `Door` (`Lockable`, `MixinApi.isLockable`); `Exit.canTraverse` vetoes with gate `'locked'` **before** destination resolution (reads off `this.door` in hand, so a locked gate can point at an unbuilt far side). A **stopgap** superseded by build-3's `lib/lock/` Lock+Key model; `lock`/`unlock` minimal, keyed-credential model deferred. See [boundary.md](./subsystems/boundary.md). |
| `lib/slot/` | `FoldableMixin` | two-state fold/unfold over `BistateMixin`; a folded host refuses its sit slot (`canOccupy` gate). The `device`-category `fold`/`unfold` verbs (reachable-scope, `MixinApi.isFoldable`-narrowed). `FoldingChair` first driver. See [slot.md](./subsystems/slot.md). |
| `lib/time/` | `TimekeepingMixin` | the display seam — an in-world object that reads/shows game-time via `WorldClockApi`. (Accurate by default; drift is a *content* concern layered on top — see the locality `MechanicalMovement`. The clock tower is prose, not a Timekeeping Stuff.) See [time.md](./subsystems/time.md). |
| `lib/spatial/` | `DoorBearingMixin` | adds `door: Door \| null` for hosts whose exits are synthesized rather than authored (`ExitableVessel`). Constrained to `Stuff & Exitable`. |
| `lib/stuff/` | `SingletonMixin` | class-level uniqueness — refuses a second `clone()` for the same templatePath. Composed by `CartesianZone` / `SphericalZone`. |
| `lib/stuff/` | `PopulatesMixin` | declarative content-spawn for Container hosts; `populates:` instruction field lists entries to clone (non-singletons) or singleton-resolve into self — each a bare templatePath (moved in) or a `{template, onto}` object (placed on an already-populated sibling surface via `placeOn`). Phase 2 applier. |
| `lib/message/` | `SensorMixin` | `handleMessage(frame)` notification hook |
| `lib/message/` | `VocalMixin` | `say(text)` with scope inference |
| `lib/command/` | `CommandGiverMixin` | `executeCommand`, `getAvailableCommands`, `getAffordances` |
| `lib/stuff/` | `PropertiedMixin` | controlled dynamic property bag, persistent |
| `lib/stuff/` | `PostRegistrationMixin` | opt-in `postRegister(context?)` lifecycle hook |
| `lib/persistence/` | `AroundSaveHookMixin` | middleware-style PM save hook |
| `lib/persistence/` | `AroundDeleteHookMixin` | middleware-style PM delete hook |
| `lib/connection/` | `HasInteractiveMixin` | "this Stuff has connected `Interactive`s" — `add`/`remove`/`getInteractives`/`isConnected`/`isLinkdead`. Composed by `Avatar` (multiplexing) and `Login` (singleton). |
| `lib/activity/` | `EngagedMixin` | actor-side engagement slot map (`body`/`hands`/`attention`/`voice`); runtime-only. `_setEngagement` / `_clearEngagement` are ApiOnly-gated; only `SchedulerApi` may mutate. Composed by `Character`. Provides the `cancel` verb and `stop` default alias. |
| `lib/biome/` | `AtmosphericMixin` | biome ref + atmospheric overrides (temperature/pressure/humidity/gravity/atmosphere) at room or per-Detail scope. Composed by `Location` and `Vessel`. See [biome.md](./subsystems/biome.md). |
| `lib/biome/` | `SkyExposedMixin` | trait stamp for biomes whose Locations look out on the open sky. Composed by `SkyExposedBiome`. |
| `lib/address/` | `AddressableMixin` | optional `_address` declaration (Pattern-A path string) placing a host in the addressing namespace; sparse, `null`-default. Read by the `AddressApi` resolve-walk; not a ref-resolver (resolution is the Api's job). Composed by `Location`. See [address.md](./subsystems/address.md). |
| `lib/perception/` | `SmellSourceMixin` | "this Stuff emits an odor"; `getEmittedConcentration()` (ppm `Quantity`) + `getOdorIdentity()` (string). Composed by smelly Thing templates (candle, garlic, etc.) and fixture-side Adornments. See [senses.md](./subsystems/senses.md). |
| `lib/perception/` | `SoundSourceMixin` | "this Stuff emits sound"; `getEmittedAmplitude()` (dB `Quantity`) + `getSoundCharacter()` (string). Composed by noisy Thing templates and fixture-side Adornments. |
| `lib/perception/` | `AudibleMixin` | the **discrete-event cross-room sound push**: `emit()` runs the `AudienceGather` fork of `SoundModality.walkAt` (reuses the exit-walk + `SoundConduit` boundary transmissivity + dB attenuation + `MAX_HOPS`) to collect `(sensor, attenuated-dB)` pairs, then `Scene.toAudible` pushes one directional frame per hearer (lights the previously-inert `meta.acousticDb` seam). Same-room = zero-hop case. Whistle first driver. No new Api. See [perception.md](./subsystems/perception.md). |
| `lib/augmentation/` | `AugmentMixin` | "this Stuff is an installable augment"; declares `confers(): readonly string[]` listing mixin names activated when installed. Wave 1 vocabulary surfaces `_augmentGated` / `_grantsModalities` on the mixins themselves. See [augmentation.md](./subsystems/augmentation.md). |
| `lib/augmentation/` | `AetherHostedMixin` | the *update* side of the aether hosting relation: a `getHost`/`setHost` back-ref + the must-be-hosted invariant, composed around an `Idea` (the incorporeal capability base). Co-composed with a capability mixin on an update class (`CommsUpdate`, `CredentialWalletUpdate`). See [augmentation.md](./subsystems/augmentation.md). |
| `lib/message/` | `AetherMixin` | **attunement** + the aether **host**: perceive the Aether (contributes the `verbal-esp` / `emotive-esp` modalities to `PerceptionApi.sensorium`) and host capability *update* `Idea`s (`getHostedUpdates` + sealed `hostUpdate`/`unhostUpdate` chokepoints). Augment-gated (`_augmentGated = true`); conferred by the `AetherImplant` **or** intrinsically by `Species.innateMixins`. No longer carries transmission — that moved to `CommsMixin`. See [augmentation.md](./subsystems/augmentation.md). |
| `lib/comms/` | `CommsMixin` | the comms transmission capability (the `dm`/`reply`/`broadcast`/`chat` verb family + DM cohort state), composed on a hosted update (`CommsUpdate`). `tell` sends on behalf of its host (the operator) via `getHost()`. See [comms.md](./subsystems/comms.md). |
| `lib/vitals/` | `VitalsMixin` (+ `DressingMixin`) | body-state: vital-sign `Quantity` fields, per-species survivable-band lookup, derived `getConditionBand` / `getConsciousness` (computed, never stored), the anatomy resolver, the active-condition collection, and the death/consciousness seams — now including the **dying clock** (`DyingRecord`, the `dying` band, `beginDying`/`stabilize`, and the reconcile arm that deliberately does NOT freeze on linkdead) and the fork-only **material slices** + gated `adoptMaterialState` that make a corpse un-reanimatable. Also hosts the **reconcile-on-read wound driver** (`reconcileConditions` on the read path, driving the harm subsystem's bleed/heal/`exsanguination` off a persisted per-trauma `tickedAt`) + `isSlotImpairedByTrauma` / `drainForLimp`. `DressingMixin` is the first-aid dressing capability (`Bandage`); the harm producer facade is `ConditionApi`/`ConditionLogic`. Requires `OrganismMixin`. Composed by `Creature`. See [vitals.md](./subsystems/vitals.md) / [harm.md](./subsystems/harm.md). |
| `lib/mortality/` | `PostmortemMixin` | what a body does AFTER it stops: the decay clock (staged, degrading forensic readability while the cause stamp stays ground truth), and the `canEvict` veto that keeps a corpse in the world until it is spent. Runs unguarded — a corpse has no player to protect. Composed by `Creature`; inert on the living. See [mortality.md](./subsystems/mortality.md). |
| `lib/mortality/` | `IncorporealMixin` | present, but unable to touch anything — the capability half of function-over-form. Platform verbs ride the participant; embodied verbs are refused by the `requiresEmbodied` validator. Carries the refusal prose so the same lever re-skins (the deferred prison work). Composed by `Shade`. See [mortality.md](./subsystems/mortality.md). |
| `lib/reserve.ts` | `ReservedMixin` | a keyed collection of `Reserve` capacity axes (decomposed-scalar persistence). Biological reserves (endurance/satiation/hydration) + the authored-thematic seam (mana is content; a cultivated plant's root-zone `moisture` is another). **Neutral, not Creature-coupled** — composed by `Creature`, `Campfire` (fuel), a `Location` (air) and `Plant` (moisture). See [reserve.md](./subsystems/reserve.md). |
| `lib/encumbrance/` | `LoadBearingMixin` | the carry-weight gauge (first vitals driver): derived-on-read `getBorneBurden` (weighted walk over contents + slot occupants with `Vessel.transmissionFactor` + slot-derived placement coupling) / `getCarryCapacity` (body mass × physiology margins) / `getLoadRatio` / `wouldExceedCeiling` / `drainForTraversal`. Requires `Container + Slotted + Tangible + Reserved + Vitals`. Composed outermost by `Creature`. See [encumbrance.md](./subsystems/encumbrance.md). |
| `lib/belief/` | `BeliefStoreMixin` | per-viewer identity memory: a realm-namespaced (`recognition` / `identification` / `regard`) keyed bag of `BeliefRecord`s, dumb CRUD, keyed by referent `templatePath`. The in-memory working set behind `RecognitionApi.describe` (naming) and `RegardApi`/`RegardLogic` (per-viewer attitude scalar); backed by `BeliefDocument` rows (`api/belief.ts`). Composed by `Character`. See [belief.md](./subsystems/belief.md). |
| `lib/status/` | `StatusMixin` | settable activity-status line ("watching the empty road"); verb / runtime-setter / static-authored-default sources, only the default persists. A **presence affix, not identity**: it rides `RecognitionApi.describeWithStatus` (the presence-scan roll-call) — **not** `getPresentation` / `describe`, which stay pure identity, so act-subject naming never drags the status along. Composed by `Character`. See [belief.md](./subsystems/belief.md). |
| `lib/disguise/` | `DisguisableMixin` / `DisguiseBearingMixin` | creature masking. `DisguisableMixin` (on `Creature`) resolves a viewer-blind `getDisguise()` over worn `DisguiseBearing` garments + a transient imposed slot; `Stuff.getPresentation()` defers to it. `DisguiseBearingMixin` (on a `Garment` → `DisguiseGarment`) carries the `{ appearsAs, covers, masksIdentity }` descriptor. See [belief.md](./subsystems/belief.md). |
| `lib/identification/` | `IdentifiableMixin` | the type axis: an item whose true type (`identifiedName`) is hidden behind its unidentified appearance until a viewer identifies it. Composed by `IdentifiableThing`; the `IdentifyScroll` carries the `identify` verb. See [belief.md](./subsystems/belief.md). |
| `lib/metabolism/` | `MetabolicMixin` | the intake-and-chemistry driver (first condition-driver): the digestion buffer + real `ingest`, the lazy reconcile-on-read over `WorldClock` game-time (absorption / mass-scaled basal drain / coupled recovery / toxin clearance), the cascade spawning `floorEffect` conditions + the death seam, the presence-freeze clock, and the toxin-burden + alcohol/BAC system. Drives `Vitals`/`Reserved`/`Posed`; composed inner of `LoadBearing`, outer of those three, by `Creature`. No Api. See [metabolism.md](./subsystems/metabolism.md). |
| `lib/metabolism/` | `NutritionLabelMixin` | opt-in consumable affordance: appends an edible `Material`'s inspectable nutrition profile to the host's long description via the `markupAugmenter` seam. Composed by content onto labelled consumables (not every Stuff). See [metabolism.md](./subsystems/metabolism.md). |
| `lib/respiration/` | `RespirationMixin` | the air-exchange driver and the first concrete engagement producer: an event-triggered bounded `RespirationDrain`/`RespirationRecovery` `SustainedEngagement` that drives `Vitals.spo2` past the consciousness floor to the anoxia death seam when the surrounding medium is unbreathable (drowning / vacuum), then recovers on return to air. Reads `BodyPlan.breathableMedia` (water-breather inversion) + the biome `breathable` column; W2 taps a worn `AirTank` `Bulkable`. Composed outer of `Metabolic` by `Creature`. No Api. See [respiration.md](./subsystems/respiration.md). |
| `lib/thermal/` | `ThermalMixin` | the generic heat-exchange capability: lazy Newton's-cooling-on-read (mirrors `Metabolic`) with a **sync** `getTemperature()` via a cached ambient refreshed at re-stamp events (`onMoved` / ambient-shift fan-out / seal toggle / bulk transfer); τ = R·C from `Tangible` mass × `Material` specific heat + the medium/wall conductivity series; a sealed `Sealable` host → vacuum barrier. Composed by any Tangible+Containable Stuff (`Receptacle`/`Flask`/`Campfire`) and by `Creature` (corpse algor mortis). No Api. See [thermal.md](./subsystems/thermal.md). |
| `lib/thermal/` | `ThermalRegulationMixin` | the Option-C body driver: overrides `getVitalSign` (sync, cached effective ambient) to **drive** `coreTemperature` — pin at setpoint within the thermoneutral band, else spend satiation (cold) / hydration (hot, wet-bulb-capped) to defend it, fail into passive `Thermal` drift; endo/ecto split (`BodyPlan.thermalStrategy`) + Q10; the hypothermia/hyperthermia/torpor cascade → death seam. Composed over `ThermalMixin`/`Metabolic`, inner of `LoadBearing`, by `Creature`. No Api. See [thermal.md](./subsystems/thermal.md). |
| `lib/wetness/` | `WetMixin` | the cross-cutting wetness gauge (weather Wave 2): a per-object stored, decaying `0..1` saturation (decomposed-scalar persistence, reconcile-on-read drainage presence-frozen, pushed accrual, banded `dry/damp/wet/soaked`). Drives electricity (wet skin) + thermal (wet heat-loss); dry rate from real `Material.waterAbsorptionCapacity` via evaporation physics. Rides the **matter seam** — composed on `Thing` + `Agent` (so all matter, incl. `Vessel` via `Thing`, is wettable; `Location` is space, not matter). No Api. See [weather.md](./subsystems/weather.md). |
| `lib/husbandry/` | `GrowingMixin` | the living-world growth model (husbandry phase 1): a cultivated thing's condition as a pure function of `(profile, clock stamp, water, light, root room, interventions)`, reconciled on read over game-time with **no far-past guard and no linkdead freeze** (an owned thing lives the full absence — bounded by a step cap, never a time cap). Three satisfactions combined by the **minimum** (Liebig); a floored root curve makes a pot-bound plant stall AND hold a band; banded `thriving/healthy/stressed/failing/dead` with a separate plain-language cause line. Since phase 2 water lives in the SOIL, so the mixin declares SEVEN host seams (`rootRoom` / `soilMoisture` / `meanSoilMoisture` / `nutrientLevel` / `waterTheSoil` / `onFloweringLatched` / `sampleLux`), takes a **fourth** min() argument (`satNutrient`, `null` → 1, so a pot is unaffected), and records `_worstLimiting` — a monotone minimum that harvest quality reads, which is what makes farming reward your worst moment. Composed by `Plant`. No Api. See [husbandry.md](./subsystems/husbandry.md) and [smallholding.md](./subsystems/smallholding.md). |
| `lib/husbandry/` | `CultivableMixin` | **ground that holds plants** (living-world phase 2): a bulk interior of soil plus a plant slot whose authored `capacity` is N — *a pot is a bed with one slot*. Owns the shared-soil division (`rootRoomPerPlant` = soil ÷ occupied, prospective for `fitsSlot`), the soil's OWN checkpoint (`moisture` + `nitrogen` reserves, `soilClockStamp`, `reconcileSoil` behind its own reentry guard, draining by summed occupant demand), the populate-then-adopt applier, and `fixedGround` — the authored flag that decides whether land use applies (a bed is ground; a pot is furniture). Requires `Container` + `Bulkable` + `Slotted` + `Populates` + `Reserved` in its BASE, composed at the call site. Composed by `PlantPot` and `GardenBed`. No Api. See [smallholding.md](./subsystems/smallholding.md). |
| `lib/thermal/` | `MeltableMixin` | the phase-change capability (fire build): a solid + a latent-heat accumulator; `ThermalApi.reconcilePhase` holds a latent-heat plateau then melts it to a molten `Bulkable` pool in the scope `Floor` (the reverse boils / solidifies a liquid). Reads its material's `meltingPoint`/`latentHeatOfFusion`; gated `ThermalLogic` is the single writer. Composed by `Ingot`/`Casting`/`Candle`. See [fire.md](./subsystems/fire.md). |
| `lib/fire/` | `CombustibleMixin` | the combustion capability: a `'fuel'` `Reserve` + a `Burning` value-object active state; reads its material's `autoignitionTemperature`/`heatOfCombustion`, pins the flame temperature while aflame (generalized Campfire pin), reconcile-on-read fuel drain → char / structural burn-through. Gated `FireLogic` is the single Burning writer. Composed by `Firewood`/`Candle`. See [fire.md](./subsystems/fire.md). |
| `lib/fire/` | `FurnaceMixin` | the sustained-heat-source capability: a `Combustible`-fuelled appliance pinned hot (`burnTemperatureK × bellows`) while lit + fuelled, heating the Meltables in its scope (`heatContents`). Generalizes the Campfire pin (`Campfire` refactored onto it byte-identically); composed by `Forge`/`Kiln`/`Oven`/`Candle`. See [fire.md](./subsystems/fire.md). |
| `lib/magic/` | `CasterMixin` | the anatomical casting faculty (the magic build): the mana pool (an absolute avail/max `pt` `Reserve`, capacity from the species `facultyProfile`'s depth band), serenity-rate reconcile-on-read recovery (metabolism stamp guards), overchannel strain + hysteresis clear, the LIVE composure read (mental-resist substrate), the numbers-free `getFacultyView`, and the `cast`/`spells` affordance via `refreshCastingAffordance()` — the dynamic self-push at `Avatar.enter` (the `refreshConferrals` mirror; the self bucket collects class statics only, so a gated mixin cannot afford selectively any other way). Composed on `Character`, **gated** — active only when `Species.innateMixins` (or an augment) confers it (the `MakerMixin` activation shape). Read the pool via `getMana()`/`getManaFraction()` (the contract surface; raw keyed reads skip the reconcile). See [magic.md](./subsystems/magic.md). |
| `lib/magic/` | `ConduitMixin` | **the coupling an arcane reserve crosses to reach a shell** — the apparatus half of `recharge` (the magic-items build). Energy does not cross into a charged item by intent: it crosses through a coupling, and couplings have impedance, so a conduit's `couplingEfficiency` is one factor of `delivered = committed × coupling × competence`. ⚠ **Both factors are below 1 by construction** — the setter clamps and a nonsense value degrades to *crude*, because 1 τ ≡ 1 kJ against a conservation law and a lossless pump is a perpetual-motion machine. Deliberately NOT `potencyFactor`, whose competence term is a bonus `≥ 1`. A **tool**, not a magic item: it stores nothing, holds no working, has nothing to top up, and wears through the shipped `Durable.getCondition()`. Concrete class `obj/magic/Conduit`. See [magic-items.md](./subsystems/magic-items.md). |
| `lib/forums/` | `ForumsMixin` | the forums transmission capability (post / reply / vote / subscribe verb family), composed on a hosted update (`ForumsUpdate`). Born-with: the `ForumsUpdate` is an `AetherHosted` implant conferring this mixin, granted at intake. Acts on behalf of its host via `getHost()`. See [forums.md](./subsystems/forums.md). |
| `lib/forums/` | `SubjectSubscriberMixin` | per-Avatar forum-subscription storage: the keyed set of subscribed `Subject`s feeding the `ForumSubscriptionRegistry` fan-out. Composed by `Avatar`. See [forums.md](./subsystems/forums.md). |
| `lib/behavior/` | `BehavedMixin` | the NPC automation layer (first behavior consumer of the activity substrate): runs a declarative `behaviors:` data-spec list, path-resolving + re-resolving "brain" code modules per fire (HMR), wiring cadence (jittered, presence-gated) + `handleMessage`-witness triggers, with slot contention over `EngagedMixin`. Branch-agnostic; composed by the thin `NPC` class. See [behavior.md](./subsystems/behavior.md). |
| `lib/corpo/` | `BrandedMixin` | the per-product corpo **mark**: a `_brandKey` durable join resolving on read (via `CorpoApi`) to a `Brand` and its owning `Corpo` (or a null corpo for an independent). MQL-visible via a `subscribableFields` `brand`/`corpo` projection; appends a derived "a product of <Corpo>" `markupAugmenter` line. Composed by content onto branded objects (`BrandedBottle` = `BrandedMixin(Thing)`), not every Stuff. See [corpo.md](./subsystems/corpo.md). |
| `lib/craft/` | `GradedMixin` | the ordinal-quality carrier: a persisted `gradeBand` word + the `Grade` value-object contract (`getGrade`/`setGrade`/`getGradeBand`/`setGradeBand`). Composed by input bottles (`GradedReceptacle`) and inherited by `CraftedMixin`. See [crafting.md](./subsystems/crafting.md). |
| `lib/craft/` | `ToolMixin` | the crafting-**capabilities** layer: `capabilities: string[]` (matched by recipe-required capability). Composed as `ToolMixin(DurableMixin(…))` by `ToolItem` — the wear/condition half lives on `DurableMixin` (below), NOT here. See [crafting.md](./subsystems/crafting.md). |
| `lib/material/` | `DurableMixin` | the **wear-state axis** of a physical object (sibling of `Tangible`/`Constructed`, NOT a crafting mixin): a `condition` (0..1) that `wear()`s on **use, not the clock**. Composed by tools, weapons, and armor alike (durability ≠ "tool"); narrow via `MixinApi.isDurable`. Split out of `ToolMixin`. See [crafting.md](./subsystems/crafting.md) + [materials-response.md](./subsystems/materials-response.md). |
| `lib/material/` | `ConstructedMixin` | the **form axis** (sibling of `Tangible`'s material axis): a durable `constructionForm` word + a resolve-on-read `getConstruction()` `Construction` value-object, plus a `markupAugmenters` per-channel legibility-pips line. Composed by `Armor`/`Weapon`/`Shield` (a shield = a `Wieldable` armor-`Construction`); `MixinApi.isConstructed`. See [materials-response.md](./subsystems/materials-response.md). |
| `lib/craft/` | `CraftedMixin` | the per-instance maker's mark (composes `GradedMixin`): `{maker, recipe, craftedAt}` stamped once at craft-resolve + the DF-style `renderVerdict()` (band-word + prose + maker, never a number). Un-spoofable (maker derived from context). Composed by `CraftedDrink`. See [crafting.md](./subsystems/crafting.md). |
| `lib/craft/` | `MakerMixin` | minimal role marker (`isMaker()`) identifying an agent that can fulfill an `order` (the bartender). **Augment-gated** (`_augmentGated = true`): active only while conferred (the on-shift Position), so `MixinApi.isMaker` (routed through `isActive`) selects the on-shift bartender. Not used to gate `serve`/`mix`. Composed by `Crafter`. See [crafting.md](./subsystems/crafting.md) + [employment.md](./subsystems/employment.md). |
| `lib/employment/` | `OrganizationMixin` | the **org chart**, lifted out of `BusinessMixin`: appointing authority (a `PrincipalRef` — `entity`/`office`/`seat`/`committee`) + positions + roster + `parentOrganization` nesting. Answers *who holds position P in organization O?* identically for a ministry, a shop and a press office — a thing that does not trade no longer has to pretend to. Composed by `OrganizationEntity` and (via the base constraint below) `BusinessEntity`. `MixinApi.isOrganization`. See [employment.md](./subsystems/employment.md). |
| `lib/employment/` | `BusinessMixin` | what's left once the chart moves out: the **trading** half only — operating locations + `banksAt` + account path. ⚠ Requires `OrganizationMixin` as a **base constraint** (`TBase extends MixinConstructor<Organization & OrganizationFields>`) rather than nesting it, because a nested generic mixin silently loses the inner mixin's whole surface at the type level; the constraint makes applying it to a non-organization base a compile error. Composed by `BusinessEntity`. `MixinApi.isBusiness`. See [employment.md](./subsystems/employment.md) + [mixins.md](./subsystems/mixins.md). |
| `lib/press/` | `PublisherMixin` | an organization that **publishes** (same base constraint on `Organization`): `label`/`realm`/`visibility`/`feedPath`/`publishingPositions`. Releases are `StoredDocument`s under the publisher's own `feedPath` branch; `realm` and `visibility` **derive** from the publisher, so nobody claims a masthead by passing a flag. Composed by `OrganizationEntity`. `MixinApi.isPublisher`. See [press.md](./subsystems/press.md). |
| `lib/employment/` | `EmployedMixin` | an actor's employment relationships (sparse null-default `employments` store, `getConferredMixinNames` = the on-shift Position's `confers` — the augment conferral seam). Mutators are gated by a **participant contract**, not `ApiOnly`: `FromMixin(Mixins.Organization, …)` — the employing organization is the legitimate caller, and the gate says *who*, not merely *not you*. Composed by `Character` (actor-agnostic; NPCs are the v1 consumer). See [employment.md](./subsystems/employment.md). |
| `lib/attendant/` | `AttendantMixin` | the **storefront-attention** service-point capability (on a `Thing` fixture, the `BankMixin`-on-a-fixture precedent): owns the queue + leases **and the behavior over them** (`requestAttention`/`release`/`serveNext`/`evictIdleLeases`/`dropCustomer` are mixin methods called directly — no Api hop) + per-venue config (`discipline`/`serverPositionKeys`/`staffingPolicy`/`attendDurationMs`/`skin`). Being-attended = an `AttendanceEngagement` on the **server's** attention slot (one-at-a-time falls out of the slot; waiting is free). The lease's idle-eviction sweep is the one Api-shaped piece (`AttendantApi`/`AttendantLogic`, the `ResidencyLogic` "engine informs, object decides" shape). Composed by `AttendancePoint` (and `BankCounter`). `MixinApi.isAttendant`. See [attendant.md](./subsystems/attendant.md). |
| `lib/persistence/` | `PersistableMixin` | the self-persistence-spine **host** capability: marks a `Stuff` as a singleton persistence host (keyed by `templatePath`) whose runtime state captures into a `PersistedRecord` (`holder_snapshots`) and restores on materialize. Drives materialize-on-`postRegister` / seed-then-persist, the capture-on-`cleanupOnDestruct` backstop, the persistable `canEvict` fall-through, and the `shouldPersist()` opt-out hook (Avatar → `!isGuest`). Composed **outermost** by `Avatar` (and by authored persistable rooms / host chests). Logic in `PersistableApi`/`PersistableLogic`. See [persistence.md](./subsystems/persistence.md). |
| `lib/combat/` | `CombatantMixin` | the "I can fight" capability: the `attack` + `fight` + `intervene` command affordances, the species-declared innate-attack hook (`naturalAttackChannel` → `getNaturalAttackChannel`), a banded `look` augmenter (the combat-state line while a fight is live), and (Build 2) the **standing combat posture** the consent handshake reads — the player-side `combat.lethality`/`combat.stopCondition` settings schema **plus** the authored `standingLethality`/`standingStopCondition` fields NPCs carry (NPCs aren't `Environment`s, so they can't hold settings). Carries **no** transient fight state — poise/tempo/flags all live on the `CombatSession`, never on the `Creature`. Composed by `Character`. Cycle 2 adds the canonical `defend` verb affordance. The combat-hooks build adds the seven **participant witness terminals** (`onSessionEntered`/`onExchangeResolved`/`onPoiseBandChanged`/`onDowned`/`onDefeated`/`onDefeatedFoe`/`onCoupBegun` — `@hook` no-op terminals, `super`-composed) and demotes `naturalAttackChannel` to the legacy single-attack fallback under `Species.naturalAttacks[]`. See [combat.md](./subsystems/combat.md) + [combat-hooks.md](./subsystems/combat-hooks.md). |
| `lib/combat/` | `CombatReactiveMixin` | the **instrument-dynamics marker** (the combat-hooks build; the `HazardMixin` scan precedent): the ONE mixin the combat engine scans for (`MixinApi.isCombatReactive`) at its declared seams — `onWielded`/`onUnwielded` (fired from the `Slotted` chokepoint), the `augmentInflict` compute hook (on the single augment carrier: wielded weapon, else the bare CombatReactive attacker), `onStrikeResolved`/`onStruck`/`onParry`/`onBypassed` witnesses. All hooks are `@hook` no-op terminals (override + `super`-compose), deliberately shadowable (never `@Final`/`@Unshadowable` — a shadow over a hook is the temporary-enchantment substrate), bound by the sync/deterministic/cheap contract, with consequences queued through the `CombatHookContext` (never executed in the hook frame). First consumer: `EnergizedMixin` composes it (the stun baton / electric eel shock delivery). See [combat-hooks.md](./subsystems/combat-hooks.md). |
| `lib/party/` | `PartyMemberMixin` | "I can belong to a party": a sparse `activePartyPath` pointer (+ transient `pendingInvitePartyPath`) with `ApiOnly`-gated setters (the `Employed` precedent), the `party` verb affordance, and the read surface combat's `PartyApi` seam consumes — `sideOf`/`areAllied` plus (combat-formations) the total `formationPathOf`/`roleOf`/`isCaptain` chain. Composed by **`Avatar`** and by the hireable **`Mercenary`** (`= PartyMemberMixin(NPC)`) — never the base `Character`, so a plain NPC/beast is `solo`. See [party.md](./subsystems/party.md) + [combat-formations.md](./subsystems/combat-formations.md). |

### Mixin Composition Constraints

Where a mixin only makes sense on top of another, encode it in the
generic bound rather than in a comment. Example:

```typescript
// MobileMixin requires a Containable base — compile error otherwise.
export function MobileMixin<TBase extends MixinConstructor<Stuff & Containable>>(
  Base: TBase
) { /* ... */ }
```

When a constraint is intentionally relaxed (e.g., `CommandGiverMixin`
isn't bound to Container/Containable so loose objects can still expose
commands), leave a comment explaining why.

### Public-Shape Interfaces Colocated With Mixins

Every mixin file exports an interface with the same name as the mixin
(e.g., `Container`, `Containable`, `Sensor`, `Vocal`, `CommandGiver`)
describing the public surface the mixin adds. These interfaces are
what `MixinApi.isX()` narrows to — they MUST live next to the mixin
implementation, never in a central type barrel.

### `Mixins` Registry

`lib/mixin.ts` lists every registered mixin by name. Always use
`Mixins.X` constants instead of string literals when calling
`MixinApi.hasMixin()`:

```typescript
// CORRECT
if (MixinApi.hasMixin(ctor, Mixins.Container)) { /* ... */ }

// WRONG — string literal, breaks under refactor
if (MixinApi.hasMixin(ctor, 'ContainerMixin')) { /* ... */ }
```

For narrowing-and-calling, prefer the type predicate:

```typescript
if (MixinApi.isContainer(obj)) {
  obj.addContainable(item); // obj: Stuff & Container
}
```

See [antipatterns.md § Duck Typing with Mixins](./antipatterns.md#duck-typing-with-mixins)
for the full rule.

## File Naming Conventions

- **Mixin files**: `Propertied.ts`, `Detailed.ts`, `Visible.ts` (NO
  "Mixin" suffix in the filename). The exported function is still
  named `PropertiedMixin()`. The internal class-name marker is still
  `_mixinName = 'PropertiedMixin'`.
  Test files: `Propertied.test.ts`, `Detailed.test.ts`.

- **Class files**: match the class name. `Avatar.ts`, `Player.ts`,
  `Thing.ts`, `Location.ts`.

- **Api files**: lowercase with `.ts`. `stuff.ts`, `player.ts`,
  `mixin.ts`, `containment.ts`, `message.ts`, …

- **Command YAML views**: `perception/look.yaml`, `social/say.yaml`,
  in `mud/cmd/<category>/` (grouped into category subdirs). Loaded
  recursively by `CommandApi`.

- **Command controllers**: in `mud/obj/command/<category>/`, e.g.
  `perception/LookController.ts`, `movement/GoController.ts`.

## Member Privacy: `#` vs TypeScript Modifiers

Two privacy mechanisms with different threat-model semantics:

- **TypeScript modifiers** (`private`, `protected`, `public`) —
  compile-time only. Fields are public properties at runtime:
  reachable via bracket access, reflection, `JSON.stringify`, Proxy
  traps, subclasses that override-and-super.
- **ECMAScript hard-private** (`#name`) — runtime-enforced. Cannot be
  reached by bracket access, reflection, Proxy traps, subclasses, or
  replaced prototype methods. Lexically bound to the class body.

Convention is **layer-based**:

- **Mediator/trusted-surface code** —
  `packages/server/src/backend/` and `packages/server/src/mud/api/`
  — defaults to `#` for private members. These layers wrap and
  mediate access for everything else, and `#` ensures internal slots
  are invisible to the wrapping Proxy.
- **Domain code** — `packages/server/src/mud/lib/`,
  `packages/server/src/mud/obj/`, `packages/server/src/mud/cmd/` —
  defaults to TypeScript modifiers. Domain code carries persistent
  fields that the `Hydrator` reflects into; those fields MUST be
  public. Use `protected` for subclass extension points (e.g.
  `prepareDestroy()`-style hooks), `private` for class-internal
  helpers and caches.

**Special cases** where `#` is appropriate inside `lib/` or `obj/`:

1. A reentry guard or invariant-critical flag where a malicious
   subclass overriding a method could corrupt state.
2. An internal slot that must be deliberately shielded from the
   call-security framework (the trusted method body bypasses
   mediation when touching its own state).
3. A field whose only legitimate access is the class itself, where
   forcing tests to use a deliberate observation seam is the desired
   outcome.

Caches, helpers, and ordinary internal state do NOT qualify. When
introducing `#` in domain code, leave a one-line comment explaining
which case applies.

**Hard constraint**: persistent fields (anything declared
`{ persistent: true }` in `fieldMeta`, at any layer of the chain)
cannot be `#` — the `Hydrator` reflects into them and `#` slots are
unreachable from outside the class body.

### Field metadata is ONE static

Everything a field declares about itself lives in `static fieldMeta`,
keyed by instance field name:

```ts
static fieldMeta: FieldMeta = {
  mass:         { persistent: true, marshaller: QuantityMarshaller.pathFor('kg') },
  container:    { instruction: true, authorable: true },
  _speciesPath: { ref: 'identity', persistent: true },
  exits:        { ref: 'instance', lifetime: 'owned' },
};
```

It replaced four parallel field-keyed statics (`fieldMeta`'s persistent entries,
`fieldMeta`'s marshaller entries, `fieldMeta`'s instruction entries, `fieldMeta`'s globIdentity entries) and the
`@authorable` / `@runtimeState` TSDoc tags that a source scan used to
grep for. Collected by `MixinApi.getAllFieldMeta`, up the prototype
chain, own-property only, **merging PROPERTIES independently** —
first-declaration-wins per property, concrete class first — so a
subclass adding a `marshaller` to a field its base declared persistent
gets both.

The `ref` / `lifetime` axes are the reference model; see
[ref-shapes.md](./ref-shapes.md). Invalid combinations throw at class
registration, and `pnpm lint:field-meta` carries the whole-tree rules
registration cannot see.

**Class-level statics did NOT fold in.** `commandContributions`,
`settings`, `subscribableFields` and `markupAugmenters` are keyed by
audience / setting key / virtual projection / nothing — not by field —
so they are a different question and stay separate.

## Bands vs Numbers — presentation default, not security veil

Saxonberg surfaces many derived quantities (competence, traits, renown,
regard, standing) as **bands** rather than raw scalars — the "honesty
firewall" language in [advancement.md](./subsystems/advancement.md),
[trait.md](./subsystems/trait.md),
[crafting.md](./subsystems/crafting.md), and the bible. That pattern is
load-bearing in some places and merely cosmetic in others, and the two
have been conflated. This section draws the line so the reflex stops
getting over-applied.

**Banding does up to four distinct jobs**, with different criticality:

1. **Epistemic honesty** — a derived estimate (e.g. a BKT `theta`) does
   not actually carry the precision a float implies. The band is the
   *honest* readout; the scalar is noise. ("No quantity without a
   referent.")
2. **Diegetic realism** — a character experiences "Bob seems to like
   me," not "regard 47." Numbers are non-diegetic.
3. **Anti-legibility** — keep social standing *contestable* rather than
   a public credit score (the *Nosedive* failure). Protects the social
   fabric.
4. **Anti-gaming / Goodhart** — blur the gradient so players optimize
   play, not the metric. **The weakest job** — visible-number grinding
   is a survivable, often fun failure mode, and (see below) it never
   defended against the only actors who could truly exploit it.

**The governing question is not "band or number." It is "whose hidden
state is this?"**

- **A character's OWN derived state** (their competence, traits,
  standing, self-renown) → banding is **presentation**, not security.
  Honest and tidy, but **not critical**. There is no exploit in a player
  seeing their own competence as a number. An author who wants a
  numeric character sheet may build one; the band is just the friendly
  default, cheaply overridden, creating no caste.
- **ANOTHER agent's hidden state** (what Bob believes/feels about you,
  whether a disguise holds, how good a rival actually is) → the
  constraint is **genuinely critical**, and it is not really "banding"
  at all — it is *no mind-reading*. A player-authored "regard-o-meter"
  exposing others' exact feelings would collapse the bluffing /
  deception / mediated-belief game for the whole server. This is
  protecting *shared hidden information*, not veiling a dirty number.

So: **band-as-presentation** (own state, optional) vs
**opacity-as-mechanic** (others' hidden state, load-bearing). Use the
security justification only for the second.

**The wizard critique, conceded.** A band was never a defense against
wizards and cannot be — a wizard reads `RAW_TARGET`, evals, mints. That
is the code-trust boundary the whole call-security model already rests
on; banding neither creates nor strengthens it. Invoking "anti-gaming"
to hide a number *from wizards* is incoherent. What actually contains a
wizard-authored player-facing exploit is (a) a small vetted wizard
population, (b) the publish/review gate (law==code), and (c) norms +
**dogfooding**. The real two-tier risk is not that wizards *see*
numbers — it is wizards ceasing to *dogfood* and playing a different
game than the mortals they build for. That is cultural, not a
number-hiding problem, and no band fixes it. The tier stays healthy by
being **earnable** (the player→contributor on-ramp) and dogfooded.

**Architectural resolution — kill the "hidden truth" framing.** Wherever
possible, *do not store an authoritative scalar* — **derive the band on
read** (competence, renown, traits, standing already do). With no stored
float there is no secret being veiled: wizard and mortal run the *same*
computation and get the *same* band; the wizard sees arithmetic, not a
privileged number. A genuine store-and-hide is justified only where a
real scalar must persist **and** it is another agent's hidden state
(belief, regard, disguise). Everywhere else the bias is: **expose the
data to authors, trust the publish gate, let the band be a default, not
a wall.** When banding an *own-state* stat, do not cite anti-gaming or
security as the reason — cite honesty or diegesis, and know it is
overridable presentation.

## Code Style

From `.prettierrc.js`:

- 80 character line width
- 2 spaces for indentation
- Double quotes for strings
- Semicolons required
- Trailing commas (ES5 style)
- LF line endings

From `.eslintrc.js`:

- React import not required in JSX (React 17+)
- Unused variables warning (allow `_` prefix)
- TypeScript recommended rules

**Import style**: NEVER use `.js` extensions in import statements.
TypeScript's `NodeNext` module resolution handles extensions
automatically. Use extension-free imports for all `.ts`/`.tsx` files:

```typescript
// CORRECT
import { Stuff } from '../stuff/Stuff';
import { Location } from './Location';

// WRONG
import { Stuff } from '../stuff/Stuff.js';
import { Location } from './Location.js';
```

## "Go Through the API Layer"

Several recurring rules collapse into one principle: **never call into
internal mechanism directly when an Api method exists for the same
job**. The Api layer threads through the security gate; direct calls
bypass it.

| Don't | Do |
|---|---|
| `obj.destroy()` | `StuffApi.destruct(obj)` |
| `new SomeStuff()` | `await StuffApi.create(() => new SomeStuff())` or `await StuffApi.clone(path)` |
| `item.setContainer(c); c.addContainable(item)` | `ContainmentApi.move(item, c)` |
| `typeof obj.getContents === 'function'` | `MixinApi.isContainer(obj)` (narrow) or `MixinApi.hasMixin(ctor, Mixins.Container)` (introspect) |
| `obj.fullName ?? obj.name ?? 'something'` | `obj.getPresentation()` |
| `creature.move(loc)` (raw containment) | `creature.travel(loc, 'walk')` (locomotion) |

See [antipatterns.md](./antipatterns.md) for the full rule with examples.

## Module scope declares; lifecycles initialize

Module evaluation in `src/mud/**` may only **declare**: imports/exports,
classes, functions, interfaces/types/enums, `declare module`
augmentations, and `const`/`let` declarations (initializers included —
pure value construction is a declaration). It must not contain
**free-standing executable statements**: bare calls, registrations into
other modules' state, loops, conditionals, assignments. Enforced by
`pnpm lint:module-scope` (`scripts/check-module-scope.ts`, CI-gating —
a real TS-parser statement scan, not a regex).

Why: import-time side effects couple correctness to import order, make
HMR re-evaluation a semantic event ("what re-runs when this module
reloads?" had per-site answers), and hide initialization from every
lifecycle you can reason about. The 2026-07 antipattern sweep removed
the whole population (~130 statements) and replaced each family with a
runtime lifecycle:

| Was (module-scope statement) | Now (lifecycle) |
|---|---|
| `SchedulerApi.registerActivity(type, cls)` in every activity module | **Capture-at-start** — `SchedulerRegistry.start(engagement)` captures `engagement.constructor` into the type→class dispatch index; `reloadActivity` re-points it after a hot reload. Registration doesn't exist as a concept. |
| ~~`SecurityApi.decorateApiClass(XApi)` tail in every `api/*.ts`~~ | **Kept as a sanctioned module-scope exception.** An `*Api` class is a thin, non-HMR-able interface imported directly, so the module tail IS its registration — there is no lifecycle to route it through. The sweep briefly moved this to `ModuleApi.stamp`; it was reverted. The tail lives in `check-module-scope`'s allowlist (`EXEMPT_API_DECORATE` on `mud/api/*.ts`). The four bootstrap-special Apis (`security`/`module`/`proxy`/`execution-context`) still self-decorate not at all. |
| `DialogueEffectRegistry.register('bank-circle', …)` at module scope + a boot side-effect import | **Instance lifecycle** — `BankCounter.postRegister` registers it: a live bank fixture is exactly when the verb becomes real. |
| `Stuff._registerTopLevelBranch(Thing)` in each of the five branch files | **Identity registration via the boot lifecycle** — `Stuff.registerTopLevelBranches([...])` from `BootstrapManager.installFrameworkWiring` (boot + vitest setup), compared by class identity (loader-agnostic, unlike a module-id check). |
| Registry-class handoffs (`register<X>RegistryClass`), the security↔shadow slot, the shadow↔command bridge, the glob merge-on-arrival ripple | **`BootstrapManager.installFrameworkWiring()`** — one idempotent boot call, invoked by `BootstrapManager.run()` in production and by the vitest setup file (`src/test-setup-registries.ts`) before every suite. |
| `installOnlineHoldersProvider()` auto-install on import | Called explicitly from `AppBootstrap.run()`. |
| `engine.registerFilter(…)` ×14 (Prose), `registerConverter(…)` ×23 (Quantity), `validator.preload = …` (validators) | **Lazy first-use initializers** (memoized builder functions) / folded into the `const` initializer (`Object.assign`). |

Entry-point scripts are exempt by nature — `main()`, `AppBootstrap`,
seeders, and the vitest setup file are *where lifecycle calls live*,
not modules someone imports for their exports. The loader-injected
`ModuleApi.stamp` tail is machine-generated framework lifecycle, not
authored code. When new initialization is needed, pick the matching
lifecycle above; do not add a module-scope statement (the lint will
refuse it).

## Phase Numbering Note

If you're reading the older planning docs and wondering where Phases
5 and 6 went: they got **absorbed**, not skipped. Phase 5
(Communications) shipped as part of Phase 3 messaging plus the
say/tell controllers in `mud/cmd/` and `mud/obj/command/`. Phase 6
(Extended Object Model) shipped as `Thing.ts`, `Detailed.ts`,
`Propertied.ts`, `CartesianLocation.ts` in `lib/stuff/` and
`lib/spatial/`. Implementation status now lives in
[roadmap.md](./roadmap.md).

## Cross-References

- **Subsystem docs** (the load-bearing details):
  - [templates.md](./subsystems/templates.md) — clone pipeline,
    Hydrator, TemplateApi, folder/leaf invariant
  - [persistence.md](./subsystems/persistence.md) — the `Document` base
    (plain records, not Stuff) vs Templates→Stuff,
    PersistenceManager, around-hooks
  - [lifecycle.md](./subsystems/lifecycle.md) — create/destroy
    choreography, construction sentinel, prepareDestroy
  - [state-model.md](./subsystems/state-model.md) — what gets
    persisted, Avatar self-contained, the `Document` track for records
  - [messaging.md](./subsystems/messaging.md) — MML, Scene composer,
    sensor routing, movement-message defaults
  - [shell-environment.md](./subsystems/shell-environment.md) —
    `EnvironmentMixin` settings keyspace, schema-on-mixin, `settings`
    / `var` commands, `resolveSetting` cross-host helper
  - [call-security.md](./subsystems/call-security.md) — proxy
    interception, decorators, policies, shadows
  - [mixins.md](./subsystems/mixins.md) — class-factory mixins,
    infrastructure, authoring conventions, integration with
    persistence / commands / security
- [antipatterns.md](./antipatterns.md) — patterns to avoid with the
  correct alternative for each
- [roadmap.md](./roadmap.md) — what's left to build
- [vision.md](./vision.md) — product vision (less load-bearing
  technically)
