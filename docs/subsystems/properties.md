# Properties

`PropertiedMixin` is the controlled API for **dynamic per-instance
state** — values that aren't structural fields on the class but need
to live on a particular runtime object. Stat sheets, equipped buffs,
quest flags, capability tokens, configuration overrides, ad-hoc data
attached at runtime — all go through `setProp` / `getProp`, not
through `obj.foo = bar`.

The shape lives in `lib/stuff/Propertied.ts`. The mixin is
**class-form `@Unshadowable`**: every method on a Propertied layer is
protected from shadow attachment. The legitimate per-property
extension surface is `maskProp`, not a shadow.

## Core Principle

**Game objects do NOT add arbitrary properties via direct
assignment.** Every dynamic value goes through the Propertied API.

```typescript
// WRONG — bypasses access control, persistence opt-in, introspection,
// and is unreachable to anything that walks a Stuff's properties
avatar.questStarted = true;
avatar.tally = 100;

// RIGHT — controlled, introspectable, opt-in persistence, gateable
avatar.setProp(Property.of<boolean>('quest_started'), true);
avatar.setProp(Property.of<number>('tally'), 100);
```

## ⚠ What props are FOR — and the two things they are not

There are exactly **two** production prop call sites, and they share a
shape:

| Site | Key |
|---|---|
| `api/event.ts` — the EventRegistry | `Property.of(eventName)` — one slot per authored event |
| `obj/api/BankingLogic.ts` — corpo circle membership | `circleProp(corpoKey)` — one flag per corpo |

Neither key exists at authoring time, and that is the whole point:

> ⭐ **A mixin field needs a statically known name. A prop is for a slot
> whose key is computed at runtime.**

A sharper test than "is it dynamic", and it explains why the count is
two rather than two hundred. This codebase is *designed* — one mind,
everything named up front — so the organic, unforeseen state props exist
to absorb has mostly not arisen. **That is the expected state, not an
under-use to correct.**

### ⚠⚠ Money is not a prop

This doc used to run on `gold` as its example. That predates the banking
build and is now **the exact breach the money system exists to
prevent**: money lives in `bank_ledger`, written only by the sealed
`postTransaction` chokepoint, which enforces
`supply == Σ account balances + Σ circulating coins`. A number on an
avatar — prop *or* field — mints currency from nowhere. See
[banking.md](./banking.md).

### And a "data-only" mixin is still a mixin

Props do not replace a thin mixin, because three framework contracts key
on `fieldMeta` and have no prop equivalent:

1. **`authorable`** — the Hydrator reflects into **fields**, never
   props, so a prop cannot be written in a template's `data:` block or
   appear in the Studio `@authorable` schema. **Props are unreachable
   from content.**
2. **narrowing** — `MixinApi.isX(obj)` threads a mixin's interface into
   TypeScript's control flow; `getProp` returns `T | undefined` on every
   Stuff.
3. **`globIdentity`** — stack-merge identity is declared in `fieldMeta`.

`LabelledMixin` is three accessors and no logic, yet it carries a setter
invariant the Hydrator routes through, contributes the `label` verb, is
narrowed on by `RecognitionApi`, vetoes stack merges, and is authorable.
**A mixin is a TYPE; a prop is a VALUE.**

Direct assignment is invisible to the framework. Props are visible to
access control, mask transformation, persistence, and enumeration —
the four things that make per-instance state work cleanly across the
codebase. See [antipatterns.md § Use props, not direct field
assignment](../antipatterns.md#use-props-not-direct-field-assignment).

## Concepts

### `Property<T>` — typed key wrapper

Property keys are nominal strings carrying a type parameter:

```typescript
const tally = Property.of<number>('tally');
const flags = Property.of<string[]>('quest_flags');
```

`Property<T>` extends `String`, so a `Property<T>` is usable wherever
a string key is. The type parameter threads through `setProp` /
`getProp` so writes and reads agree on the value type:

```typescript
avatar.setProp(tally, 100);          // ok
avatar.setProp(tally, 'rich');       // type error: 'rich' is not number
const t: number | null = avatar.getProp(tally);
```

### Transient vs saved

Every property is one or the other:

- **Transient** (default) — lives in `transientProps`, never persisted.
  Use for run-time-only state: cached values, current target, in-flight
  command context, frame-counters.
- **Saved** — lives in `savedProps`, which is declared a persistent
  field on `PropertiedMixin` so the standard hydrator round-trips it.

```typescript
// Transient (default — auto-init on first set)
avatar.setProp(Property.of<string>('current_target'), 'goblin-42');

// Saved — explicit init
avatar.initProp(Property.of<number>('tally'), { transient: false });
avatar.setProp(Property.of<number>('tally'), 100);
```

The distinction is settable later via `configureProp`; the value moves
between storages atomically.

### Access control vs masks

These are separate axes. Don't conflate them.

| | **Access Control** | **Masks** |
|---|---|---|
| Question | *Who* may operate on this property? | *What value* is returned by getProp? |
| Form | Predicate `(prop, op, special) => boolean` | Transformer `(prop, value, ...extra) => value` |
| Set when | At `initProp` / `configureProp` | Via `maskProp` / `unmaskProp` |
| Per-property | One `checkAccess` total | Many masks, keyed by owner |
| Operations | `Set` / `Get` / `Configure` / `Remove` / `Mask` / `Unmask` | Implicitly `Get` only |
| Effect on `setProp` | Can deny | Ignored (writes hit raw storage) |

A property can have access control without masks, masks without access
control, both, or neither.

## API

```typescript
interface Propertied {
  getProps(): Readonly<Record<string, PropValue>>;

  initProp<T>(prop: Property<T>, options?: Partial<PropOptions<T>>): boolean;
  configureProp<T>(prop: Property<T>, options: Partial<PropOptions<T>>): boolean;
  setProp<T>(prop: Property<T>, value: T): boolean;
  getProp<T>(prop: Property<T>): T | null;
  removeProp<T>(prop: Property<T>): boolean;

  maskProp<T>(prop: Property<T>, mask: PropValueMask<T>, owner?: Stuff, ...extra: unknown[]): boolean;
  unmaskProp<T>(prop: Property<T>, owner?: Stuff): boolean;
  isMaskingProp<T>(prop: Property<T>, owner?: Stuff): boolean;

  checkProp<T>(prop: Property<T>): PropOptions<T> | null;
  getAllPropNames(): Property<PropValue>[];
  generateUniquePropName<T>(seed?: string): Property<T>;

  defaultPropAccess(prop: Property<PropValue>, op: PropOperation, special: unknown): boolean;
}
```

### `setProp` / `getProp`

Auto-initializes on first `setProp`. The auto-init makes the simple
case ergonomic:

```typescript
avatar.setProp(Property.of('quest_started'), true);
avatar.getProp(Property.of('quest_started'));   // true
```

Defaults: transient, `defaultPropAccess` (allow-all). `getProp` returns
`null` for non-existent props or denied access. `setProp` returns
`false` on denial; `true` on success. `getProp` applies any active
masks to the stored value before returning.

Auto-init means an attacker model can't matter: a property created by
unauthorized first-write doesn't gain protection by being created.
Properties that need protection must be `initProp`'d explicitly first
with the appropriate `checkAccess`.

### `initProp` and `configureProp`

`initProp` introduces the property with options; `configureProp`
changes them. Returns `false` if the property already exists (init) or
doesn't exist (configure):

```typescript
avatar.initProp(Property.of<number>('tally'), {
  transient: false,
  checkAccess: (prop, op, special) => { ... },
});

// Later, lock writes:
avatar.configureProp(Property.of<number>('tally'), {
  checkAccess: (prop, op) => op === PropOperations.Get,
});
```

Configure passes the new options bag to `checkAccess` as `special` for
the `Configure` op — so an `initProp` whose `checkAccess` denies
`Configure` can pin the property's options for the rest of the
object's lifetime (see "Configuration-frozen" pattern below).

Toggling `transient` migrates the existing value between storages
atomically.

### `removeProp`

Removes the property entirely (config + storage + masks). Subject to
the `Remove` access op.

### `checkProp` / `getAllPropNames` / `generateUniquePropName`

Introspection. `checkProp` returns the property's options when the
prop exists AND the caller is allowed to see it; otherwise `null`.
The Get access op gates visibility, so a denied caller gets the same
`null` as for a non-existent prop — one answer to "does this prop
exist and can I read it?" `EventApi.on` relies on this Get-gating to
distinguish "no such event / not allowed to subscribe" from
"declared and emitted-but-no-payload-yet" without a sentinel value.
`getAllPropNames` walks every initialized property — transient +
saved. `generateUniquePropName` mints a unique `<seed>.<nanoid>`
key, useful when multiple anonymous slots need to coexist (per-buff
state, per-effect counters):

```typescript
const slot = avatar.generateUniquePropName<BuffState>('buff');
avatar.setProp(slot, { strength: 5, duration: 60 });
// later:
avatar.removeProp(slot);
```

### `defaultPropAccess`

The catch-all when `initProp` / auto-init doesn't supply an access
predicate. Allow-all by default; subclasses can override to install
an object-wide policy:

```typescript
class GuardedAvatar extends PropertiedMixin(Stuff) {
  override defaultPropAccess(_prop, op, _special) {
    // Default: anyone can read, only owner can write
    if (op === PropOperations.Get) return true;
    return ExecutionContextApi.getCaller() === this;
  }
}
```

## Storage and Persistence

`savedProps: Record<string, PropValue>` and its companion
`savedPropMarshallers: Record<string, string>` (per-prop name →
marshaller templatePath) are declared in
`PropertiedMixin.persistentFields`, so any class composing
`PropertiedMixin` automatically round-trips its saved props through
the persistent hydrator. No subclass changes required. Pass
`marshaller:` to `initProp` for props holding rich values
(quantities, etc.); the binding persists in `savedPropMarshallers` so
`getProp` re-applies it on reload.

```typescript
class Avatar extends PropertiedMixin(SomeBase) {
  // savedProps is auto-persisted
  static fieldMeta: FieldMeta = {
    name: { persistent: true },
    location: { persistent: true },
  };
}
```

`transientProps` is NOT persistent — by name, the field isn't on the
declared list. Reload reconstructs an empty `transientProps` and an
empty `propOptions` map. **Property options (transient flag, access
control function) do NOT persist.** Re-installing access predicates
after load is the responsibility of the class — typically in a
post-hydrate hook or `postRegister` — because access functions are
closures and aren't representable in MongoDB.

`getProps()` returns a read-only view combining both storages
with transient taking precedence. Useful for enumerating, snapshotting,
or rendering — never for write-back.

## Access Control Patterns

Pattern recognition is the point of this section. When a feature needs
"per-instance state that not everyone should be able to read or write,"
reach for a property with `checkAccess`.

### Capability gate

> "This object can do X if and only if the framework's policy says so."

A capability is a property whose access predicate is the
authority-check itself. Setting the property is what installs the
capability; reading it is the actual permission check at use-site.

```typescript
const canBroadcast = Property.of<true>('caps.broadcast_global');

obj.initProp(canBroadcast, {
  transient: true,
  checkAccess: (_prop, op, _special) => {
    // Only the trusted-grantor module may set this capability.
    if (op === PropOperations.Set || op === PropOperations.Remove) {
      return ExecutionContextApi.getCaller() instanceof CapabilityRegistry;
    }
    return true; // Anyone may read/test.
  },
});

// Use site (somewhere far away):
if (obj.getProp(canBroadcast) === true) {
  // Authorized — proceed.
}
```

Two things make this work: `getProp` is cheap and deterministic, and
`checkAccess` runs in the call-stack context of the operation — so
"who is calling" is grounded in `ExecutionContextApi`, not a
ceremonious capability-token shuttling pattern.

The event-broadcast example above is illustrative — the actual rules
for which Stuff may broadcast which events are owned by the eventing
subsystem; this doc just shows the shape.

### Ownership-gated mutation

> "Anyone can read this; only the owning Stuff can write it."

```typescript
avatar.initProp(Property.of<number>('tally'), {
  transient: false,
  checkAccess: (_prop, op, _special) =>
    op !== PropOperations.Set || ExecutionContextApi.getCaller() === avatar,
});
```

The mutation rule lives at the property, not at every call site that
touches it. Banking systems, currencies, and per-actor counters are
natural fits.

### Read-only-from-outside, writable-from-self

> "The object computes and stores its own derived state; outsiders see
> the result but can't poke it."

```typescript
location.initProp(Property.of<number>('occupancy'), {
  transient: true,
  checkAccess: (_prop, op) => op !== PropOperations.Set
    || ExecutionContextApi.getCaller() === location,
});
```

The read predicate stays open; the write predicate locks down.
Composable with masks for derived values that depend on other state.

### Configuration-frozen-after-init

> "Once configured, the property's options can't be changed."

```typescript
obj.initProp(critical, {
  transient: false,
  checkAccess: (_prop, op) => op !== PropOperations.Configure,
});
```

After `initProp`, `configureProp` denies the `Configure` op. Useful
for security-critical props (capabilities, audit flags) that should
never relax — once a Configure check is denied at init, any later
attempt to relax it via `configureProp` also fails.

### Operation-specific gates

`PropOperations` distinguishes six operations: `Configure`, `Set`,
`Get`, `Remove`, `Mask`, `Unmask`. The `special` parameter carries
operation-specific extra context:

| Op | `special` is |
|---|---|
| `Set` | the incoming value (T) |
| `Configure` | the partial options being applied |
| `Mask` / `Unmask` | the owning Stuff |
| `Get` / `Remove` | `null` |

So a predicate can do:

```typescript
checkAccess: (_prop, op, special) => {
  if (op === PropOperations.Set && (special as number) > 1_000_000) {
    return false;  // Reject implausibly large writes.
  }
  if (op === PropOperations.Mask) {
    return isTrustedSource(special as Stuff);  // Only some sources may mask.
  }
  return true;
}
```

## Masks — properties' answer to shadows

`PropertiedMixin` is `@Unshadowable`: you cannot install a Shadow that
intercepts `getProp` to alter the returned value. The reason a Shadow
might want to is exactly the reason masks exist:

> Per-instance, owner-keyed, stackable transformation of a value
> without rewriting the producer.

That's masks. A mask is a value transformer attached to a single
property, owned by a single in-world Stuff:

```typescript
class StrengthRing extends Stuff {
  activate(target: Avatar) {
    target.maskProp(strength, (_p, v) => v + 5);
  }
  deactivate(target: Avatar) {
    target.unmaskProp(strength);
  }
}
```

Masks are applied in registration order during `getProp`. Multiple
masks compose:

```
base 10 → ring (+5) → potion (×2) → curse (-3) → final 27
```

`setProp` writes the **base** value; masks transform what `getProp`
returns. To mutate the base, set it directly and let the masks
re-apply on the next read.

### Owner identity

The `owner` parameter to `maskProp` is the in-world Stuff this mask
"belongs to" — the ring, the potion, the curse, the buff. Three uses:

1. **Removal.** `unmaskProp(prop, owner)` removes only that owner's
   masks, leaving everyone else's intact.
2. **Idempotency.** A given owner can hold at most one mask per
   property. A second `maskProp` from the same owner on the same prop
   returns `false`. (Compose multiple effects via the `extra` args, or
   create a new owner Stuff per effect.)
3. **Access control.** `owner` is passed to `checkAccess` as `special`
   for the `Mask` / `Unmask` operations.

### Owner defaults to the calling Stuff

`owner` is optional. When omitted, it resolves to the **nearest Stuff
on the call stack** — walks `ExecutionContextApi.getCallStack()`
top-down looking for the most recent frame whose `caller` is a `Stuff`.
For the common case — Stuff A reaches into Stuff B and adds a mask —
the explicit owner argument is noise:

```typescript
class StrengthRing extends Stuff {
  activate(target: Avatar) {
    target.maskProp(strength, (_p, v) => v + 5);   // owner = this ring
  }
}
```

If no Stuff is reachable on the stack (raw test code, framework
bootstrap), the call **throws**. Misattributing ownership silently
would be worse than the loud error.

If you genuinely want a third party to be the owner — e.g., the ring
is the *medium* but the spell that enchanted the ring is the *owner*
— pass it explicitly: `target.maskProp(strength, fn, theSpell)`.

### Auto-cleanup of failed masks

A mask whose owner has been destroyed will throw when invoked (its
captured state is gone). `getProp` catches that, drops the offending
mask, and continues with the rest. This is the simplest way to handle
buff expiration: destroy the buff Stuff, and the next `getProp` cleans
up. The buff doesn't need to remember everywhere it's been attached.

### Why masks aren't shadows

Shadows are designed to intercept *methods*. A Stuff carries hundreds
of properties; the value of `obj.getProp(prop)` depends on *which*
prop. Routing every prop's transformation through a single
`getProp`-shadow chain that re-dispatches by name would re-implement
masks badly, and would force `getProp` to be shadowable — which
exposes a hot path covering every property to attack.

Masks live one level deeper: they intercept *the value of one
property*, not the method. The mixin stays unshadowable, the property
keeps its identity, and per-property transformations are exactly as
expressive as shadow chains.

## Lifecycle Notes

- **Auto-init at first `setProp`** is the ergonomic default but means
  *protection requires explicit `initProp` first*. A property that
  needs `checkAccess` MUST be `initProp`'d before any `setProp` reaches
  it; otherwise the auto-init installs allow-all and writes win.
- **`propOptions` and `propMasks` are runtime-only.** Reload starts
  fresh. Re-install access predicates and masks in a post-hydrate
  hook if they need to survive restart. Persisting closures isn't
  practical; classes that need declarative access rules typically
  reconstruct them from a per-class table at load time.
- **`@Unshadowable` is class-form** — every method on a Propertied
  layer is protected. If a real need to shadow surfaces (admin debug
  tracing of all property access, perhaps), downgrade specific methods
  to "shadowable but only by Admin" rather than removing the class
  marker.

## Design Rationale

### Why not just use class fields?

Class fields are static structure. A new field requires editing the
class. Props handle the cases where:

- The set of useful keys depends on what the player is doing (active
  buffs, current capabilities, in-flight quest state).
- The keys are author-defined at runtime (mod content, custom NPCs).
- The protection rules differ per-key (a corpo-membership flag needs
  ownership-gating; a script's scratch counter doesn't).
- The values are best treated uniformly (enumeration, save/load,
  introspection).

Class fields cover the structural shape of a Stuff. Props cover the
dynamic, instance-specific, context-dependent shape on top.

### Why not a Map plus a getter?

The same observation as masks-vs-shadows: a Map exposes raw read /
write to anyone with a reference, with no per-key protection, no value
transformation, no auto-persistence opt-in. The mixin is the four
crosscutting concerns expressed once instead of per-call-site.

### Why two storage maps instead of a flag?

`savedProps` is the persistent field; `transientProps` is the runtime
sibling. The two-map shape lets the persistence pipeline serialize a
single, well-defined object without filtering, and lets a reload
simply re-attach `savedProps` in place. A single map with a flag would
require filter passes on every save/load and would couple the
persistence cost to total prop count rather than saved prop count.

### Why named operations instead of "read" / "write"?

Operations are six because the threats differ:

- `Set` and `Configure` both modify state, but `Configure` modifies
  the *protection rules* — a denial there is what enables the
  Configuration-frozen pattern.
- `Mask` and `Unmask` are gates on third-party transformations of the
  value, distinct from writing the base.
- `Remove` is structural deletion; the protection rules around it are
  often stricter than for `Set` (you can usually overwrite a flag;
  rarely should you be able to make it cease to exist).
- `Get` is read.

Six is the smallest set that captures meaningfully-different policy
decisions seen in practice.

### Why `@Unshadowable` on the whole class?

`getProp` is hot-path everywhere; `setProp` carries field-shape
invariants; shadowing them would undermine `maskProp`, which is the
legitimate per-property override mechanism. The mask system *is* the
per-property "shadow" — installing a generic method shadow on top of
masks would just be a slower, less safe duplicate.

## Cross-References

- [persistence.md](./persistence.md) — how `savedProps` rides the
  hydrator + persistent field aggregation.
- [call-security.md](./call-security.md) — `ExecutionContextApi` is
  what `checkAccess` predicates and the auto-resolved mask owner read
  from. Also explains `@Unshadowable`.
- [antipatterns.md § Use props, not direct field
  assignment](../antipatterns.md#use-props-not-direct-field-assignment)
  — the headline rule.
