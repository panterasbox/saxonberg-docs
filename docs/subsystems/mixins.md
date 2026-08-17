# Mixins

Saxonberg builds the standard model out of **mixins** — TypeScript
class-factory functions that contribute fields, methods, and behavior to
any base class. A `Stuff` is rarely an instance of one class; it's an
instance of a tower of mixin layers stacked on top of `Stuff` (or a
subtree like `Idea` / `Agent` / `Vessel`).

The supporting *infrastructure* is small — a type alias, a name registry,
and one Api class with about a dozen helpers. The set of *patterns*
built on top of that infrastructure is large, because every cross-cutting
subsystem (persistence, commands, security, messaging) reaches into a
mixin's static side to discover what each mixin contributes.

This doc covers the framework: what a well-formed mixin looks like, how
mixins are composed and detected, how they plug into the rest of the
system, and the antipatterns to avoid. Specific mixins (`NamedMixin`,
`PropertiedMixin`, `ContainerMixin`, …) are documented in the subsystem
that owns their concern.

## What a Mixin Is Here

A mixin is a function:

```typescript
function NamedMixin<TBase extends MixinConstructor>(Base: TBase) {
  return class NamedMixin extends Base {
    static _mixinName = 'NamedMixin';
    static fieldMeta: FieldMeta = {
      name: { persistent: true },
      surname: { persistent: true },
    };

    name: string = '';
    surname?: string;
    /* ... */
  };
}
```

Read top-down:

- It takes a constructor, and returns a new class that **extends** that
  constructor. The returned class adds fields and methods on top of
  whatever `Base` already provided.
- The returned class is itself a constructor, so it can be the input to
  the next mixin. Mixins compose by nesting.
- Stacking is left-to-right reading inside-out:
  `MobileMixin(ContainerMixin(ContainableMixin(Stuff)))` is "Stuff,
  then Containable layered on top, then Container, then Mobile."
- TypeScript sees the result as a single anonymous class; the public
  surface is published via a separate `interface` (see "Public-shape
  interface" below).

### Why mixins, not other shapes

| Alternative | Why we don't | What mixins give |
|---|---|---|
| Multiple inheritance / traits | TypeScript / JS doesn't support it | Stack as many capabilities as needed |
| Interfaces + per-class implementations | No code reuse — every class re-implements `name`, `keywords`, etc. | One body per concern, shared across every consumer |
| Composition by has-a (`obj.named`) | Two-step access (`obj.named.fullName`); breaks duck checks; harder to type | Methods land directly on the instance, narrow with one predicate |
| Decorators | Per-method only, no field contribution, build-step quirks | Add fields **and** methods; pure JS factory functions |
| Static helpers reading data fields | Loses encapsulation; helpers proliferate | Cohesive units of behavior, each a single file |

The standard model needs many overlapping capabilities — a Door is
Visible, Detailed, Perceptible, Sealable, Containable. A creature is
all of those *plus* Sensor, Vocal, Container, Mobile, CommandGiver. No
two-axis inheritance hierarchy fits that. Mixins do.

## The Shipped Infrastructure

The entire mixin framework is two files plus a handful of conventions
applied across mixin authoring.

### `lib/mixin.ts`

```typescript
export type MixinConstructor<T = object> =
  | (new (...args: any[]) => T)
  | (abstract new (...args: any[]) => T);

export const Mixins = {
  Named: 'NamedMixin',
  Container: 'ContainerMixin',
  Containable: 'ContainableMixin',
  Mobile: 'MobileMixin',
  Sensor: 'SensorMixin',
  Vocal: 'VocalMixin',
  Soul: 'SoulMixin',         // social-cluster build: emotes on every Character
  Contacts: 'ContactsMixin', // social-cluster build: per-Avatar named lists
  /* … one entry per registered mixin … */
} as const;

export type MixinName = typeof Mixins[keyof typeof Mixins];
```

That's it. Two exports: a constructor type and a name registry. Every
mixin imports `MixinConstructor` for its generic parameter; every
mixin-introspection call site uses `Mixins.X` constants instead of
string literals.

### `api/mixin.ts` — `MixinApi`

A static utility class with three primitives and a wall of type
predicates. Decorated by `SecurityApi.decorateApiClass(MixinApi)` like
every other Api.

| Method | Returns | Purpose |
|---|---|---|
| `queryMixins(ctor)` | `MixinClass[]` | Walk the prototype chain, collect every layer marked with `_mixinName`. The primitive every other helper builds on. |
| `hasMixin(ctor, name)` | `boolean` | "Does this constructor's chain include the named mixin?" Use with `Mixins.X` constants. |
| `getMixinFields(ctor)` | `string[]` | Union of every a `persistent` entry in `fieldMeta` declared on a mixin in the chain. |
| `getAllPersistentFields(ctor)` | `string[]` | Like `getMixinFields`, but also walks **classes** in the chain (not just mixin layers). The function the persistence pipeline actually calls. |
| `is{Container,Named,Sensor,…}(obj)` | `obj is Stuff & X` | Narrowing predicate per registered mixin — the dominant runtime detector. |

The predicates are a mechanical wall — one method per `Mixins.X` entry,
each of the form:

```typescript
public static isFoo(obj: Stuff): obj is Stuff & Foo {
  return this.hasMixin(obj.constructor, Mixins.Foo);
}
```

That's all of MixinApi. The substance of "the mixin framework" lives in
the discipline that mixin authors and call-site authors agree to follow,
not in this Api.

## What Makes a Mixin Well-Formed

Five conventions. Skip any of them and other subsystems silently treat
the mixin as if it weren't there.

### 0. Docstring must say what the mixin IS and IS NOT for

Every mixin file's top-level docstring **must** include two explicit
sections:

  - **What this mixin IS for** — concrete examples of correct use.
  - **What this mixin is NOT for** — concrete examples of misuse,
    pointing at the right alternative.

This is a hard requirement, not a preference. The mixin set is the
content authoring contract — seed YAML, controller code, and future
content all decide which mixin to compose based on what the docstring
says. Without an explicit IS NOT section, authors (human and AI)
routinely smuggle generic labels into `Named`, descriptive strings
into `Detailed`, or non-tangible Stuff into `Tangible`, then the
review surfaces the mistake one MR at a time. Document the rule once
and the mistake stops recurring.

Canonical exemplar: `NamedMixin`
has a full IS / IS NOT section explaining why "Alice" is a name but
"lobby" is a short description.

### 1. Factory function form

```typescript
export function FooMixin<TBase extends MixinConstructor>(Base: TBase) {
  return class FooMixin extends Base { /* … */ };
}
```

The `class FooMixin` name inside the factory is not load-bearing —
TypeScript uses it for stack traces and the `_mixinName` marker doubles
as the canonical identity.

### 2. The `_mixinName` static marker

```typescript
return class FooMixin extends Base {
  static _mixinName = 'FooMixin';
  /* … */
};
```

`MixinApi.queryMixins` discriminates "this prototype is a mixin layer"
from "this prototype is the regular class hierarchy" by looking for an
own `_mixinName` static. Forget it and the layer is invisible — it
contributes fields/methods, but no introspection notices it, no
persistence aggregator picks up its `fieldMeta`'s persistent entries, no command
discovery picks up its `commandProvider`.

The string value MUST appear in the `Mixins` registry in
`mixin.ts`. Two sources of truth would drift; the registry is
single source.

### 3. Public-shape interface, same name, colocated

```typescript
// Named.ts
export interface Named {
  name: string;
  fullName: string;
  /* the public surface only */
}

export function NamedMixin<TBase extends MixinConstructor>(Base: TBase) {
  return class NamedMixin extends Base { /* implements Named */ };
}
```

Every mixin file exports an interface named for the *thing the mixin
makes you*, not for the mixin function. `NamedMixin` produces
something that **is** `Named`; `ContainerMixin` produces a `Container`;
`SensorMixin` produces a `Sensor`. The `MixinApi.isX` predicates
narrow to `Stuff & Foo` using these interfaces:

```typescript
if (MixinApi.isContainer(obj)) {
  obj.addContainable(item);    // obj: Stuff & Container — typechecks
}
```

The interface MUST live next to the mixin implementation. Pulling it
into a central type barrel breaks the colocation rule and makes
"what does this mixin add" harder to read.

### 4. Static contributions are opt-in surface

A mixin participates in a cross-cutting subsystem by declaring a
specific static. Three are recognized today:

| Static | Read by | What it contributes |
|---|---|---|
| `static fieldMeta: FieldMeta` | `PersistentHydrator`, `Document.toDocument`, `StudioLogic` | Everything a field declares about itself — `persistent` / `marshaller` / `instruction` / `globIdentity` / `authorable` / `runtimeState`, plus the `ref` + `lifetime` reference axes |
| `static commandProvider: CommandProviderRegistry` | `CommandGiverMixin.getAvailableCommands` | YAML command files exposed when this mixin is in scope |
| `static _mixinName: string` | `MixinApi.queryMixins` | Identity (above) |
| `static cleanupOnDestruct(stuff: Stuff): void` | `StuffApi.destruct` dispatcher | Substrate-invariant cleanup when an instance of this mixin destructs (see below) |

Subsystems extend this list by reading additional statics on the same
mixin object — the convention scales.

### `cleanupOnDestruct` — framework cleanup hook

A mixin can register substrate-invariant cleanup work as a `static
cleanupOnDestruct(stuff)`. The destruct dispatcher in
`StuffApi.#destructCore` walks the host's mixin chain via
`MixinApi.queryMixins`, filters for layers whose own static has this
method (own-property — not inherited), and invokes each with the
destructing proxy. Walk order is most-derived first / base last; per-
handler try/catch is log-and-continue.

Slot order in `#destructCore`:

```
canDestruct veto  →  user onDestruct witness  →  cleanupOnDestruct walk
                  →  ShadowApi._detachAllForHost  →  Stuff.destroy()
```

This is the load-bearing R2.4 enforcement point: a subclass `onDestruct`
override that omits `super.onDestruct()` CANNOT bypass the framework
cleanup because statics aren't on the prototype chain. The hook is
opt-in — mixins that don't hold cross-Stuff refs simply don't declare
it.

```typescript
export function FactionMembershipMixin<TBase>(Base: TBase) {
  return class extends Base {
    static _mixinName = 'FactionMembershipMixin';
    static cleanupOnDestruct(stuff: Stuff): void {
      const faction = (stuff as any).getFaction?.();
      faction?.removeMember(stuff);
    }
    // …
  };
}
```

Author guidance:
- If your mixin owns a collection of live Stuff refs OR is the held
  side of one, write a `cleanupOnDestruct`.
- If your mixin owns refs whose failure would just leak objects, put
  your cleanup in `onDestruct` — convention-chained.
- If your mixin doesn't hold cross-Stuff refs at all, don't write
  either — destruct works without ceremony.

The hook is **mixin-only by design**: a concrete leaf class doesn't
appear in `queryMixins` output (it has no `_mixinName`). Concrete-class
cleanup uses the `onDestruct` witness instead.

HMR note: the dispatcher reads `mixinCtor.cleanupOnDestruct` per
invocation, not per class-load — so a hot-reload that replaces the
static is observed immediately, with the small caveat that a class
replaced mid-destruct (between dispatcher entry and the call) is a
corner case the hot-reload subsystem owns.

See [`docs/ref-shapes.md`](../ref-shapes.md) for the cleanup rules
(R2.1–R2.4) and how `cleanupOnDestruct` fits into the substrate's
ref-cleanup contract.

### Putting it together

A canonical mixin file:

```typescript
// lib/<subsystem>/Foo.ts
import type { MixinConstructor } from '../mixin';

export interface Foo {
  /* public surface */
}

export function FooMixin<TBase extends MixinConstructor>(Base: TBase) {
  return class FooMixin extends Base {
    static _mixinName = 'FooMixin';
    static fieldMeta: FieldMeta = {
      someField: { persistent: true },
    };          // optional
    static commandProvider = { self: ['foo.yaml'], }; // optional

    someField: string = '';

    /* methods that satisfy the Foo interface */
  };
}
```

Then in `lib/mixin.ts`:

```typescript
export const Mixins = {
  /* … */
  Foo: 'FooMixin',
} as const;
```

And in `api/mixin.ts`:

```typescript
public static isFoo(obj: Stuff): obj is Stuff & Foo {
  return this.hasMixin(obj.constructor, Mixins.Foo);
}
```

The four touch-points are deliberately mechanical so that adding a
mixin is a checklist, not a design exercise.

### Marker mixins (empty public surface)

Most mixins contribute fields and methods, but a few exist purely as
identity — `SingletonMixin` is the canonical example. Composing
`SingletonMixin(Foo)` doesn't add anything callable; it sets
`_mixinName = 'SingletonMixin'` so framework code can detect the
class via `MixinApi.hasMixin(ctor, Mixins.Singleton)`. The mechanical
touch-points are unchanged — `_mixinName`, `Mixins.Singleton`,
`MixinApi.isSingleton` — but the public-shape interface is empty:

```typescript
export interface Singleton {} // marker — no public surface

export function SingletonMixin<TBase extends MixinConstructor<Stuff>>(
  Base: TBase
) {
  return class SingletonMixin extends Base {
    static _mixinName = 'SingletonMixin';
  };
}
```

This is a legitimate shape. Don't invent fields or methods to "give
the mixin substance" — the marker IS the substance. The work happens
elsewhere (in StuffApi's clone pre-flight, in this case).

Marker mixins remain rare. Add fields and methods to a mixin when
you have real surface to expose; reach for the marker shape only
when the cross-cutting subsystem lookup is the entire purpose.

## Composition

### Stacking

```typescript
const CharacterBase = CommandGiverMixin(
  MobileMixin(
    ContainerMixin(
      ContainableMixin(
        VisibleMixin(VocalMixin(SensorMixin(GenderedMixin(NamedMixin(Agent)))))
      )
    )
  )
);

export abstract class Character extends CharacterBase { /* … */ }
```

The `XxxBase` named constant is the codebase idiom for any composition
deeper than two layers. It splits "what does this class compose" from
"what does this class add" — and the constant gives stack traces a
readable name. A two-layer compose is fine inline:

```typescript
const LoginBase = HasInteractiveMixin(Idea);
export class Login extends LoginBase { /* … */ }
```

The compose order is the prototype-chain order. Inner-most is closest
to `Stuff`; outer-most is the class your code interacts with. Mixin A
wrapped over mixin B can call B's methods via plain `this` — JS
prototype dispatch resolves through the chain.

### Order matters when one mixin's body calls another

`MobileMixin` calls `setContainer` / `getContainer` from
`ContainableMixin`, so `MobileMixin` must wrap `ContainableMixin`, not
the other way around. The call chain is "outer mixin's body calls into
inner mixin's body via `this.foo()`."

When the dependency is hard, encode it in the generic bound:

```typescript
export function MobileMixin<TBase extends MixinConstructor<Stuff & Containable>>(
  Base: TBase
) { /* … */ }
```

Now `MobileMixin(SomethingNotContainable)` is a compile error, not a
runtime explosion. The pattern generalizes: any mixin that needs
methods from another mixin to be present on `Base` should declare it in
the bound, not in a comment.

When the constraint is intentionally relaxed — `CommandGiverMixin`
isn't bound to `Container` / `Containable` so loose objects (a
disembodied admin console, say) can still expose commands — leave a
short comment explaining the relaxation. Loose bounds are the
exception, not the default.

### What overrides what

JS prototype dispatch: the outermost layer that defines a method wins.
If `class Avatar extends Character` (which composes nine mixins) wants
to override `name`'s getter, it just declares one — TypeScript and JS
both happily resolve to the subclass's version.

This means **overriding a mixin method on a subclass is the
ergonomic way to specialize**. No special framework concept — it's
plain JS. The interesting case (intercepting from outside the class
hierarchy) is shadows; see [call-security.md § Shadow
Subsystem](./call-security.md#shadow-subsystem).

### Composition validation (`__validateComposition__`)

When a mixin has a hard composition rule that the TypeScript bound
can't express — `Globbable ⊥ Container`, `Globbable ⊥ Singleton`,
"every glob-identity field must also be persistent" — it can opt
into a runtime check that fires once per concrete class:

```typescript
export function GlobbableMixin<TBase extends MixinConstructor<Stuff>>(
  Base: TBase
) {
  return class GlobbableMixin extends Base {
    static _mixinName = 'GlobbableMixin';

    static __validateComposition__(ctor: AnyConstructor): void {
      if (MixinApi.hasMixin(ctor, Mixins.Container)) {
        throw new Error(`${ctor.name} composes Globbable + Container; not allowed.`);
      }
      // …
    }
  };
}
```

`MixinApi.assertComposable(ctor)` walks the prototype chain calling
every `__validateComposition__` it finds (`hasOwnProperty`-filtered,
so each level only fires for the mixin that declared it). It's
called from `StuffApi.register` the first time an instance of a
given class lands in the registry; the result is memoized in a
`WeakSet` keyed on constructor identity.

#### When a TypeScript bound suffices, prefer it

The validation hook is the runtime fallback for rules that don't fit
the bound. Specifically:

- **Use the bound** when the rule is "mixin A requires mixin B in
  the base" (`MobileMixin` over `Containable` — covered earlier in
  this doc). The error is at compile time, the diagnostic is
  precise, and refactors that break the chain fail loudly.
- **Use `__validateComposition__`** when the rule is "mixin A
  cannot co-exist with mixin B at the same level" (Globbable ⊥
  Container), or when the rule reads other static data on the class
  (`globIdentityFields ⊂ persistentFields`). The bound can't reject
  "extends both" — at the type level both are present, the conflict
  is semantic.

Don't use the runtime hook as a replacement for a documented bound;
the bound is cheaper, earlier, and more specific.

#### ⚠ Prefer the bound over NESTING, too — and this one is measured

When mixin A requires mixin B, the tempting shape is to nest:

```typescript
export function AMixin<TBase extends MixinConstructor>(Base: TBase) {
  return class AMixin extends BMixin(Base) { … };   // ⚠ don't
}
```

It works at runtime and **loses B's entire surface at the type level**.
TypeScript does not surface an anonymous mixin base's members through a
generic factory, so `class Entity extends AMixin(Concrete)` sees A's
members and the *base's*, and none of B's. Measured in isolation: a
two-level nest makes B's methods and fields invisible on the leaf class,
which then shows up as dozens of "property does not exist" errors at
every consumer and every test.

Neither escape works either. Declaration merging (`interface Entity
extends B {}`) is banned by `no-unsafe-declaration-merging` — correctly,
since it *asserts* a surface nothing verifies. And annotating the factory
`: TBase & MixinConstructor<B>` is worse than useless: extending an
intersection of construct signatures inherits **nothing at all**.

So state the requirement as a **base constraint** and compose at the
concrete class:

```typescript
export function AMixin<TBase extends MixinConstructor<B>>(Base: TBase) {
  return class AMixin extends Base implements AOwnSurface { … };
}

class Entity extends AMixin(BMixin(Concrete)) {}   // ✓ everything visible
```

Applying A to a non-B base is now a **compile error** rather than a
convention, and the leaf class carries the whole composed surface. The
one cost is that A can only `implements` its *own* half of the interface
— which is why `Business` is `interface Business extends Organization,
BusinessTrade {}` with the mixin implementing `BusinessTrade`.

`CraftedMixin extends GradedMixin(Base)` predates this and takes the
other trade-off: it accepts the type loss and narrows through
`MixinApi.isCrafted` instead. That still works; it just means the graded
surface is invisible on the concrete class.

#### HMR semantics

`HotReloadApi.reload(path)` re-evaluates a module and registers a
new class binding (same name, fresh constructor identity). The
WeakSet doesn't have the new identity, so the next first-instance
registration triggers `assertComposable` against the new chain.
Existing instances of the old class keep their old check — that's
the same "old objects keep old behavior" trade-off the rest of HMR
makes.

**Leaf reload is required for a new check to fire.** Reloading a
mixin module alone doesn't retroactively rewire the leaf classes
that import it — `class Coin extends GlobbableMixin(Idea)`
captures the mixin function that existed at class-evaluation time.
Reloading `Globbable.ts` produces a new mixin function but doesn't
touch `Coin`; since `Coin`'s identity hasn't changed, the WeakSet
hit memoizes the old validation indefinitely. The dev has to reload
the leaf module too (`Coin.ts`) — which re-evaluates the class
expression against the new mixin and produces a fresh identity.

There's no auto-cascade. Reloading `Globbable.ts` doesn't fan out
to every leaf that imports it. That's intentional: bulk
re-instantiation while a player is mid-action would shock the world.
For "rotate every class that composes Globbable," use an MQL query
to find the population and reload deliberately — `world:[mixin.GlobbableMixin]`
plus an explicit `reload` per class. Forgetting to reload leaves is
*not* a correctness bug — the old check just keeps applying and the
new constraint silently doesn't tighten. Worst case: the dev
notices a constraint isn't taking and re-reloads the leaf.

#### Current opt-ins

- `GlobbableMixin` — `⊥ Container`, `⊥ Singleton`,
  `globIdentityFields ⊂ persistentFields`.
- `PerceiverMixin` — requires `Sensor` on the chain. The TS bound is
  loose (`MixinConstructor`); the `Perceiver extends Sensor`
  interface narrows the type but doesn't enforce composition.

Everything else with a documented constraint is enforced at compile
time through the `MixinConstructor<Stuff & X>` bound (`Adornable`,
`Mobile`, `Wearable` / `Wieldable`, `Postured` / `Mountable` /
`Drivable`, `Workspace`, `Business` over `Organization`, `Publisher`
over `Organization`). When you add a new mixin, prefer the
bound; reach for `__validateComposition__` only for `⊥` (mutual-
exclusion) rules and for cross-static-data checks the bound can't
express (`globIdentityFields ⊂ persistentFields`).

Cross-reference: [hot-reload.md § Composition validation](./hot-reload.md#composition-validation).

## Detection at Runtime

Three operations, ranked by frequency.

### `MixinApi.isX(obj)` — narrowing predicate

The dominant pattern. Use whenever you have an `obj: Stuff` and want to
*both* check presence and call interface methods:

```typescript
function announce(obj: Stuff) {
  if (MixinApi.isNamed(obj)) {
    console.log(obj.fullName);   // obj: Stuff & Named, no cast
  }
  if (MixinApi.isContainer(obj)) {
    for (const child of obj.getContents()) { /* … */ }
  }
}
```

The predicate threads the mixin's public interface into TypeScript's
control-flow narrowing. No casts; refactors fail loudly.

### `MixinApi.hasMixin(ctor, Mixins.X)` — introspection

Use when you don't have an instance, or you don't need the narrowing.
Typical: walking class metadata, building a registry, deciding whether
to expose an option in the editor.

```typescript
if (MixinApi.hasMixin(SomeClass, Mixins.Visible)) {
  // SomeClass has VisibleMixin in its chain
}
```

Always pass `Mixins.X`, never a string literal — see "Antipatterns"
below.

### `MixinApi.queryMixins(ctor)` — primitive walker

When you need every mixin layer in order — almost always to read a
static off each layer (e.g., `commandProvider`). The two existing
consumers are:

- `CommandGiverMixin.getAvailableCommands` — walks every mixin on
  self, inventory, environment, peers and pulls `commandProvider` off
  each layer.
- `MixinApi.getMixinFields` — walks and pulls `fieldMeta`'s persistent entries.

```typescript
for (const mixin of MixinApi.queryMixins(self.constructor)) {
  if (mixin.commandProvider?.self) { /* … */ }
}
```

Reaching for `queryMixins` directly is rare. Prefer
`getAllPersistentFields` (already aggregates) or write a similar
aggregator if you find yourself walking the same chain in three places.

## Cross-Cutting Integration

The infrastructure is small because the *integrations* with other
subsystems do the heavy lifting. Each integration is one consumer
reading one well-known mixin static.

### Persistence — a `persistent` entry in `fieldMeta` and an `instruction` entry in `fieldMeta`

The standard hydrator walks every mixin and class in the chain,
collects every `fieldMeta`'s persistent entries / `fieldMeta`'s instruction entries declaration,
and runs a **two-phase dispatch** over the template's `data` blob
(see [templates.md § The Hydrator Contract](./templates.md#the-hydrator-contract)):

```typescript
// PersistentHydrator (sketch)
// Phase 1 — property fields: setX-method-first, bracket-assign fallback.
for (const field of MixinApi.getAllPersistentFields(ctor)) {
  if (!(field in data)) continue;
  const setter = target['set' + MixinApi.pascalCase(field)];
  if (typeof setter === 'function') await setter.call(target, data[field]);
  else target[field] = data[field];
}
// Phase 2 — instruction fields: applyX-method REQUIRED.
for (const field of MixinApi.getAllInstructionFields(ctor)) {
  if (!(field in data)) continue;
  await target['apply' + MixinApi.pascalCase(field)].call(target, data[field]);
}
```

A mixin author who adds a new persistent field declares it once on the
mixin's `fieldMeta`'s persistent entries static. Every class that composes the mixin
— present or future — round-trips the field automatically. Same for
`Document` records: `Document.getAllFields()` calls the same
aggregator, so auth records also pick up mixin contributions.

The `setX` method (or the bracket-assign fallback through an accessor
pair) fires per-field invariants during hydrate. **Instruction fields**
— declarations consumed to produce derived runtime state via an
`applyX` method (e.g., `exits` on `ExitableMixin`) — are declared
as `{ instruction: true }` in `fieldMeta`, and require the applier method;
the absence of `applyX` for a declared instruction field is a
configuration error. See
[antipatterns.md § Per-Field Invariants
](../antipatterns.md#per-field-invariants-belong-on-setters-not-in-normalize-hooks)
for the pattern.

A mixin that owns complex state (a `Set<Stuff>`, a typed graph) and
can't round-trip via plain JSON copy declines to declare the field in
`fieldMeta`'s persistent entries — see `ContainerMixin.inventory`. The composing
class either doesn't persist it or supplies a custom Hydrator subclass.

### Commands — `static commandProvider`

`CommandGiverMixin.getAvailableCommands` discovers commands by reading
`commandProvider` off every mixin in scope:

```typescript
static commandProvider = {
  self: ['inventory.yaml', 'get.yaml', 'drop.yaml'],
  environment: [],
  inventory: [],
  peers: [],
};
```

Anything composing `ContainerMixin` automatically exposes the
inventory commands when *it* is the giver. Anything composing
`MobileMixin` automatically exposes `go`/`open`/`close`. The
contributing class doesn't have to know which mixins it composed —
discovery walks the chain on every command-resolution pass.

See [command-routing.md § Discovery](./command-routing.md) for the full
self/inventory/environment/peers semantics.

### Security — `@CallSecurity`, `@Unshadowable` on mixin methods

Mixin methods can carry security decorators. The closest decorator on
the prototype chain wins:

```typescript
@Unshadowable
class PropertiedMixin { /* every method on the layer is unshadowable */ }

class FooMixin {
  @CallSecurity(SecurityPolicies.ApiOnly)
  doSomethingPrivileged() { /* … */ }
}
```

A subclass that **redeclares the method without a decorator** keeps the
mixin's policy. Override the method body and you preserve the contract
for free. To weaken or strengthen, redeclare the decorator too. Full
rule: [call-security.md § Mixin defaults + class
override](./call-security.md#mixin-defaults--class-override).

`PropertiedMixin` is class-form `@Unshadowable` because masks (its own
per-property override mechanism) are the legitimate way to intercept
property values; shadows on `getProp` would re-implement masks badly.
See [properties.md § Why @Unshadowable on the whole
class](./properties.md#why-unshadowable-on-the-whole-class).

### Shadows — composition declares type, declaration declares intercept

Shadows (see [call-security.md § Shadow
Subsystem](./call-security.md#shadow-subsystem)) interact with mixins
in a deliberately specific way. The rule has two halves.

**Composition gives the shadow type-shape, not intercept surface.**
A shadow that does `class FooShadow extends NamedMixin(Shadow) {}` is
type-compatible with `Named` — it has `name`, `surname`, `fullName`
on its own state. But composition alone does NOT enrol any of those
methods into the shadow's intercept set on the host.
`ShadowApi.attach` looks at the shadow's *own class body* — own
properties of `shadow.constructor.prototype` minus `constructor`,
plus any `@Shadowing(...)`-decorated methods. Methods inherited from
composed mixin layers are part of the shadow's type contract, not
its intercept set.

```typescript
// Intercepts NOTHING — empty surface, attach throws.
class BareShadow extends NamedMixin(Shadow) {}

// Intercepts ONLY `fullName`. NamedMixin gave the type/state; the
// override is the only own-prototype method.
class LiarShadow extends NamedMixin(Shadow) {
  override get fullName(): string { return 'Bob'; }
}

// Intercepts `say` via decorator — no mixin composition needed.
class TraceShadow extends Shadow {
  @Shadowing('say')
  trace(text: string): string {
    console.log(text);
    return this.callDown<string>(text);
  }
}
```

**Why the rule is "explicit declaration only".** The earlier rule
auto-enroled every method a composed mixin contributed. That
collapses badly with Witness mixins (the `canX` / `onX` optional
hooks `ContainmentApi.move` and friends fire). A shadow that
composed `Containable` purely to react to `onMoved` would
auto-intercept `setContainer` / `getContainer` with the mixin's
defaults, masking the host's real behavior. Explicit declaration
sidesteps the trap: mixins on a shadow give you the *type* without
silently hijacking the host.

#### Can a shadow add a mixin's surface to a host that lacks it?

Tempting idea: attach a `class CrownShadow extends NamedMixin(Shadow)
{ … }` to a `Location` that has no `NamedMixin`, and have the
location appear `Named` for the duration. **It does not work
today** — and the limitation is the proxy's get-trap, not the shadow
chain.

Trace `host.fullName` on a host with no `NamedMixin`, with a shadow
`@Shadowing('fullName')`-decorated:

1. `ProxyApi.wrap`'s get-trap looks up the descriptor by walking
   the **target's** prototype chain. The host has no `fullName`
   anywhere on its chain, so descriptor is `undefined`.
2. The getter branch (`if (descriptor?.get)`) is skipped.
3. `Reflect.get(target, 'fullName', receiver)` returns `undefined`.
4. `typeof undefined !== 'function'` is true, so the trap returns
   `undefined` directly without running the interceptor pipeline.
5. The security gate never fires; the shadow chain is never asked.

The same applies to non-getter methods: missing methods short-circuit
to `undefined` before any shadow lookup happens.
`ShadowApi.attach` does not check that the host has the methods
the shadow intends to intercept, so attach succeeds — but reads of
those methods on the proxied host return `undefined`.

Two consequences worth knowing:

- **Shadows can intercept what the host already exposes.** Override
  behavior, wrap with logging, swap returns. The host's mixin chain
  is the surface available; the shadow chain layers on top.
- **Shadows cannot add new surface to a host.** Capability addition
  via the proxy doesn't flow through shadows in the current
  implementation. If a feature really needs runtime capability
  injection, it would need a different mechanism — e.g., a proxy-
  level fallback that consults shadows on missing methods, or a
  surface-extension pattern bolted onto `MixinApi`. Neither exists
  today; flag it before designing against the assumption.

#### Mixin defaults vs `@Unshadowable` mixins

A mixin can declare itself class-form `@Unshadowable`
(`PropertiedMixin` is the canonical example). Every method the
mixin contributes is then unshadowable on every class that composes
it — `ShadowApi.attach` rejects shadows that intercept any of those
methods. This is how mixins protect their internal contract while
still offering an extension mechanism (masks, in
`PropertiedMixin`'s case). Most mixins are not `@Unshadowable`; a
shadow that overrides `fullName` from `NamedMixin` is supported.

### Lifecycle — `PostRegistrationMixin`

`StuffApi.create` / `clone` checks `MixinApi.isPostRegistration(obj)`
and `await`s `obj.postRegister(context)` after registration. Mixin-
gated lifecycle hooks are how the framework adds extension points
without bolting them onto `Stuff` itself — the hook is opt-in by
composition.

### Messaging — `SensorMixin` / `VocalMixin`

Same pattern: `MixinApi.isSensor(obj)` is the gate that decides whether
a frame is delivered to that obj's `handleMessage`. Anything composing
`SensorMixin` is a participant; anything not is invisible to the
delivery routing. See [messaging.md](./messaging.md).

The throughline: **a cross-cutting subsystem reaches into the mixin
chain to discover what each mixin contributes**, and the framework
infrastructure is tiny because the discovery primitives are tiny.

## File Layout

### Subsystem-folder placement

Mixins live in the `lib/<subsystem>/` folder that owns the concern they
model. **There is no `lib/mixins/` folder** — that grouping is
explicitly prohibited.

| Folder | Mixins |
|---|---|
| `lib/character/` | `GenderedMixin` |
| `lib/description/` | `NamedMixin`, `VisibleMixin`, `PerceptibleMixin`, `DetailedMixin` |
| `lib/spatial/` | `ContainerMixin`, `ContainableMixin`, `MobileMixin`, `ExitableMixin`, `SealableMixin`, `CartesianCoordinatesMixin`, `SphericalCoordinatesMixin` |
| `lib/message/` | `SensorMixin`, `VocalMixin` |
| `lib/command/` | `CommandGiverMixin` |
| `lib/connection/` | `HasInteractiveMixin` |
| `lib/persistence/` | `AroundSaveHookMixin`, `AroundDeleteHookMixin` |
| `lib/stuff/` | `PropertiedMixin`, `PostRegistrationMixin` |

"Mixin" is an implementation technique, not a subsystem — a `lib/mixins/`
folder would group by *form* rather than by what the code is *for*. If
a new mixin doesn't fit any existing subsystem folder, propose a new
subsystem folder for it (and write a doc).

### File naming

- **Mixin file**: `Foo.ts` — *no `Mixin` suffix in the filename*. The
  exported function is still `FooMixin()`; the static marker is still
  `_mixinName = 'FooMixin'`. The filename matches the public-shape
  interface.
- **Test file**: `Foo.test.ts`, in a colocated `__tests__/` folder.
- **Class files**: `Avatar.ts`, `Login.ts` — match the class name.

## Antipatterns

### Duck typing instead of `MixinApi.isX`

```typescript
// WRONG
if (typeof obj.getContents === 'function') {
  obj.getContents();
}

// RIGHT
if (MixinApi.isContainer(obj)) {
  obj.getContents();
}
```

Duck typing has no compile-time check, doesn't communicate which mixin
provides the surface, and breaks silently on rename. Full rationale:
[antipatterns.md § Duck Typing with
Mixins](../antipatterns.md#duck-typing-with-mixins).

### String mixin names instead of `Mixins.X`

```typescript
// WRONG — string literal, breaks under refactor
if (MixinApi.hasMixin(ctor, 'ContainerMixin')) { /* … */ }

// RIGHT
if (MixinApi.hasMixin(ctor, Mixins.Container)) { /* … */ }
```

The registry is the single source of truth. A typo against a string
literal compiles and silently always returns `false`.

### Forgetting `_mixinName`

A returned class without `_mixinName` is invisible to `queryMixins`,
which means: no persistent fields aggregated, no commands discovered,
no `MixinApi.isX` matches. The mixin's fields and methods *do* land on
instances (JS doesn't care), so the failure mode is "everything looks
right, but the persistence track / command surface / introspection
quietly drops it." Easy to miss in code review. The compose-and-test
pattern (see test fixtures) catches it.

### Forgetting to add to the `Mixins` registry

The marker string and the registry constant must agree. If
`_mixinName = 'FooMixin'` and `Mixins.Foo` doesn't exist, `MixinApi`
calls become `string`-typed instead of `MixinName`-typed and the
`isFoo` predicate is missing.

### Forgetting `MixinApi.isFoo`

The wall of predicates in `MixinApi` is one entry per registered mixin.
A new mixin needs a new predicate. Without it, callers fall back to
`hasMixin` + cast, which loses the narrowing. The repetition is the
price of keeping `MixinApi` mechanical.

### Putting the public-shape interface in a barrel

```typescript
// WRONG — interface in a central types file, far from impl
// types/mixin-shapes.ts
export interface Container { /* … */ }

// lib/spatial/Container.ts
import type { Container } from '../../types/mixin-shapes';
```

Colocate. The interface is the documented surface of the mixin; reading
it should not require finding a separate file.

### Using `lib/mixins/` as a folder

Hard rule. Mixin is form, not concern. Locate by what the mixin
*models* — `description`, `spatial`, `message`, `command`. If nothing
fits, add a subsystem folder.

### Mixin-vs-Shadow confusion

Mixins are static structure: every class composing the mixin gets
the surface. Shadows are per-instance behavior overrides. They aren't
substitutes for each other — and a few footguns sit at the seam.

- **Reaching for a custom subclass to add behavior to one specific
  instance** — buffs, tracers, polymorph effects, runtime renames.
  That's what shadows are for. Subclassing per-instance multiplies
  classes for no gain.
- **Reaching for a shadow to add a brand-new capability to a host
  that doesn't have it.** Doesn't work today — see
  [§ Shadows — composition declares type](#shadows--composition-declares-type-declaration-declares-intercept)
  for the proxy short-circuit that prevents it.
- **Composing a mixin on a shadow expecting it to intercept the
  host's same-named methods.** Composition gives the shadow the
  *type contract*; only methods the shadow declares in its own body
  (or `@Shadowing`-decorates) intercept. Same section.

## Design Rationale

### Why `_mixinName` instead of `instanceof`?

`instanceof` doesn't work cleanly with anonymous classes returned from
factory functions. The closure captures `Base`, the returned class is
distinct per call, and there's no stable constructor to compare
against. `_mixinName` gives a string-keyed identity that survives every
re-application of the factory.

### Why a name registry?

Stringly-typed mixin checks are typo-prone. The registry constants
turn `Mixins.Container` into a discoverable, refactorable, autocompleted
identifier. The string value of `Mixins.Container` (i.e.
`'ContainerMixin'`) is an implementation detail that `_mixinName`
happens to share — callers shouldn't depend on the literal.

### Why interfaces published next to mixins?

Two reasons:

1. **Discovery.** Reading `Container.ts` shows the contract and the
   implementation in one file. No file-jumping.
2. **Refactor safety.** Renaming the mixin without renaming the
   interface (or vice versa) is a single-file change reviewers can see
   at a glance.

### Why is `MixinApi` so mechanical?

The wall of predicates is intentional. Each mixin gets:

- An entry in `Mixins`.
- A predicate `isX` on `MixinApi`.
- A public-shape interface in its own file.

The repetition is the cost of giving every mixin a first-class,
narrowing predicate. The win — refactor-safe, autocompleted, type-
narrowed mixin checks across the codebase — is worth it.

### Why class factories instead of TC39 decorators?

TC39 decorators target methods and classes, not field-shape
contributions. A mixin contributes fields, methods, getters/setters,
and statics; decorators don't naturally do all four. Factories also
sidestep the "decorator stage 3 vs experimental" build-config wrinkles
that bite in TS-strict mode. The codebase *does* use decorators
elsewhere (`@CallSecurity`, `@Unshadowable`, `@Final`); the security
framework's surface is method-centric and decorators fit well there.
For the standard model's "compose me a class with these capabilities"
shape, factories are the better fit.

### Why no `lib/mixins/` folder?

A mixin grouping by form is a grouping by **how it's implemented**,
not by **what it's for**. The codebase favors locating by concern: a
spatial mixin lives next to spatial classes; a description mixin lives
next to description classes. The difference shows up at refactor time
— "spatial" is a real subsystem you can talk about; "mixins" is just
a list of unrelated capabilities that happen to share an
implementation strategy.

### Why are some statics aggregated and not others?

Two of the three known statics aggregate across the chain:

- `fieldMeta`'s persistent entries aggregates because hydration touches every layer
  that owns state.
- `commandProvider` aggregates because every layer in scope can
  contribute commands.
- `_mixinName` is per-layer (it identifies a single layer; aggregating
  it doesn't make sense).

A future mixin static — say, `eventListeners` — would decide its
aggregation strategy based on whether multiple layers can sensibly
contribute or whether one wins.

## Cross-References

- [architecture.md § Mixin
  Organization](../architecture.md#mixin-organization) — the headline
  rules (subsystem folder, no `lib/mixins/`, `Mixins` registry).
- [collections.md](./collections.md) — canonical surfaces for mixins
  that own a collection: Shape A (Set membership), Shape B (keyed
  Map), Shape C (ordered list), Shape D (property bag).
- [antipatterns.md § Duck Typing with
  Mixins](../antipatterns.md#duck-typing-with-mixins) — the `isX` /
  `hasMixin` rule and migration recipes.
- [persistence.md § Field
  Aggregation](./persistence.md#field-aggregation) — how
  `fieldMeta`'s persistent entries rides the chain through `Document` and the
  Hydrator.
- [templates.md](./templates.md) — `PersistentHydrator` and how the
  clone pipeline calls `getAllPersistentFields`.
- [command-routing.md](./command-routing.md) — `commandProvider` discovery walks
  every mixin in self/inventory/environment/peers.
- [call-security.md § Mixin defaults + class
  override](./call-security.md#mixin-defaults--class-override) — how
  `@CallSecurity` / `@Unshadowable` on mixin methods are inherited and
  overridden.
- [call-security.md § Shadow
  Subsystem](./call-security.md#shadow-subsystem) — full mechanics for
  attach/dispatch/callDown/callBypass; the explicit-declaration rule
  for shadow intercept sets and the Witness-mixin rationale that
  drove it.
- [properties.md](./properties.md) — `PropertiedMixin` as a worked
  example of a class-form `@Unshadowable` mixin with its own per-
  property override mechanism.
- [lifecycle.md](./lifecycle.md) — `PostRegistrationMixin` and the
  framework's mixin-gated lifecycle hooks.
- [messaging.md](./messaging.md) — `SensorMixin` / `VocalMixin` as the
  scope predicates for message routing.
- [events.md § Witness pattern](./events.md#witness-pattern) — optional
  `canX` / `onX` hooks declared on existing mixin interfaces; the
  rationale for why composing-a-mixin-on-a-shadow no longer
  auto-enrols every method.
